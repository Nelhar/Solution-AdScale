# Отказоустойчивость и Disaster Recovery

##  Стратегия High Availability

AdScale должна обеспечивать 99.5% SLA (MVP) и 99.9% SLA (Target). Это требует многоуровневой отказоустойчивости.

##  Определения

**RTO (Recovery Time Objective):** максимальное время восстановления после сбоя = 5 минут для MVP.

**RPO (Recovery Point Objective):** максимальный объем потерянных данных = 1 минута (приемлемо).

**SLA Бюджет:** время downtime в месяц
- 99.5% = 21 минута downtime/месяц (MVP)
- 99.9% = 4.3 минуты downtime/месяц (Target)

## � Pod-Level Failover

### Как это работает

K8s постоянно мониторит health каждого pod'а:

1. **Liveness Probe** проверяет, жив ли pod (каждые 10 секунд)
   - Проверяет /health/live endpoint
   - Если 3 ошибки подряд  pod считается dead
   - K8s автоматически убивает pod
   - Новый pod стартует (из deployment replicas)

2. **Readiness Probe** проверяет, готов ли pod принимать трафик (каждые 5 секунд)
   - Проверяет /health/ready endpoint
   - Если fail  pod удаляется из Load Balancer
   - Трафик перенаправляется на здоровые pods
   - Pod может восстановиться и вернуться в LB

3. **Zero-Downtime Recovery**
   - Healthy pods продолжают обрабатывать запросы
   - Трафик автоматически переходит на живые pods
   - Recovery time: < 30 секунд (от момента failure до ready)

### Пример сценария

```
14:00:00 - RTB Service pod-1 работает нормально
14:00:10 - Liveness probe успешен
14:00:15 - Readiness probe успешен

14:05:00 - Pod-1 crashed (OOM, segfault, etc.)
14:05:10 - Liveness probe fails (3 times)
14:05:30 - K8s удаляет pod-1
14:05:30 - Новый pod-2 стартует
14:05:35 - Pod-2 ready, добавлен в LB
14:05:40 - Pod-2 принимает трафик

Downtime: 0 (трафик обрабатывался pod-3, pod-4, pod-5)
```

## � Database Failover

### Текущая проблема (AS-IS)

Одна инстанция PostgreSQL = Single Point of Failure. Если она падает, система полностью вниз.

### Решение (MVP)

Master-Slave репликация:
- **Primary (Master):** все WRITES, OLTP операции
- **Replica 1:** read-only, Analytics queries
- **Replica 2:** hot standby, используется для failover

### Failover процедура

1. **Detection** (10 секунд)
   - Мониторинг (Prometheus) проверяет health Master каждые 5 секунд
   - После 2 failed checks (10 сек) = Master is down
   - Alert срабатывает

2. **Promotion** (20 секунд)
   - DBA или automation script: PROMOTE REPLICA 2 to PRIMARY
   - Replica 2 становится Master (перестает принимать binary log)
   - Replica 2 начинает принимать WRITES

3. **Update Connections** (rolling restart, ~30 сек)
   - Update connection string в environment variables
   - K8s rolling restart pods (max 1 pod at a time)
   - Pods подключаются к новому Master

**Всего downtime: 30-60 секунд** (RTO достигается)

### Мониторинг failover

Метрики для алертинга:
- Replication lag > 10 seconds (warning)
- Replication lag > 30 seconds (critical, failover)
- Master connection failed (immediate failover)
- Replica lagging > 1 hour (check WAL archive)

## � Kafka Broker Failover

### MVP: Single Broker
- Риск: если broker падает, события теряются
- Решение: только для MVP, приемлемо

### Target: 3-Broker HA Cluster

Kafka использует replication factor = 3 для высокой доступности:

1. **Producer** публикует impression в topic
   - Kafka распределяет replicas на 3 разные broker'ы
   - Min ISR = 2 (minimum in-sync replicas)
   - Producer ждет confirmation с 2 broker'ов (быстро)

2. **Broker Failure**
   - Если 1 broker падает: 2 replicas еще живы 
   - Kafka автоматически rebalances
   - Leader election на оставшихся broker'ах

3. **Consumer Rebalancing**
   - Consumer group автоматически rebalances
   - Reads переходят на доступные replicas
   - Zero data loss

