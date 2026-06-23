# TO-BE MVP: Модульный Монолит + Инфраструктура

##  Стратегия: Strangler Fig Pattern

Вместо полного переписывания системы используется постепенная миграция. Текущий монолит продолжает работать. Новые компоненты (Redis, Kafka, read-replicas) добавляются параллельно. Критичные части (Bidding Service) выделяются в отдельные модули внутри монолита. Обо это происходит без downtime.

##  MVP Архитектура

Nginx (маршрутизация + TLS) перенаправляет на BFF Layer. BFF валидирует OpenRTB и перенаправляет на Модульный Монолит (optimized). Модульный Монолит содержит Bidding Service (выделен как отдельный модуль), Campaign Service (модуль), Financial Service (модуль), Analytics (async через Kafka).

Bidding Service читает из Redis cache (95% hits) и PostgreSQL Master. Campaign Service CRUD через PostgreSQL Master. Analytics потребляет события из Kafka и пишет в PostgreSQL Slave (read-replica).

PostgreSQL Master используется для всех writes. PostgreSQL Slave используется для analytics reads и backup. Kafka содержит события для async обработки. Redis кэширует campaigns и бюджеты.

Service Mesh (Istio) обеспечивает Circuit Breaker, retry, rate limiting, distributed tracing между модулями.

##  Как это решает Bottleneck'и

Auction Engine bottleneck: кэшируем campaigns в Redis, читаем за 1ms вместо 5ms DB запроса. Результат: P95 latency 110ms  50ms (снижение в 2 раза).

PostgreSQL SPOF: добавляем Master-Slave репликацию. Если Master падает, Slave можно продвинуть на Primary за 30-60 секунд.

Statistics write bottleneck: вместо синхронной записи, events отправляются в Kafka. Async consumer пишет в фоне. Result: no lock contention.

Analytics blocking production: Analytics читает из read-replica БД, не конкурирует с production write'ами.

##  План Внедрения (3 месяца)

Месяц 1: Инфраструктура. Развернуть Redis кластер, настроить PostgreSQL Master-Slave репликацию, развернуть Kafka broker, поднять Service Mesh (Istio).

Месяц 2: Bidding Service. Выделить код Bidding Service как отдельный модуль, интегрировать Redis cache, интегрировать Kafka events. Развернуть в production (canary deployment: 5%  25%  100% трафика).

Месяц 3: Оптимизация. Tune кэширование, подключить новую DSP, performance testing (18K RPS), мониторинг и alerting, документация и runbooks.

##  Результаты

P95 latency: 110-120ms  50-60ms (достижение SLA требования).

RPS: 15K  18K+ (увеличение на 20%, headroom для роста).

PostgreSQL load: 100%  40-50% (разделение нагрузки благодаря read-replicas).

Cache hit ratio: 0%  95%+ (10x уменьшение DB нагрузки).

Downtime risk: High  Low (благодаря Master-Slave и async processing).

Readiness для DSP: Система готова интегрировать новую DSP без degradation SLA.

##  Риски и Смягчение

Сложность репликации: Протестировать failover сценарии заранее.

Консистентность кэша: Использовать TTL + event-based invalidation.

Миграция live трафика: Gradual rollout (5%25%100%).

Kafka reliability: Replication factor = 3 даже в MVP.

##  Success Criteria

P95 latency ≤ 100ms. RPS ≥ 18,000. Cache hit ratio ≥ 95%. SLA ≥ 99.5%. Zero downtime при внедрении. Fallback механизмы работают.




