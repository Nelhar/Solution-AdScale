# TO-BE Target: Микросервисная Архитектура (12 месяцев)

##  Стратегия: Полные Микросервисы

Целевая архитектура полностью избавляется от монолита и представляет 6 независимых микросервисов, развернутых в нескольких регионах. Система достигает 50 тысяч RPS, P99 latency менее 50 миллисекунд и SLA 99.9%. Эта архитектура является результатом эволюции через Strangler Fig Pattern за 12 месяцев.

##  Target Архитектура

6 микросервисов развернуты независимо в 2+ регионах. Каждый сервис имеет свою базу данных (Database per Service паттерн). Сервисы общаются синхронно через gRPC (internal) и REST (external), асинхронно через Kafka. Service Mesh (Istio + Envoy) управляет трафиком, circuit breaking, retry логикой и distributed tracing.

**Микросервисы:**
- Bidding Service: Real-Time Bidding, аукцион (10-20 replicas)
- Campaign Service: управление кампаниями (3-5 replicas)
- Financial Service: финансовые операции, SAGA (3-5 replicas)
- Analytics Service: агрегация метрик (2-4 replicas)
- Admin Service: операции, мониторинг (2 replicas)
- Notification Service: алерты, коммуникация (2-3 replicas)

**Инфраструктура:**
- PostgreSQL: 4 шарда, каждый с Master + 2 Read Replicas
- Redis: 6-node кластер  с Sentinel (автоматический failover)
- Kafka: 3-5 брокеров, 28 партиций, мульти-региональная репликация
- Service Mesh: Istio + Envoy для всех сервисов

##  Как это решает проблемы

Bidding Service независимо масштабируется до 20 replicas при пиковой нагрузке. Каждый сервис имеет собственную БД (нет конкуренции за ресурсы). Analytics работает на отдельных replicas (не блокирует production). PostgreSQL sharding распределяет нагрузку (в 4 раза меньше данных на шард). Service Mesh обеспечивает circuit breaking, retry и timeout enforcement автоматически.

Результат: P99 < 50ms, 50K RPS, 99.9% SLA достигаются стабильно.

##  Путь Эволюции (12 месяцев)

Фаза 1 (месяцы 1-3): MVP с Strangler Fig (модульный монолит, Redis, Kafka, Master-Slave). Результат: P95 110ms  50ms, 18K RPS, SLA 99.5%.

Фаза 2 (месяцы 4-9): Масштабирование (Bidding Service отделяется, PostgreSQL 2 шарда, Kafka 3-брокер кластер, начало развёртывания в мульти-региональном кластере). Результат: 30K RPS, SLA стабилизируется.

Фаза 3 (месяцы 10-12): Полные микросервисы (6 сервисов независимо, PostgreSQL 4 шарда, Service Mesh включена полностью, создание мульти-региональной инфраструктуры). Результат: 50K RPS, P99 <50ms, SLA 99.9%.

##  Target Метрики

P50 latency: 20ms. P95: 50ms. P99: 50ms. P99.9: 100ms.

RPS: 50,000 peak. Kafka throughput: 100,000 msg/sec. Финансовые транзакции: 10,000/sec.

SLA: 99.9% (допустимо 4.3 минуты downtime в месяц). Auto-recovery: < 30 секунд. Failover: < 5 минут.

##  Multi-Region Deployment

Регион 1 (Primary): все сервисы Master, databases Master, Kafka leaders. Регион 2 (Secondary): сервисы реплик, databases Read Replicas (асинхронные), Kafka replicas только.

Sync: database binary log + Kafka MirrorMaker. Health checks каждые 10 секунд. GeoDNS routing: автоматический failover если Регион 1 down.

##  Database per Service Strategy

Bidding DB: 4 шарда (hash(advertiser_id)), каждый Master + 2 Replicas, ~100GB per shard.

Campaign DB: Master + 2 Replicas, ~50GB, backup S3 + GCS.

Financial DB: Master + 2 Replicas, Event Sourcing, ~200GB audit trail, replication factor 3.

Analytics DB: TimeSeries (InfluxDB), ~500GB, 1-year retention, 3 replicas.

Admin DB: Master + Slave, ~10GB. Notification DB: write-optimized, ~5GB, partitioned by date.

##  Взаимодействие

Синхронные (Service Mesh управляет): DSP  Nginx REST  BFF Layer  Bidding Service gRPC (P99 < 50ms, timeout 100ms). Bidding  Campaign Service gRPC (5ms timeout, fallback to cache). Bidding  Financial Service gRPC (budget queries, readonly).

Асинхронные (Kafka): Bidding  impressions.recorded, clicks.recorded. Financial  transactions.completed. Campaign  campaigns.updated (cache invalidation). Analytics + Financial + Notification consume события.

Kafka topics: impressions (10 partitions, 1-day), clicks (10 partitions, 1-day), transactions (5 partitions, 7-day), campaigns (3 partitions, 1-hour).

##  Service Mesh (Istio + Envoy)

Traffic Management: VirtualServices (routing), DestinationRules (load balancing ROUND_ROBIN/LEAST_REQUEST).

Resilience: Circuit Breaker (5 ошибок  open, 30s half-open), Retry (3x exponential backoff 10/20/40ms), Rate Limiting (per service/consumer), Timeout Enforcement (100ms boundary).

Observability: Jaeger distributed tracing (correlation IDs), Prometheus metrics (latency/error/throughput), Fluentd + ELK centralized logging.

##  MVP vs Target Сравнение

MVP: модульный монолит, 18K RPS, P95 50-60ms, SLA 99.5%, 1 регион, Master + 2 Slave, Redis single, Kafka 1 broker, Service Mesh базовая.

Target: 6 микросервисов, 50K RPS, P99 <50ms, SLA 99.9%, 2+ регионов, 4 шарда × Master + 2 Replica, Redis 6-node cluster, Kafka 3-5 brokers, Service Mesh полная, 4-6 команд (team autonomy).

##  Success Criteria

50,000 RPS peak load. P99 < 50ms globally. SLA 99.9% достигается. Zero data loss (финансовые). Multi-region failover < 5 min. Independent service scaling. Team autonomy (каждая команда деплоит свой сервис).


