# Стратегии Масштабирования

##  Сценарии Масштабирования

AdScale должна расти от 15K RPS (AS-IS) через 18K RPS (MVP) к 50K RPS (Target) за 1 год.

##  Фазы Масштабирования

### Фаза 0: AS-IS (15K RPS)
- Монолит на одной машине
- Одна инстанция PostgreSQL
- P95 latency: 110-120ms (проблема!)
- Кэш: отсутствует
- Async: отсутствует

### Фаза 1: MVP (18K RPS, 3 месяца)
- Модульный монолит с выделенными компонентами
- Redis cache (95% hit ratio)
- PostgreSQL Master-Slave
- Kafka single-broker
- P95 latency: 50-60ms 
- Горизонтальное масштабирование возможно (добавлять pods)

### Фаза 2: Scaling Phase 1 (30K RPS, 6 месяцев)
- Bidding Service выделена как отдельный микросервис
- PostgreSQL: 2 shards (hash(campaign_id) % 2)
- Kafka: 2-3 brokers (для HA)
- Redis: Cluster (вместо single instance)
- Каждый микросервис масштабируется независимо

### Фаза 3: Scaling Phase 2 (50K RPS, 12 месяцев)
- 6 полноценных микросервисов
- PostgreSQL: 4 shards
- Service Mesh полностью включен (Istio)
- Multi-region deployment (для 99.9% SLA)
- Event Sourcing для критичных сервисов
- CQRS для read-heavy сервисов

## � Как Масштабируется Каждый Компонент

### PostgreSQL Масштабирование

#### Шаг 1: Добавить Read Replicas (Фаза 0  Фаза 1)
```
Было:  Master (OLTP + OLAP)
Стало: Master (OLTP) + Slave 1 (OLAP) + Slave 2 (Backup)
```
- Analytics queries идут в Slave, не блокируют Master
- OLTP latency улучшается

#### Шаг 2: Sharding (Фаза 1  Фаза 2)
```
Было:  One PostgreSQL, все данные
Стало: Shard 0 (advertiser_id 0-25K)
       Shard 1 (advertiser_id 25K-50K)
```
- Каждый shard имеет свою Master + Slave
- Write нагрузка распределяется
- Throughput растет линейно с числом шардов

#### Шаг 3: Масштабирование Shards (Фаза 2  Фаза 3)
```
Было:  2 shards
Стало: 4 shards (advertiser_id % 4)
```
- Миграция данных на лету (перехеширование)
- Gradual rebalancing (без downtime)
- Throughput растет с 30K RPS до 50K RPS

### Redis Масштабирование

#### Шаг 1: Single Instance (Фаза 0)
- Одна инстанция Redis
- Memory: 10-50MB
- Достаточно для 15-20K RPS

#### Шаг 2: Cluster (Фаза 1  Фаза 2)
- Redis Cluster (6 nodes)
- Каждый node: primary + replica
- Hash slots: 16384, распределены по nodes
- Automatic failover (Sentinel)
- Memory: до 2GB

#### Шаг 3: Multi-Region (Фаза 3)
- Redis в каждом регионе (для low latency)
- Cross-region replication (async, eventual consistency OK)
- Cache invalidation: через Kafka или direct replication

### Kafka Масштабирование

#### Шаг 1: Single Broker (Фаза 1)
- 1 broker достаточно для 18K RPS
- Topics: impressions.recorded (10 partitions), clicks.recorded (10), etc.
- Replication factor: 1 (no redundancy, но OK для MVP)

#### Шаг 2: 3-Broker Cluster (Фаза 2)
- HA cluster: if 1 broker down, система работает
- Replication factor: 3 (трех-кратная избыточность)
- Min ISR: 2 (consistency guarantee)
- Throughput: 30K msg/sec

#### Шаг 3: Multi-Region (Фаза 3)
- Kafka cluster в каждом регионе
- Cross-region replication (MirrorMaker или Kafka Connect)
- Each region independent, но синхронизированы
- Throughput: 50K+ msg/sec

### Bidding Service Масштабирование

#### Шаг 1: Single Deployment (Фаза 0)
- Все в монолите
- 3-5 pods (K8s replicas)
- Load balancer перед ними

#### Шаг 2: Separate Deployment (Фаза 1  Фаза 2)
- Bidding Service как отдельный deployment
- 5-10 pods (зависит от нагрузки)
- Auto-scaling: если P99 latency > 80ms, добавить pods
- Каждый pod stateless (можем добавлять/удалять)

#### Шаг 3: Geo-Distributed (Фаза 3)
- Bidding Service в нескольких регионах
- Глобальный load balancer (DNS)
- Region-local caching (Redis)
- Latency: <50ms для любого пользователя

##  Метрики для Масштабирования

Мониторим эти метрики, чтобы знать, когда масштабировать:

### Pod-Level Метрики
- **CPU utilization:** если > 70%  scale up
- **Memory usage:** если > 80%  scale up
- **P99 latency:** если > 80ms  scale up
- **Error rate:** если > 1%  check health

### Database Метрики
- **Connection pool utilization:** если > 80%  increase pool size или scale DB
- **Query latency:** если > 1000ms  add index или scale
- **Replication lag:** если > 100ms  network issue
- **Disk usage:** если > 80%  cleanup or expand

### Cache Метрики
- **Hit ratio:** если < 85% (was 95%)  memory pressure
- **Eviction rate:** если > 1/sec  memory insufficient
- **Latency:** если > 5ms (was 1ms)  network congestion

### Message Queue Метрики
- **Consumer lag:** если > 60s (was 10s)  consumers slow
- **Throughput:** если near max capacity  scale brokers
- **Broker CPU/Memory:** если > 70%  scale cluster



Опубликовано: 2026-06-23


