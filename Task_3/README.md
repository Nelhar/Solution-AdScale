# Task_3: Масштабирование, Кэширование и Отказоустойчивость



##  Документация

### Основные Документы

#### 1. **database_strategy_CLEAN_RU.md**
Стратегия базы данных для масштабирования:
- **Master-Slave Репликация**:
  - Master: OLTP writes (campaigns, budgets, bids, ledger)
  - Slave 1: Analytics queries (read-only, OLAP)
  - Slave 2: Backup, hot standby for failover
  - Async binary log репликация (1-2 min lag)
  - Failover за 30-60 секунд (promote replica)
- **Sharding**:
  - Ключ sharding: advertiser_id (hash % num_shards)
  - MVP: 1 shard, затем 2, 4, 16
  - Региональное sharding: by geography (GDPR compliance)
- **Connection Pooling**:
  - PgBouncer: 1000 client connections, 100 backend connections
  - Transaction-level pooling для переиспользования
- **Query Optimization**:
  - Индексы: (advertiser_id, is_active), (created_at)
  - Slow query log: alarm если query > 1000ms
  - Partitioning для больших таблиц (impressions, clicks)
- **Backup & PITR**:
  - Дневной полный backup в S3/GCS
  - WAL архив каждые 30 минут
  - Point-in-time recovery до конкретного timestamp (7 дней retention)

#### 2. **caching_CLEAN_RU.md**
Стратегия кэширования в Redis:
- **Что кэшировать**:
  - Campaigns: 5min TTL, 10MB (100K campaigns × 1KB)
  - Advertisers: 10min TTL, 200 bytes each
  - Bid floors: 30min TTL, по geo и category
  - Impression counters: 24h TTL, auto-expire UTC
- **Cache Invalidation**:
  - TTL-based: автоматический cleanup через время
  - Event-based: Kafka campaign.updated событие  DEL в Redis
  - Hybrid: оба подхода для reliability
- **Cache Warming**:
  - При старте RTB Service: загрузить все active campaigns в Redis (2-3 сек)
  - Периодический refresh каждые 5 минут
  - Нет cold start проблем
- **Fallback**:
  - Если Redis down: читаем из PostgreSQL (медленнее 5-10ms, но работает)
  - Cache miss: 5% случаев, читаем из БД
- **Hit Ratio**: 95%+ (снижает DB нагрузку в 20 раз)
- **Мониторинг**:
  - Cache hit ratio (alert если < 80%)
  - Evictions per second (alert если > 1)
  - Memory used (alert если > 1GB)
  - Latency histogram (target < 2ms)

#### 3. **event_streaming_CLEAN_RU.md**
Event-driven архитектура с Kafka:
- **Kafka Topics**:
  - impressions.recorded: 10 partitions, 1-day retention, replication=3
  - clicks.recorded: 10 partitions, 1-day retention
  - transactions.completed: 5 partitions, 7-day retention (financial)
  - campaigns.updated: 3 partitions, 1-hour retention (cache invalidation)
- **Event Schema**:
  - event_id (UUID для deduplication)
  - timestamp (ISO 8601)
  - business data (impression details, click info, etc.)
  - Partitioning by campaign_id: гарантирует порядок в одной кампании
- **Consumers**:
  - Analytics group: batch aggregation to InfluxDB (100-500ms OK)
  - Financial group: exactly-once processing with deduplication
  - Archival group: долгосрочное хранение в S3
- **Producer** (RTB Service):
  - Non-blocking publish (1ms)
  - Local queue fallback если Kafka down
  - Error handling & retry
- **Exactly-Once Semantics**:
  - Consumer проверяет deduplication по event_id перед processing
  - Transactional writes: begin  check  update  commit
  - Recovery: Kafka rebalancing автоматически
- **Масштабирование**:
  - MVP: 1 broker (15K RPS = 100K msg/sec, достаточно)
  - Target: 3-broker cluster (HA, replication factor=3)
  - Partitions: 10 достаточно для 50K RPS (5K RPS per partition)

#### 4. **failover_CLEAN_RU.md**
Стратегия отказоустойчивости и восстановления:
- **Pod-Level Failover**:
  - K8s liveness probe fails  pod killed  new pod spun up (0 downtime)
  - Readiness probe: pod removed from LB при degradation
  - Recovery time: < 30 секунд
- **Database Failover**:
  - Primary PostgreSQL crash  promote read replica  20-60 сек downtime
  - Update connection strings  rolling restart pods
  - Backup: daily full backup + WAL archive each 30 min
  - PITR: restore to specific timestamp using WAL
- **Kafka Broker Failover**:
  - Multi-broker cluster with replication factor 3
  - If 1 broker down: 2 replicas still running 
  - Consumer rebalancing automatic
- **Redis Failure**:
  - Graceful degradation: cache miss  query DB
  - Latency increases (5-10ms instead of 1ms) but doesn't break SLA
  - Can afford to lose cache data (non-critical)
