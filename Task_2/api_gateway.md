# API Gateway и BFF Слой

##  Архитектура (обновленная - Nginx вместо Kong)

Вместо fat API Gateway (Kong) используется разделение ответственности. Nginx обеспечивает маршрутизацию и TLS termination. BFF Layer (Backend For Frontend) выполняет валидацию OpenRTB, аутентификацию и бизнес-логику. Service Mesh (Istio + Envoy Proxy) обеспечивает resilience patterns как Circuit Breaker, retry logic и distributed tracing. Ядро системы (Bidding Service) содержит только бизнес-логику.

## � Nginx Слой

Nginx слушает на порту 443 (HTTPS) с TLS 1.3 шифрованием. Он перенаправляет все входящие запросы к /v2/bid endpoint на BFF Layer через upstream с least_conn стратегией load balancing. Nginx сохраняет keep-alive соединения для переиспользования. Timeout'ы установлены в 10ms на соединение, 90ms на отправку и получение.

Nginx логирует все запросы для мониторинга и отправляет метрики в Prometheus. P99 latency на Nginx уровне должна быть меньше 5 миллисекунд.

## � BFF Layer (Backend For Frontend)

BFF Слой валидирует входящий JSON на соответствие OpenRTB schema. Проверяет наличие обязательных полей, корректность типов данных, размеры payload'ов (максимум 1 МегаБайт). Отклоняет запросы, которые не проходят валидацию с ошибкой 400 Bad Request.

BFF Layer извлекает X-DSP-Key из заголовка и проверяет, что DSP партнер активен в системе. Если ключ невалиден, возвращает ошибку 401 Unauthorized.

BFF Layer проверяет rate limit для каждого DSP партнера. Лимит установлен в 100 тысяч запросов в минуту на партнера. Если лимит превышен, возвращает ошибку 429 Too Many Requests.

BFF Layer обогащает запрос дополнительной информацией: добавляет DSP ID, DSP name, timestamp, correlation ID для трейсинга через всю систему.

BFF Layer перенаправляет валидированный запрос к Bidding Service через Service Mesh. P99 latency на BFF уровне должна быть меньше 10 миллисекунд.

##  Service Mesh (Istio + Envoy)

Service Mesh работает как прозрачный proxy между BFF и Bidding Service. Он реализует Circuit Breaker паттерн: если Bidding Service возвращает ошибки, после 5 ошибок вход в Istio блокируется на 30 секунд, все запросы возвращают fast-fail.

Service Mesh реализует Retry паттерн: каждый запрос может быть переотправлен до 3 раз с exponential backoff (10ms, 20ms, 40ms).

Service Mesh реализует Rate Limiting: максимум 500 pending requests на сервис, максимум 2 requests per connection для HTTP/1.1.

Service Mesh собирает Distributed Tracing через Jaeger для отладки и мониторинга. Каждый request получает unique trace ID, который проходит через всю систему.

##  Мониторинг

Мониторятся метрики: P99 latency (alarm если > 100ms), P99.9 latency (alarm если > 120ms), error rate (alarm если > 1%), rate limit exceeds (alarm если > 10), throughput (RPS текущей нагрузки).

Все метрики отправляются в Prometheus и visualize'ятся в Grafana.

---

Опубликовано: 2026-06-22


