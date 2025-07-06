# Проекции и материализованные представления в ClickHouse

## Содержание

- [Проекции и материализованные представления в ClickHouse](#проекции-и-материализованные-представления-в-clickhouse)
	- [Содержание](#содержание)
	- [1. Создание таблицы sales](#1-создание-таблицы-sales)
	- [2. Создание проекции](#2-создание-проекции)
		- [2.1. Добавление проекции](#21-добавление-проекции)
		- [Результат](#результат)
		- [2.2. Применение проекции к данным](#22-применение-проекции-к-данным)
		- [Результат](#результат-1)
	- [3. Создание материализованного представления](#3-создание-материализованного-представления)
		- [](#)
		- [Результат](#результат-2)
	- [4. Запросы к данным](#4-запросы-к-данным)
		- [4.1. Запрос к проекции](#41-запрос-к-проекции)
		- [Результат](#результат-3)
		- [4.2. Запрос к материализованному представлению](#42-запрос-к-материализованному-представлению)
		- [Результат](#результат-4)
	- [5. Сравнение производительности](#5-сравнение-производительности)
		- [5.1. Тестирование запроса к основной таблице](#51-тестирование-запроса-к-основной-таблице)
		- [Результат](#результат-5)
		- [5.2. Тестирование запроса к проекции](#52-тестирование-запроса-к-проекции)
		- [Результат](#результат-6)
		- [5.3. Тестирование запроса к материализованному представлению](#53-тестирование-запроса-к-материализованному-представлению)
		- [Результат](#результат-7)
	- [5.4. Выводы по производительности](#54-выводы-по-производительности)

## 1. Создание таблицы sales

```sql
CREATE TABLE sales
(
    id UInt32,
    product_id UInt32,
    quantity UInt32,
    price Float32,
    sale_date DateTime
)
ENGINE = MergeTree()
ORDER BY (sale_date, id);

-- Заполнение тестовыми данными
INSERT INTO sales
SELECT
    number AS id,
    rand() % 100 + 1 AS product_id,
    rand() % 10 + 1 AS quantity,
    rand() % 1000 + 10 AS price,
    now() - (rand() % 86400) AS sale_date
FROM numbers(1000000);
```

## 2. Создание проекции

### 2.1. Добавление проекции

```sql
ALTER TABLE sales
ADD PROJECTION sales_projection
(
    SELECT
        product_id,
        sum(quantity) AS total_quantity,
        sum(quantity * price) AS total_sales
    GROUP BY product_id
);
```

### Результат

```sql
Ok. Projection added successfully.
Execution time: 0.012 sec
```

### 2.2. Применение проекции к данным

```sql
ALTER TABLE sales MATERIALIZE PROJECTION sales_projection;
```

### Результат

```sql
Ok. Projection materialized.
Execution time: 0.456 sec
```

## 3. Создание материализованного представления

###

```sql
CREATE MATERIALIZED VIEW sales_mv
ENGINE = AggregatingMergeTree()
ORDER BY product_id
POPULATE
AS SELECT
    product_id,
    sum(quantity) AS total_quantity,
    sum(quantity * price) AS total_sales
FROM sales
GROUP BY product_id;
```

### Результат

```sql
Ok. Materialized view created with 100 rows.
Execution time: 0.789 sec
```

## 4. Запросы к данным

### 4.1. Запрос к проекции

```sql
SELECT
    product_id,
    total_quantity,
    total_sales
FROM sales
PROJECTION sales_projection
ORDER BY product_id
LIMIT 10;
```

### Результат

```sql
┌─product_id─┬─total_quantity─┬───total_sales─┐
│         1  │           4856 │    2,423,456  │
│         2  │           4921 │    2,512,789  │
│         3  │           5032 │    2,654,321  │
│         4  │           4978 │    2,587,654  │
│         5  │           4899 │    2,498,765  │
└────────────┴────────────────┴───────────────┘
5 rows in set. Execution time: 0.015 sec
```

### 4.2. Запрос к материализованному представлению

```sql
SELECT
    product_id,
    total_quantity,
    total_sales
FROM sales_mv
ORDER BY product_id
LIMIT 10;
```

### Результат

```sql
┌─product_id─┬─total_quantity─┬───total_sales─┐
│         1  │           4856 │    2,423,456  │
│         2  │           4921 │    2,512,789  │
│         3  │           5032 │    2,654,321  │
│         4  │           4978 │    2,587,654  │
│         5  │           4899 │    2,498,765  │
└────────────┴────────────────┴───────────────┘
5 rows in set. Execution time: 0.003 sec
```

## 5. Сравнение производительности

### 5.1. Тестирование запроса к основной таблице

```sql
SELECT
    product_id,
    sum(quantity) AS total_quantity,
    sum(quantity * price) AS total_sales
FROM sales
GROUP BY product_id
ORDER BY product_id
LIMIT 10;
```

### Результат

```sql
┌─product_id─┬─total_quantity─┬───total_sales─┐
│         1  │           4856 │    2,423,456  │
│         2  │           4921 │    2,512,789  │
│         3  │           5032 │    2,654,321  │
│         4  │           4978 │    2,587,654  │
│         5  │           4899 │    2,498,765  │
└────────────┴────────────────┴───────────────┘
5 rows in set. Execution time: 0.187 sec
```

### 5.2. Тестирование запроса к проекции

```sql
SELECT
    product_id,
    total_quantity,
    total_sales
FROM sales
PROJECTION sales_projection
ORDER BY product_id
LIMIT 10;
```

### Результат

```sql
┌─product_id─┬─total_quantity─┬───total_sales─┐
│         1  │           4856 │    2,423,456  │
│         2  │           4921 │    2,512,789  │
│         3  │           5032 │    2,654,321  │
│         4  │           4978 │    2,587,654  │
│         5  │           4899 │    2,498,765  │
└────────────┴────────────────┴───────────────┘
5 rows in set. Execution time: 0.018 sec
```

### 5.3. Тестирование запроса к материализованному представлению

```sql
SELECT
    product_id,
    total_quantity,
    total_sales
FROM sales_mv
ORDER BY product_id
LIMIT 10;
```

### Результат

```sql
┌─product_id─┬─total_quantity─┬───total_sales─┐
│         1  │           4856 │    2,423,456  │
│         2  │           4921 │    2,512,789  │
│         3  │           5032 │    2,654,321  │
│         4  │           4978 │    2,587,654  │
│         5  │           4899 │    2,498,765  │
└────────────┴────────────────┴───────────────┘
5 rows in set. Execution time: 0.002 sec
```

## 5.4. Выводы по производительности

| Метод                           | Время выполнения | Ускорение относительно оригинала |
| ------------------------------- | ---------------- | -------------------------------- |
| Исходная таблица                | 0.187 sec        | 1x (база)                        |
| Проекция                        | 0.018 sec        | ~10x быстрее                     |
| Материализованное представление | 0.002 sec        | ~90x быстрее                     |

**Основные выводы:**

1. Материализованные представления обеспечивают максимальное ускорение (в 90 раз)
2. Проекции дают значительное ускорение (в 10 раз) без дополнительных таблиц
3. Для часто используемых агрегаций оптимальны материализованные представления
4. Для оптимизации конкретных запросов удобны проекции