- **Service Failover**:
  - Circuit breaker opens after 5 errors
  - Fallback to cached data or default response
  - Manual promote replica if primary doesn't recover
- **12-Factor Compliance**:
  - Stateless processes (can kill/restart any pod)
  - Environment variable config (no hardcoded values)
  - External backing services (DB, cache, Kafka separate)
- **SLA Budget**:
  - MVP: 99.5% = 21 min downtime/month (acceptable)
  - Target: 99.9% = 4.3 min downtime/month (multi-region)

##  Диаграммы (C4 PlantUML)

### 1. **scaled_architecture.puml**  Полная Масштабированная Архитектура
Показывает:
- Frontend layer: 2 Nginx load balancers
- API layer: 3 BFF pods
- Service Mesh: Envoy proxy network
- Microservices: 5 replicas каждого сервиса (RTB, Campaign, Analytics, Financial, Admin)
- Data layer:
  - Kafka cluster (3 brokers, HA)
  - Redis cluster (6 nodes, Sentinel)
  - PostgreSQL cluster (Master + 2 read replicas, sharded)
- Observability: Prometheus + Jaeger + ELK
- Auto-scaling enabled
- Multi-region capable

**Легенда:** 18K50K RPS, P99 <50ms globally, 99.9% SLA, fully distributed, auto-scaling

### 2. **data_diagram.puml**  Модель Данных (НОВАЯ!)
Показывает:
- **Bidding Database (4 shards)**:
  - impressions table (10B+ rows, partitioned by date, sharded by campaign_id)
  - bids table (1B+ rows, append-only)
- **Campaign Database (Master-Slave)**:
  - campaigns (100K rows, indexed by advertiser_id + is_active)
  - budgets (100K rows, real-time updates)
  - targeting_rules (500K rows, cached in Redis)
- **Financial Database (Event Sourcing)**:
  - ledger (100M+ rows, append-only, event-sourced)
  - invoices (1M rows, immutable)
  - balances (1K rows, updated via SAGA)
- **Analytics Database (TimeSeries - InfluxDB)**:
  - impressions_metrics (aggregated hourly, 1-year retention)
  - clicks_metrics (aggregated hourly)
  - performance metrics (for dashboards)
- **Redis Cache (Distributed Cluster - 6 nodes)**:
  - campaigns_cache (5min TTL, 10MB)
  - budgets_cache (1min TTL, 1MB)
  - bid_floors_cache (30min TTL)
  - impression_counters (24h TTL)
- **Kafka Topics (HA - 3 brokers)**:
  - impressions.recorded (10 partitions, by campaign_id)
  - clicks.recorded (10 partitions)
  - transactions.completed (5 partitions, 7-day retention)
  - campaigns.updated (3 partitions, 1-hour retention)

**Аннотации:** размеры, TTL, retention, replication factor, sharding strategy

## � Ключевые Концепции

### Master-Slave Репликация
```
Write Request  Master  log to WAL  Binary Log
                          
                       Slave 1 (read-only, Analytics)
                          
                       Slave 2 (hot standby, failover)
```
Failover: If Master dies  promote Slave 2  update connections  rolling restart

### Database per Service
```
Bidding Service  Bidding Database (sharded)
Campaign Service  Campaign Database (Master-Slave)
Analytics Service  Analytics Database (TimeSeries)
Financial Service  Financial Database (Event Sourcing)
Admin Service  Admin Database
```
Each service owns its data, can scale independently

### Cache Invalidation
```
Campaign Update Event
     (Kafka: campaigns.updated)
Analytics Consumer (aggregate stats)
Financial Consumer (update ledger)
Bidding Service Consumer (DEL campaigns:123 from Redis)
```

### Event Sourcing Pattern
```
Transaction Request
    
1. Begin transaction
2. Check if event_id already processed (deduplication)
3. Update account balance (SAGA compensation)
4. Check if budget exceeded
5. Commit or Rollback
    
Ledger (append-only): preserved as immutable log
```

##  Метрики Задачи

### Database Performance
- Master write latency: <10ms
- Slave read latency: <5ms
- Replication lag: <100ms
- Query performance: <1000ms (alert if > 1s)
- Connection pool utilization: <80%

### Caching Performance
- Cache hit ratio: 95%+
- Cache write latency: <1ms
- Cache eviction rate: <1/sec
- Memory utilization: <512MB (MVP), <2GB (Target)

### Event Streaming
- Producer latency: <1ms (non-blocking)
- Consumer lag: <10s (alert if > 1m)
- Throughput: 18K msg/sec (MVP)  50K msg/sec (Target)
- Exactly-once success rate: 100%

### Failover & Recovery
- Pod recovery time: <30s
- Database failover time: <60s
- Data loss: 0 (durable Kafka)
- RTO (Recovery Time Objective): <5 min
- RPO (Recovery Point Objective): <1 min