**Пример:** 3-broker cluster, impression.recorded topic
- Partition 0: replicas [broker-1, broker-2, broker-3]
- Broker-2 падает  cluster продолжает работать на [broker-1, broker-3]
- Consumer продолжает читать без перерыва

## � Redis Failure Handling

### Graceful Degradation

Redis используется только для оптимизации (cache). Данные не критичны.

Если Redis падает:
1. RTB Service пытается читать из Redis  fail
2. Circuit Breaker открывается (после 5 ошибок)
3. Fallback: читаем campaigns напрямую из PostgreSQL (5-10ms вместо 1ms)
4. Latency деградирует на 20-30%, но SLA не нарушается
5. Redis восстанавливается  cache warming автоматически

**Downtime: 0 (система продолжает работать с деградацией)**

## � Application-Level Failover

### Circuit Breaker в действии

Если Campaign Service недоступна:

1. **Первые 5 запросов:** пытаемся обращаться к Campaign Service (gRPC с 5ms timeout)
2. **После 5 fail'ов:** Circuit Breaker открывается
3. **Следующие запросы (30 сек):** fast-fail (1ms, не пытаемся)
4. **Через 30 сек:** Half-Open (пробуем одно соединение)
5. **Если успешно:** Circuit Breaker закрывается, нормальная работа

**Fallback при fail'е:**
- Используем Redis cache (campaigns_cache)
- Если cache miss: используем last-known campaigns
- Если all fail: возвращаем default bid (быстро)

**Latency:** может вырасти до 50-60ms вместо 50ms, но система работает

##  Backup и PITR (Point-In-Time Recovery)

### Strategy

Данные в PostgreSQL критичны (финансовые, campaign info). Нужна надежная backup стратегия.

### Backup Schedule

1. **Daily Full Backup** (01:00 UTC)
   - pg_dump или pg_basebackup в S3
   - Размер: ~100GB (для большой БД)
   - Retention: 30 дней
   - Инструмент: pgbackrest или WAL-E

2. **WAL (Write-Ahead Log) Archive** (каждые 30 минут)
   - WAL архивируется в S3/GCS после каждого checkpoint
   - Размер WAL: ~16MB per 30 min
   - Retention: 7 дней
   - Позволяет восстановиться до конкретного timestamp

### Recovery Procedure

**Сценарий:** Database corrupted в 14:30, нужно восстановиться на 14:00

1. **Restore from latest daily backup** (01:00)
   - Restore full dump в новую инстанцию
   - Восстановлен state на 01:00

2. **Replay WAL archive** (01:00  14:00)
   - PostgreSQL читает WAL logs и replays transactions
   - Восстановлен state на 14:00

3. **Verify & Switchover**
   - Test queries на восстановленной БД
   - Update connection strings
   - Rolling restart pods

**Recovery time: 10-30 минут** (в зависимости от размера)

##  12-Factor Application Compliance

Для отказоустойчивости система должна быть stateless и portable:

1. **Stateless Processes:** RTB Service не хранит state (можем kill/restart любой pod)
2. **Environment Config:** все настройки из env vars (DB_HOST, CACHE_URL, etc.)
3. **Backing Services:** external (PostgreSQL, Redis, Kafka) не зависят от процесса
4. **Graceful Shutdown:** SIGTERM  wait 15s  close connections  exit
5. **Port Binding:** сервис self-contained, слушает на PORT
6. **Concurrency:** процессы независимы, К8s управляет масштабированием

Преимущество: если pod падает, новый pod может встать и работать идентично

##  Failover Checklist

Для MVP (99.5% SLA):

-  K8s health checks (liveness + readiness)
-  Pod replication (3-5 replicas)
-  PostgreSQL Master-Slave репликация
-  Failover script для promotion Slave
-  Kafka 1-3 brokers (зависит от бюджета)
-  Redis с Sentinel для failover (опционально)
-  Daily backup + WAL archive
-  Graceful shutdown в коде
-  Health checks endpoints
-  Мониторинг и alerting

Для Target (99.9% SLA):

-  Multi-region deployment
-  3-broker Kafka cluster (HA)
-  Redis cluster (6 nodes, Sentinel)
-  PostgreSQL cluster (Master + 2+ Replicas)
-  Geo-redundant backup (S3 + GCS)
-  Canary deployments
-  Chaos engineering тесты
-  Runbooks для каждого сценария failover

---

Опубликовано: 2026-06-23

