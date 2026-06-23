# Паттерны Надежности

## Circuit Breaker

Если Campaign Service возвращает ошибку, счетчик увеличивается. После 5 ошибок подряд Circuit Breaker переходит в OPEN. Все следующие requests возвращают ошибку fast-fail за 1ms. Через 30 секунд переходит в Half-Open и пробует одно соединение. Если успешно, закрывается.

## Retry Pattern

Максимум 3 попытки с exponential backoff: 10ms, 20ms, 40ms. Всего буджет 70ms из 95ms, остается 25ms buffer.

## Timeout Иерархия

100ms от DSP  99ms в RTB  95ms для логики, 5ms overhead. Campaign fetch 5ms, auction 30ms, render 10ms, response 3ms. Если timeout, используется fallback.

## Fallback Стратегии

1. Redis cache (campaigns)
2. Last-known campaigns (память)
3. Default bid (fast-fail за 1ms)

## Булкхед Паттерн

Разные thread pools per DSP partner. Google 20 threads, Programmatic 15 threads, Test 5 threads. Один партнер не может исчерпать все ресурсы.

## Health Checks

Liveness probe проверяет /health/live каждые 10 секунд. Если не отвечает, Kubernetes перезагружает pod. Readiness probe проверяет /health/ready каждые 5 секунд. Если не отвечает, pod удаляется из Load Balancer.

## Graceful Shutdown

На SIGTERM сигнал: stop accepting requests (readiness = false), wait максимум 15 секунд для in-flight requests, закрыть соединения, exit.

## Distributed Tracing

Каждый request получает correlation ID. Пропускается через все сервисы. Все логи и метрики содержат этот ID для отладки.

## Мониторинг Надежности

Метрики: P99/P99.9 latency, error rate, Circuit Breaker transitions, cache misses, timeout errors, fallback activations. Alerts если P99 > 80ms, P99.9 > 120ms, error rate > 1%.

---

Опубликовано: 2026-06-22

