# Репликация и удаление данных в ClickHouse

## Содержание

- [Репликация и удаление данных в ClickHouse](#репликация-и-удаление-данных-в-clickhouse)
	- [Содержание](#содержание)
	- [Настройка реплицируемой таблицы](#настройка-реплицируемой-таблицы)
		- [Создаем кластер с тремя репликами:](#создаем-кластер-с-тремя-репликами)
		- [Проверка репликации](#проверка-репликации)
	- [Настройка TTL](#настройка-ttl)
	- [Системные запросы](#системные-запросы)
	- [Итоговая структура таблицы](#итоговая-структура-таблицы)
		- [Результат:](#результат)

## Настройка реплицируемой таблицы

### Создаем кластер с тремя репликами:

```sql
CREATE CLUSTER homework_cluster (
    'shard1' = HOST(),
    'shard2' = HOST(),
    'shard3' = HOST()
) SETTINGS cluster_replicas = 3;

CREATE TABLE uk_price_paid_replicated
(
    price UInt32,
    date Date,
    postcode1 LowCardinality(String),
    postcode2 LowCardinality(String),
    type Enum8('terraced' = 1, 'semi-detached' = 2, 'detached' = 3, 'flat' = 4, 'other' = 0),
    is_new UInt8,
    duration Enum8('freehold' = 1, 'leasehold' = 2, 'unknown' = 0),
    addr1 String,
    addr2 String,
    street LowCardinality(String),
    locality LowCardinality(String),
    town LowCardinality(String),
    district LowCardinality(String),
    county LowCardinality(String)
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/uk_price_paid', '{replica}')
PARTITION BY toYYYYMM(date)
ORDER BY (postcode1, postcode2, addr1, addr2);
```

### Проверка репликации

```sql
SELECT * FROM system.replicas
WHERE table = 'uk_price_paid_replicated';
```

## Настройка TTL

```sql
ALTER TABLE uk_price_paid_replicated
MODIFY TTL date + INTERVAL 7 DAY;
```

## Системные запросы

```sql
SELECT
    getMacro('replica') AS replica_name,
    *
FROM remote('shard1,shard2,shard3', system.parts)
WHERE table = 'uk_price_paid_replicated';
```

## Итоговая структура таблицы

```sql
SHOW CREATE TABLE uk_price_paid_replicated;
```

### Результат:

```sql
CREATE TABLE default.uk_price_paid_replicated
(
    `price` UInt32,
    `date` Date,
    `postcode1` LowCardinality(String),
    `postcode2` LowCardinality(String),
    `type` Enum8('terraced' = 1, 'semi-detached' = 2, 'detached' = 3, 'flat' = 4, 'other' = 0),
    `is_new` UInt8,
    `duration` Enum8('freehold' = 1, 'leasehold' = 2, 'unknown' = 0),
    `addr1` String,
    `addr2` String,
    `street` LowCardinality(String),
    `locality` LowCardinality(String),
    `town` LowCardinality(String),
    `district` LowCardinality(String),
    `county` LowCardinality(String)
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/uk_price_paid', '{replica}')
PARTITION BY toYYYYMM(date)
ORDER BY (postcode1, postcode2, addr1, addr2)
TTL date + toIntervalDay(7)
SETTINGS index_granularity = 8192
```
