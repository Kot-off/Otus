# 📊 Домашнее задание: Работа с функциями в ClickHouse

## 📚 Содержание

- [🎯 Цель](#-цель)
- [📁 Набор данных](#-набор-данных)
- [🔹 Агрегатные функции](#-агрегатные-функции)
  - [1. Общий доход от всех операций](#1-общий-доход-от-всех-операций)
  - [2. Средний доход с одной сделки](#2-средний-доход-с-одной-сделки)
  - [3. Общее количество проданной продукции](#3-общее-количество-проданной-продукции)
  - [4. Количество уникальных пользователей](#4-количество-уникальных-пользователей)
- [🔹 Функции для работы с типами данных](#-функции-для-работы-с-типами-данных)
  - [1. Преобразование даты в строку](#1-преобразование-даты-в-строку)
  - [2. Извлечение года и месяца](#2-извлечение-года-и-месяца)
  - [3. Округление цены](#3-округление-цены)
  - [4. Преобразование ID в строку](#4-преобразование-id-в-строку)
- [🔹 User-Defined Functions (UDF)](#-user-defined-functions-udf)
  - [1. Расчёт общей стоимости](#1-расчёт-общей-стоимости)
  - [2. Категоризация транзакций](#2-категоризация-транзакций)
- [✅ Результат](#-результат)
- [📎 Как представить](#-как-представить)

---

## 🎯 Цель

Цель этого задания — научиться применять:

- Агрегатные функции
- Функции для работы с типами данных
- Функции, определяемые пользователем (UDF), в ClickHouse

---

## 📁 Набор данных

Исходная таблица:

```sql
CREATE TABLE transactions (
    transaction_id UInt32,
    user_id UInt32,
    product_id UInt32,
    quantity UInt8,
    price Float32,
    transaction_date Date
) ENGINE = MergeTree()
ORDER BY (transaction_id);


## Агрегатные функции

### 1. Общий доход от всех операций

```

SELECT SUM(quantity \* price) AS total_revenue
FROM transactions;

```

### 2. Средний доход с одной сделки

```

SELECT AVG(quantity \* price) AS avg_transaction_revenue
FROM transactions;

```

### 3. Общее количество проданной продукции

```

SELECT SUM(quantity) AS total_quantity_sold
FROM transactions;

```

### 4. Количество уникальных пользователей, совершивших покупку

```

SELECT COUNT(DISTINCT user_id) AS unique_users
FROM transactions;

```

## Функции для работы с типами данных

### 1. Преобразование transaction_date в строку формата YYYY-MM-DD

```

SELECT transaction_id,
formatDateTime(transaction_date, '%Y-%m-%d') AS transaction_date_str
FROM transactions
LIMIT 5;

```

### 2. Извлечение года и месяца из transaction_date

```

SELECT transaction_id,
toYear(transaction_date) AS year,
toMonth(transaction_date) AS month
FROM transactions
LIMIT 5;

```

### 3. Округление price до ближайшего целого

```

SELECT transaction_id,
price,
round(price) AS rounded_price
FROM transactions
LIMIT 5;

```

### 4. Преобразование transaction_id в строку

```

SELECT transaction_id,
toString(transaction_id) AS transaction_id_str
FROM transactions
LIMIT 5;

```

## User-Defined Functions (UDFs)

ClickHouse не поддерживает пользовательские функции в привычном SQL-стиле, но вы можете использовать выражения или внешние функции через clickhouse-local или lambda-функции (внутри array/map функций). Ниже — имитация UDF с помощью выражений.

### 1. "UDF" для расчета общей стоимости транзакции

```

SELECT transaction_id,
quantity \* price AS total_price
FROM transactions
LIMIT 5;

```

### 2. "UDF" для классификации транзакций (порог = 100)

```

SELECT transaction_id,
quantity _ price AS total_price,
IF(quantity _ price >= 100, 'высокоценная', 'малоценная') AS transaction_category
FROM transactions
LIMIT 10;

```

###

```

```

###

```

```

```
