# Task_1: Анализ Архитектуры и Планирование


##  Документация

### Основные Документы

#### 1. **drivers.md**
Архитектурные драйверы системы:
- Функциональные требования (RTB, кампании, DSP интеграция)
- Нефункциональные требования (P95 ≤100ms, 18K50K RPS, 99.5%99.9% SLA)
- Бизнес-ограничения (3 месяца MVP, 1 год целевой вариант, 35-человечная команда)
- 7 критичных проблем для решения
- Критерии успеха для каждого варианта

#### 2. **AS-IS.md**
Анализ текущей архитектуры:
- Описание компонентов монолита (Ad Server, Auction Engine, Delivery Service, Statistics Service, Financial Service, Analytics Service, Advertiser Dashboard)
- 4 критичных bottleneck'а:
  - Auction Engine: синхронные вызовы БД в hot path
  - PostgreSQL SPOF: одна инстанция без HA
  - Statistics Service: write contention при 15K RPS
  - Analytics Service: OLAP блокирует OLTP
- Текущие метрики: 15K RPS, P95 110-120ms (требуется ≤100ms)
- Архитектурные ограничения (нет масштабирования, кэша, репликации, async обработки)

#### 3. **to_be_mvp.md**
MVP архитектура (модульный монолит):
- Strangler Fig Pattern для постепенной миграции
- Добавленная инфраструктура:
  - Redis для кэширования (95% hit ratio)
  - PostgreSQL Master-Slave репликация
  - Kafka для async events
  - Nginx + BFF + Service Mesh (Istio)
- Как решаются bottleneck'и:
  - Кэш снижает P95 с 110ms до 50-60ms
  - Репликация обеспечивает HA
  - Kafka убирает write contention
  - Analytics на read-replica не конкурирует
- План внедрения: 3 месяца (Месяц 1: инфра, Месяц 2: RTB Service, Месяц 3: оптимизация)
- Целевые результаты: 18K RPS, P95 ≤100ms, 99.5% SLA

##  Диаграммы (C4 PlantUML)

### 1. **as_is.puml**  Текущая Архитектура
Показывает:
- Монолитную архитектуру системы
- Все компоненты: Ad Server, Auction Engine, Delivery Service, Statistics Service, Financial Service, Analytics Service, Advertiser Dashboard
- Одну инстанцию PostgreSQL (SPOF)
- Критичные места отмечены (Bottleneck, Critical Path, Single Point of Failure)
- Связи между компонентами и БД (все синхронные вызовы)

**Легенда:** 15K RPS, P95 110-120ms, нет кэша, нет репликации, нет async

### 2. **to_be_mvp.puml**  MVP Архитектура
Показывает:
- Nginx (маршрутизация, TLS)  P99 <5ms
- BFF Layer (валидация, rate limiting)  P99 <10ms
- Service Mesh (Istio + Envoy)  Circuit Breaker, retry, tracing
- Модульный монолит с выделенными сервисами
- Redis cache для campaigns и budgets
- PostgreSQL Master-Slave репликация
- Kafka для impressions, clicks, transactions
- Analytics consumer за Kafka с read-replica БД
- Financial consumer с exactly-once гарантией

**Легенда:** 18K RPS, P95 50-60ms, SLA 99.5%, 3-месячная реализация

### 3. **to_be_target.puml**  Целевая Архитектура (Microservices)
Показывает:
- 6 микросервисов: RTB Service, Campaign Service, Analytics Service, Financial Service, Admin Service, Notification Service
- Каждый микросервис имеет свою БД (Database per Service)
- Kafka для event-driven коммуникации
- Redis distributed cache
- PostgreSQL sharded (по campaign_id и по географии)
- Service Mesh для управления микросервисами
- Multi-region готовность

**Легенда:** 50K RPS, P99 <50ms, 99.9% SLA, микросервисная архитектура, Event Sourcing + CQRS

## � Ключевые Концепции

### Strangler Fig Pattern
Постепенная миграция от монолита к микросервисам:
1. Оставить монолит работающим (zero-downtime)
2. Добавить инфраструктуру (Redis, Kafka, read-replicas)
3. Выделить критичный сервис (RTB Service)
4. Постепенно мигрировать трафик
5. Полное разделение на микросервисы

### Master-Slave БД Репликация
- Master: все writes (campaigns, budgets, bids, financial)
- Slave: read-only (Analytics, reporting, backup)
- Async binary log репликация (низкая latency)
- Failover за 30-60 секунд если Master падает

### Кэширование Redis
- Campaigns: 5 минут TTL, 10MB на MVP
- Budgets: 1 минута TTL
- Bid floors: 30 минут TTL
- Impression counters: 24 часа TTL (auto-expire UTC)
- Hit ratio целевой: 95%+ (уменьшает DB нагрузку 20x)

### Kafka Event Streaming
- Async обработка: RTB Service публикует events без блокировки
- Topics: impressions.recorded, clicks.recorded, transactions.completed, campaigns.updated
- Partitions по campaign_id: гарантирует порядок в одной кампании
- Consumers: Analytics (batch), Financial (exactly-once)

### Service Mesh (Istio + Envoy)
- Circuit Breaker: 5 ошибок  open, 30s recovery
- Retry: 3 попытки с exponential backoff (10ms, 20ms, 40ms)
- Rate limiting: per service, per DSP partner
- Distributed tracing: correlation IDs через всю систему

##  Метрики и Целевые Показатели

### AS-IS (Текущее состояние)
- RPS: 15,000
- P95 Latency: 110-120ms  (требуется ≤100ms)
- SLA: ~99%
- DB: 1 инстанция (SPOF)
- Кэш: отсутствует
- Async: отсутствует
- Масштабирование: невозможно

### MVP (Target для 3 месяцев)
- RPS: 18,000+ 
- P95 Latency: 50-60ms  (достижение SLA)
- SLA: 99.5% 
- DB: Master-Slave репликация
- Кэш: Redis (95% hit ratio)
- Async: Kafka events
- Масштабирование: горизонтальное возможно

### Target (Целевой вариант, 1 год+)
- RPS: 50,000 
- P99 Latency: <50ms 
- SLA: 99.9% 
- DB: 6 независимых БД (Database per Service) + sharding
- Кэш: Redis distributed cluster
- Async: Event-driven архитектура
- Масштабирование: полностью независимое per сервис
- Multi-region: готовность




