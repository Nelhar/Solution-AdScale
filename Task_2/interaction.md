# Взаимодействие Компонентов

## Синхронные Вызовы (Критический Путь)

DSP отправляет bid request на REST endpoint /v2/bid. Nginx маршрутизирует на BFF. BFF валидирует и перенаправляет на Bidding Service через Service Mesh (gRPC). Bidding Service вызывает Campaign Service через gRPC (internal, очень быстро). Campaign Service читает из PostgreSQL или Redis. RTB возвращает ответ через Service Mesh обратно на BFF. BFF возвращает OpenRTB response на DSP.

Всего P99 latency: ~65ms (Nginx 5ms + BFF 10ms + RTB 50ms).

## Асинхронные События (Non-blocking)

После отправки ответа DSP, Bidding Service асинхронно публикует impression.recorded событие в Kafka. Это операция non-blocking (1ms) и не влияет на P99 latency.

Analytics consumer получает событие за 100-500ms и агрегирует статистику.

Financial consumer получает событие за 100ms и обновляет budget ledger с exactly-once гарантией.

## Выбор Протоколов

REST используется для external API (DSP  Nginx  BFF) потому что это стандарт, простой для отладки и совместим со всеми DSP платформами.

gRPC используется для internal calls (BFF  Bidding Service  Campaign Service) потому что это быстро (10x быстрее REST), имеет типизированные контракты и встроенную поддержку streaming.

Kafka используется для async events потому что это decoupled, scalable, replay-able, и гарантирует at-least-once delivery.

## Error Handling

Если Campaign Service down, Bidding Service использует Redis cache. Если cache miss, используется in-memory last-known campaigns. Если все fail, возвращает default response за 1ms.

Если Kafka down, Bidding Service queues события в локальную память и retry'ит, когда Kafka восстановится. Это не блокирует response пользователю.

Если БД down, Bidding Service использует cache до тех пор, пока БД не восстановится.

## Service Discovery

Kubernetes DNS используется для discovery: bidding-service:5000, campaign-service:5000, postgres:5432. Обновления отправляются автоматически при добавлении/удалении pod'ов.

## Мониторинг Interaction

Tracing: каждый request имеет correlation ID. Jaeger собирает traces через всю систему для отладки latency issues.

Metrics: latency per component, error rate per component, throughput per component, retry rate, fallback activation rate.

---

Опубликовано: 2026-06-22


