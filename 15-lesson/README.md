# Метрики и мониторинг. Логирование

---

## 📝 Содержание

- [Метрики и мониторинг. Логирование](#метрики-и-мониторинг-логирование)
  - [📝 Содержание](#-содержание)
  - [🎯 Цель](#-цель)
  - [1. Запросы для мониторинга ресурсов ClickHouse](#1-запросы-для-мониторинга-ресурсов-clickhouse)
    - [🔸 Использование CPU по запросам](#-использование-cpu-по-запросам)
    - [🔸 Использование памяти по пользователям](#-использование-памяти-по-пользователям)
    - [🔸 Использование дискового пространства](#-использование-дискового-пространства)
    - [🔸 Самые тяжёлые запросы (CPU + память)](#-самые-тяжёлые-запросы-cpu--память)
  - [2. Создание таблицы для хранения мониторинга](#2-создание-таблицы-для-хранения-мониторинга)
  - [3. Вставка данных мониторинга](#3-вставка-данных-мониторинга)
  - [🛠 Что нужно:](#-что-нужно)

---

## 🎯 Цель

Настроить **персонализированный мониторинг ClickHouse** с акцентом на **оптимизацию использования ресурсов** — CPU, память, диск. Отображать результаты мониторинга в виде таблиц или графиков.

---

## 1. Запросы для мониторинга ресурсов ClickHouse

### 🔸 Использование CPU по запросам

```sql
SELECT
    query,
    sum(read_rows) AS total_rows,
    sum(read_bytes) / 1048576 AS read_mb,
    round(avg(query_duration_ms), 2) AS avg_ms
FROM system.query_log
WHERE event_date >= today() - 1
  AND type = 'QueryFinish'
GROUP BY query
ORDER BY total_rows DESC
LIMIT 10;
```

### 🔸 Использование памяти по пользователям

```sql
SELECT
    user,
    round(avg(memory_usage) / 1048576, 2) AS avg_memory_mb
FROM system.query_log
WHERE event_date >= today() - 1
  AND type = 'QueryFinish'
GROUP BY user
ORDER BY avg_memory_mb DESC;
```

### 🔸 Использование дискового пространства

```sql
SELECT
    table,
    database,
    round(sum(bytes_on_disk) / 1048576, 2) AS size_mb
FROM system.parts
GROUP BY table, database
ORDER BY size_mb DESC;
```

### 🔸 Самые тяжёлые запросы (CPU + память)

```sql
SELECT
    query,
    round(avg(query_duration_ms), 2) AS avg_duration_ms,
    round(avg(memory_usage)/1048576, 2) AS avg_memory_mb
FROM system.query_log
WHERE event_date >= today() - 1
  AND type = 'QueryFinish'
GROUP BY query
ORDER BY avg_duration_ms DESC
LIMIT 10;
```

---

## 2. Создание таблицы для хранения мониторинга

```sql
CREATE TABLE IF NOT EXISTS monitoring.query_metrics
(
    event_time     DateTime DEFAULT now(),
    metric_name    String,
    metric_value   Float64,
    metric_label   String
)
ENGINE = MergeTree()
ORDER BY event_time;
```

---

## 3. Вставка данных мониторинга

Пример вручную:

```sql
INSERT INTO monitoring.query_metrics (metric_name, metric_value, metric_label)
VALUES
('avg_memory_mb', 315.2, 'user1'),
('disk_usage_mb', 20500, 'table_orders'),
('query_duration_ms', 2400, 'SELECT * FROM logs');
```

---

## 🛠 Что нужно:

1. **Создай таблицу** с метриками (`CREATE TABLE ...`)
2. **Запусти SQL-запросы** из раздела 1
3. **Вставь результаты** в свою таблицу мониторинга
4. **Открой Web UI** ClickHouse (`http://localhost:8123`) (`http://localhost:8123/dashboard`)
5. **Выполни запрос**: `SELECT * FROM monitoring.query_metrics`
6. **Сделай скриншот** → прикрепи как `dashboard_screenshot.png` или вставь прямо в README

---
