# Стратегия Кэширования (Redis)

## Что Кэшировать

Campaigns: ключ campaigns:{campaign_id}, TTL 5 минут. Около 10 тысяч кампаний × 1 КБ = 10 МБ памяти. Cache hit rate целевой 95%.

Advertisers: ключ advertisers:{advertiser_id}, TTL 10 минут. Около 1000 рекламодателей, 200 байт каждого.

Bid floors: ключ bidfloors:{geo}:{category}, TTL 30 минут. 100 entries, 100 байт каждого.

Impression counters: ключ impressions:count:{campaign_id}:{date}, TTL 24 часа (автоматический сброс в полночь UTC).

## Cache Invalidation

TTL-based: автоматическое удаление через установленное время.

Event-based: при обновлении кампании в Campaign Service, событие campaign.updated публикуется в Kafka. RTB Service consumer получает событие и удаляет соответствующий кэш DEL campaigns:{campaign_id}.

Гибридный подход: TTL для автоматического cleanup (защита от stale данных) + events для immediate invalidation (быстрое обновление при изменениях).

## Cache Warming

При старте RTB Service загружаются все активные campaigns из БД в Redis. Это занимает 2-3 секунды. После этого cache ready к использованию, нет холодного старта.

Периодический refresh каждые 5 минут: перезагружаются все campaigns из БД и обновляются в Redis.

## Fallback

Если Redis недоступна, RTB Service падает в режим работы без кэша. Читает campaigns напрямую из PostgreSQL (медленнее на 5-10ms, но продолжает работать).

## Мониторинг

Метрики: cache hit ratio (целевой > 95%), evictions per second (целевой < 1), memory used (целевой < 512 MB для MVP), p99 latency (целевой < 2ms). Alerts если hit ratio < 80% (признак cache miss surge), если memory > 1GB (признак утечки).

---

Опубликовано: 2026-06-22
