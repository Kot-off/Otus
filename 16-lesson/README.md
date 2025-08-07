# Профилирование запросов в ClickHouse

---

## 1. Создание таблицы

Если таблица уже есть — сначала удаляем её:

```sql
DROP TABLE IF EXISTS github_events_simple;
```

Создаём таблицу:

```sql
CREATE TABLE github_events_simple
(
    id String,
    type String,
    actor String,
    repo String,
    created_at String
)
ENGINE = MergeTree()
ORDER BY created_at;
```

---

## 2. Загрузка данных

В терминале Linux (где есть скачанный файл `2015-01-01-15.json.gz`):

```bash
zcat 2015-01-01-15.json.gz | clickhouse-client --query="INSERT INTO github_events_simple FORMAT JSONEachRow"
```

---

## 3. Выполнение запросов для профилирования

### Запрос без использования первичного ключа (фильтрация по `type`):

```sql
SELECT count()
FROM github_events_simple
WHERE type = 'PushEvent';
```

---

### Запрос с использованием первичного ключа (фильтрация по `created_at`):

```sql
SELECT count()
FROM github_events_simple
WHERE created_at LIKE '2015-01-01T15:%';
```

---

## 4. Анализ запросов через EXPLAIN

```sql
EXPLAIN SELECT count() FROM github_events_simple WHERE type = 'PushEvent';
```

```sql
EXPLAIN SELECT count() FROM github_events_simple WHERE created_at LIKE '2015-01-01T15:%';
```

---

## 5. Просмотр логов запросов (если включены)

```sql
SELECT
    query,
    read_rows,
    query_duration_ms,
    ProfileEvents['ReadBufferFromFileDescriptorReadBytes'] AS read_bytes
FROM system.query_log
WHERE type = 'QueryFinish'
ORDER BY event_time DESC
LIMIT 10;
```

---

## 6. (Опционально) Проверка наличия логов запросов

```sql
SELECT count(*) FROM system.query_log;
```

Если 0 — значит логи запросов не включены в конфиге ClickHouse.

---

# Итог

- Выполни создание таблицы и загрузку данных.
- Сделай два запроса: с и без использования PK.
- Получи EXPLAIN для обоих запросов.
- Посмотри логи запросов (если они есть).
- Сравни производительность и использование индексов.

---
