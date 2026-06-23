# Проектирование RTB Service

##  Граница и Ответственность Микросервиса

RTB Service отвечает за получение bid request от DSP партнера, выполнение матчинга активных кампаний на основе критериев таргетирования, запуск рекламного аукциона для выбора победителя и возврат winning ad в формате banner.

RTB Service НЕ отвечает за управление кампаниями (это делает Campaign Service), финансовые расчеты (это делает Financial Service) и долгосрочное хранение аналитики (это делает Analytics Service).

##  API Контракты

Входящее API от DSP использует REST протокол с JSON payload в формате OpenRTB 2.5. Endpoint расположен на POST /v2/bid. Требуемое время отклика 100 миллисекунд (это дедлайн от DSP партнера). P99 latency должна быть меньше 50 миллисекунд на уровне RTB Service, что обеспечивает buffer для остальных слоев (API Gateway, Network).

Внутреннее API к Campaign Service использует gRPC протокол для максимальной скорости. Timeout для этого вызова устанавливается в 5 миллисекунд, так как это операция должна быть очень быстрой. Если Campaign Service не отвечает за 5 миллисекунд, используется fallback.

Асинхронные события отправляются в Kafka. Topic impressions.recorded содержит информацию о каждом показанном баннере. Ключ сообщения используется campaign_id для обеспечения порядка обработки событий внутри одной кампании. Replication factor устанавливается в 3 для MVP версии для защиты от потери данных.

##  Внутренняя Архитектура Обработки

RTB Service обрабатывает bid request в следующем порядке. Сначала идет API Handler, который парсит JSON и валидирует payload за 1 миллисекунду. Затем Campaign Matcher выполняет матчинг активных кампаний, используя кэш Redis для получения списка кампаний за 5 миллисекунд (в 95 процентов случаев этот кэш попадает). Если кэш miss, то делается синхронный вызов к Campaign Service за 5 миллисекунд.

Далее запускается Auction Engine, который для каждой matching campaign оценивает targeting rules, вычисляет финальную ставку и выбирает победителя. Это занимает 30 миллисекунд. Затем Ad Renderer получает HTML и JavaScript для winning campaign, что занимает 10 миллисекунд. Наконец, Response Builder форматирует ответ в JSON формат OpenRTB за 3 миллисекунды.

После того как ответ отправлен пользователю, RTB Service асинхронно публикует событие impression.recorded в Kafka. Эта операция не блокирует ответ пользователю и не влияет на P99 latency.

Общее время на критичном пути: 1 + 5 + 30 + 10 + 3 = 49 миллисекунд, что находится в буджете 50 миллисекунд для P99.

##  Модель Данных

RTB Service использует PostgreSQL для хранения логов impressions. Таблица impressions_log содержит append-only логи всех показанных баннеров с информацией о bid, campaign, price и timestamp. Дополнительно ведется таблица bids_cache для отслеживания статистики ставок в реальном времени (бизнес-метрики).

Redis cache хранит актуальную информацию о кампаниях с TTL 5 минут для campaigns и 24 часа для impression counters по дням.

##  Patterns Надежности

RTB Service использует Circuit Breaker паттерн для вызовов к Campaign Service. Если Campaign Service возвращает ошибку, счетчик ошибок увеличивается. После 5 последовательных ошибок Circuit Breaker переходит в состояние OPEN, и все следующие запросы падают fast-fail за 1 миллисекунду без попыток соединения. Через 30 секунд Circuit Breaker переходит в Half-Open состояние и делает пробный запрос. Если он успешен, Circuit Breaker закрывается.

RTB Service реализует Timeout паттерн со строгой иерархией. DSP партнер дает 100 миллисекунд на весь request. API Gateway использует 1 миллисекунду из этого, оставляя 99 миллисекунд для RTB Service. RTB Service использует 4 миллисекунды на overhead, оставляя 95 миллисекунд на операции. Campaign fetch имеет timeout 5 миллисекунд, auction 30 миллисекунд, render 10 миллисекунд, response build 3 миллисекунды.

RTB Service реализует Fallback стратегии. Если Campaign Service недоступна, используется Redis cache с campaigns, полученными ранее. Если cache miss, используются last-known campaigns, загруженные при старте сервиса. Если все fallback'и исчерпаны, возвращается default bid за 1 миллисекунду. В результате latency может деградировать на 20-30 процентов, но система продолжает работать.

RTB Service гарантирует Idempotency всех операций. Каждое событие имеет уникальный event_id и timestamp. Kafka consumer может упорядочить события и пропустить дубликаты, используя event_id для деduplication. Это обеспечивает at-least-once delivery гарантию.

##  Мониторинг и Метрики

RTB Service отправляет метрики для мониторинга P99 latency, P99.9 latency, общий throughput в RPS, количество active requests (target меньше 100), ошибки Campaign Service, cache misses, total impressions, average bid price, winning campaigns distribution.

Все эти метрики отправляются в Prometheus и алерт срабатывает, если P99 превышает 80 миллисекунд или P99.9 превышает 120 миллисекунд.

##  Развертывание и Масштабирование

RTB Service развертывается в Kubernetes с 3-5 replicas для обеспечения высокой доступности. Каждая replica имеет CPU запрос 500 миллиcore и лимит 2000 миллиcore, память запрос 512 МегаБайт и лимит 2 ГигаБайта.

RTB Service использует liveness probe для проверки, что под еще живой (проверяет /health/live endpoint каждые 10 секунд). Если probe не отвечает, Kubernetes перезагружает под.

RTB Service использует readiness probe для проверки, что под готов принимать трафик (проверяет /health/ready endpoint каждые 5 секунд). Если probe не отвечает, Kubernetes удаляет этот под из Load Balancer.

Rolling update настроена с max surge 1 pod и max unavailable 0, обеспечивая zero-downtime развертывание.

---

Опубликовано: 2026-06-22

