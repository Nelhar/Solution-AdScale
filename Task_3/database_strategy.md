# Стратегия Базы Данных

## Master-Slave Репликация

PostgreSQL Master содержит все данные для написания. PostgreSQL Slave реплицирует данные асинхронно и используется только для чтения. Репликация основана на binary log и может отставать на 1-2 минуты.

Master отвечает за: campaigns, budgets, bids, financial ledger, user data (OLTP).

Slave отвечает за: Analytics queries, reporting, read-heavy операции (OLAP).

Если Master падает, можно продвинуть Slave на роль Primary за 30-60 секунд.

## Connection Pooling

PgBouncer используется для multiplexing соединений. Максимум 1000 клиентских соединений, 100 соединений в пуле на одного backend'а. Transaction-level pooling обеспечивает переиспользование соединений между транзакциями.

## Sharding (Будущее)

Ключ sharding: advertiser_id. Формула: shard_id = advertiser_id % num_shards. Начинаем с 1 shard (текущее), затем 2, потом 4. Миграция данных при добавлении шардов не требует downtime (хеширование позволяет плавный переход).

## Query Optimization

Индексы создаются для hot queries: idx_campaigns_active (на is_active), idx_impressions_date (на created_at для Analytics). Slow query log настроен на 1000ms порог. Все queries > 1000ms логируются для анализа.

## Backup и PITR

Дневной полный backup в S3 в 01:00 UTC. WAL архив каждые 30 минут. Retention: 30 дней полных backups, 7 дней WAL. Можно восстановиться в любую точку времени в пределах 7 дней.

## Мониторинг

Метрики: active connections, replication lag (alarm если > 10s), slowest queries, unused indexes, cache hit ratio. Alerts если replication lag > 10s или master down.

