# Event-Driven Архитектура и Kafka

## Kafka Топики

impressions.recorded: каждый показанный баннер. Partition key: campaign_id (гарантирует порядок событий внутри кампании). Retention: 1 день. Replication factor: 3.

clicks.recorded: каждый клик на баннер. Partition key: campaign_id. Retention: 1 день.

transactions.completed: финансовые транзакции. Partition key: campaign_id. Retention: 7 дней (финансовые данные хранятся дольше).

campaigns.updated: обновление кампании. Retention: 1 час (для invalidation cache).

## Event Schema

Каждое событие содержит event_id (UUID для deduplication), event_type, timestamp, correlation_id (для tracing), и business_data (impression details, click info и т.д.). JSON формат.

## Consumer Groups

analytics-impressions: потребляет impressions.recorded для агрегации статистики. Обработка может быть delayed (100-500ms OK). Сохраняет offset в Kafka.

financial-impressions: потребляет impressions.recorded для обновления budget ledger. Обработка должна быть fast (< 1s). Использует exactly-once processing с deduplication по event_id.

archival-events: потребляет все события для долгосрочного хранения.

## Масштабирование

MVP: 1 Kafka broker достаточно (throughput 100K msg/sec, нам нужно 18K).

6 месяцев: 3 Kafka broker для HA (replication factor = 3).

Partitions: начинаем с 10 партиций, растим с нагрузкой. Каждая партиция может обслуживать 1 consumer.

## Мониторинг

Метрики: messages published per second (целевой 18K), consumer lag (alert если > 60s), replication lag (alert если > 100ms), partition rebalancing events. Alerts если throughput падает или lag растет.


