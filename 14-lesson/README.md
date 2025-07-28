# Мониторинг и оптимизация ClickHouse

## Содержание

- [Мониторинг и оптимизация ClickHouse](#мониторинг-и-оптимизация-clickhouse)
	- [Содержание](#содержание)
	- [Вариант 1: Персонализированный мониторинг](#вариант-1-персонализированный-мониторинг)
		- [Создание таблицы с запросами мониторинга](#создание-таблицы-с-запросами-мониторинга)
		- [Пример](#пример)
	- [Вариант 2: Настройка Prometheus](#вариант-2-настройка-prometheus)
		- [Конфигурация ClickHouse](#конфигурация-clickhouse)
		- [Конфигурация Prometheus](#конфигурация-prometheus)
	- [Репликация логов](#репликация-логов)
		- [](#)
		- [Проверка](#проверка)
		- [Реп 1](#реп-1)
		- [Реп 2](#реп-2)
		- [Результат](#результат)
		- [Каждая реплика zookeeper](#каждая-реплика-zookeeper)
	- [Оптимизация производительности](#оптимизация-производительности)
		- [Управление памятью](#управление-памятью)
		- [Контроль параллельных запросов:](#контроль-параллельных-запросов)
		- [Мониторинг ресурсов:](#мониторинг-ресурсов)

---

## Вариант 1: Персонализированный мониторинг

### Создание таблицы с запросами мониторинга

```sql
CREATE TABLE monitoring.queries (
    query_name String,
    query_text String,
    update_interval UInt32,
    last_executed DateTime
) ENGINE = MergeTree()
ORDER BY query_name;
```

### Пример

```sql
INSERT INTO monitoring.queries VALUES
('Active Queries', 'SELECT count() FROM system.processes', 5, now()),
('Memory Usage', 'SELECT formatReadableSize(sum(memory_usage)) FROM system.processes', 10, now()),
('Slow Queries', 'SELECT query, elapsed FROM system.query_log WHERE elapsed > 1 ORDER BY elapsed DESC LIMIT 5', 60, now()),
('Replication Status', 'SELECT table, absolute_delay FROM system.replicas WHERE is_readonly', 30, now());
```

## Вариант 2: Настройка Prometheus

### Конфигурация ClickHouse

```xml
<prometheus>
    <endpoint>/metrics</endpoint>
    <port>9363</port>
    <metrics>true</metrics>
    <events>true</events>
    <asynchronous_metrics>true</asynchronous_metrics>
</prometheus>
```

### Конфигурация Prometheus

```yaml
scrape_configs:
  - job_name: 'clickhouse'
    scrape_interval: 15s
    static_configs:
      - targets: ['clickhouse-host:9363']
```

## Репликация логов

###

```sql
-- Таблица-источник (не хранит данные)
CREATE TABLE logs.source (
    event_time DateTime,
    log_level Enum8('INFO'=1, 'WARNING'=2, 'ERROR'=3),
    message String,
    source String
) ENGINE = Null;

-- Реплицируемая таблица
CREATE TABLE logs.replicated (
    event_time DateTime,
    log_level Enum8('INFO'=1, 'WARNING'=2, 'ERROR'=3),
    message String,
    source String,
    replica_name String MATERIALIZED hostName()
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/logs', '{replica}')
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_time, log_level);

-- Материализованное представление
CREATE MATERIALIZED VIEW logs.mv TO logs.replicated
AS SELECT * FROM logs.source;
```

### Проверка

### Реп 1

```sql
INSERT INTO logs.source VALUES (now(), 'INFO', 'Test message', 'app1');
```

### Реп 2

```sql
SELECT * FROM logs.replicated;
```

### Результат

┌───────────event_time─┬─log_level─┬─message──────┬─source─┬─replica_name─┐
│ 2023-05-15 12:00:00 │ INFO │ Test message │ app1 │ replica2 │
└─────────────────────┴───────────┴──────────────┴────────┴──────────────┘

### Каждая реплика zookeeper

```xml
<zookeeper>
    <node>
        <host>zookeeper</host>
        <port>2181</port>
    </node>
</zookeeper>
```

## Оптимизация производительности

### Управление памятью

```sql
SET max_memory_usage = 10000000000; -- 10GB
```

### Контроль параллельных запросов:

```sql
SET max_concurrent_queries = 100;
```

### Мониторинг ресурсов:

```sql
SELECT metric, value FROM system.asynchronous_metrics
WHERE metric LIKE '%Memory%' OR metric LIKE '%CPU%';
```
