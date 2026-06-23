# Task_2: Проектирование Микросервисов и Паттернов Надежности

##  Документация

### Основные Документы

#### 1. **bidding_service.md**
Проектирование Bidding Service (критичный сервис):
- Граница и ответственность микросервиса
- API контракты: REST (OpenRTB от DSP), gRPC (внутри), Kafka (async)
- Архитектура обработки bid request'а:
  - API Handler (1ms)
  - Campaign Matcher с Redis cache (5ms, 95% hit)
  - Auction Engine (30ms)
  - Ad Renderer (10ms)
  - Response Builder (3ms)
  - **Всего: P99 < 50ms** 
- Модель данных: impressions_log, bids_cache в PostgreSQL
- Паттерны надежности: Circuit Breaker, Timeout, Fallback
- Мониторинг: P99/P99.9 latency, error rates, cache hit ratio
- K8s развертывание: 3-5 replicas, zero-downtime rolling updates

#### 2. **api_gateway.md**
Проектирование API Gateway (Nginx + BFF + Service Mesh):
- Nginx слой: маршрутизация, TLS termination, load balancing (P99 <5ms)
- BFF Layer: валидация OpenRTB, аутентификация (X-DSP-Key), rate limiting (100K RPM per DSP), request enrichment (P99 <10ms)
- Service Mesh (Istio): Circuit Breaker, Retry, Rate Limiting, Distributed Tracing
- Мониторинг: P99 latency, error rate, rate limit exceeds, throughput
- **Архитектурное улучшение:** Kong  Nginx + BFF + Istio (разделение ответственности)

#### 3. **reliability.md**
Семь паттернов надежности системы:
1. **Circuit Breaker**: 5 ошибок  open  30s recovery  half-open
2. **Retry Pattern**: 3 попытки с exponential backoff (10ms, 20ms, 40ms)
3. **Timeout Иерархия**: 100ms (DSP)  99ms  95ms (RTB logic)  per-call timeouts
4. **Fallback Стратегии**: Redis cache  last-known campaigns  default bid (1ms)
5. **Булкхед Паттерн**: per-DSP thread pools (Google 20, Programmatic 15, Test 5)
6. **Health Checks**: liveness (/health/live) + readiness (/health/ready)
7. **Graceful Shutdown**: SIGTERM  stop accepting  wait 15s  close connections

#### 4. **interaction.md**
Взаимодействие компонентов:
- Синхронные вызовы (критический путь):
  - DSP  Nginx  BFF  Service Mesh  Bidding Service  Campaign Service
  - Protocol selection: REST (external, standard), gRPC (internal, fast), Kafka (async, decoupled)
- Асинхронные события (non-blocking):
  - Bidding Service публикует impression.recorded в Kafka (1ms, не блокирует)
  - Analytics consumer: batch aggregation (100-500ms OK)
  - Financial consumer: exactly-once с deduplication по event_id
- Error handling: cascade fallbacks при fail'е зависимостей
- Service Discovery: Kubernetes DNS (rtb-service:5000, campaign-service:5000, postgres:5432)
- Мониторинг: distributed tracing (Jaeger), metrics per component, latency per component

##  Диаграммы (C4 PlantUML)

### 1. **services_component.puml**  Bidding Service Компоненты
Показывает:
- Внутренние компоненты Bidding Service:
  - API Handler (парсинг OpenRTB)
  - Campaign Matcher (матчинг по targeting rules)
  - Auction Engine (выбор победителя)
  - Ad Renderer (получение creative)
  - Response Builder (форматирование ответа)
  - Event Emitter (публикация в Kafka)
  - Circuit Breaker (обработка fail'ов Campaign Service)
  - Cache Client (Redis lookup с fallback)
- Критичный путь с timings (P99 < 50ms)
- Fallback strategies визуально
- Связи с Campaign Service, PostgreSQL, Redis, Kafka

**Легенда:** Critical path 50ms, cache hit 95%, fallback strategies active

## � Ключевые Концепции

### Bidding Service Architecture
- Stateless: можно добавлять/удалять инстансы
- Idempotent: каждый impression имеет event_id для deduplication
- Fast path: кэш + fallback + timeout обеспечивают SLA
- Observable: correlation IDs, distributed tracing, metrics

### API Gateway Layers
1. **Nginx** (transport): маршрутизация, TLS, load balancing (5ms)
2. **BFF** (business logic): валидация, rate limit, auth (10ms)
3. **Service Mesh** (resilience): CB, retry, tracing (0ms overhead)
4. **Bidding Service** (core): auction logic (50ms)
**Total: P99 < 65ms** 

### Reliability Patterns в Action
```
Request  API Handler
          Campaign Matcher
           - Try Redis cache (95% hit, 1ms)
           - Fallback to Campaign Service (5ms timeout)
           - If fail: use last-known campaigns (Circuit Breaker)
          Auction Engine (30ms)
          Ad Renderer (10ms)
          Response Builder (3ms)
Response  Event Emitter (async to Kafka)
```

##  Метрики Задачи

### Bidding Service
- Throughput: 18,000 RPS (MVP)
- P99 Latency: <50ms
- Cache hit ratio: 95%+
- Fallback activation: <1% (healthy system)
- Error rate: <1%

### API Gateway
- Nginx: P99 <5ms
- BFF: P99 <10ms
- Rate limit: 100K RPM per DSP
- Circuit Breaker: 5 fails  open  30s recovery

### Reliability
- Retry success rate: 95%+ (after first fail)
- Bulkhead isolation: one DSP cannot exhaust all resources
- Graceful shutdown: zero in-flight request loss
- Health checks: pod recovery < 30s

##  Результаты Task_2

После завершения Task_2 у вас будет:

 Полный дизайн Bidding Service  
 Архитектура API Gateway с Nginx + BFF + Istio  
 Семь паттернов надежности с обоснованием  
 Дизайн взаимодействия компонентов (sync/async)  
 C4 Component диаграмма Bidding Service  
 Код примеры критичных компонентов (Go)  
 Мониторинг и alerting стратегия  

## � Связь с другими Task'ами

- **Task_1**: архитектурные драйверы и общий дизайн (основа для Task_2)
- **Task_3**: детальные стратегии кэширования, БД, масштабирования, failover




