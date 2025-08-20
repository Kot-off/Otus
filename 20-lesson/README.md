# Интеграция ClickHouse с PostgreSQL

## 1. В PostgreSQL (создание данных)

```sql
-- Подключение к PostgreSQL
docker-compose exec postgres psql -U airflow -d test_db

-- Создание таблицы и данных
CREATE TABLE sales (
    id SERIAL PRIMARY KEY,
    product_name VARCHAR(100),
    category VARCHAR(50),
    quantity INT,
    price DECIMAL(10,2),
    sale_date DATE
);

INSERT INTO sales (product_name, category, quantity, price, sale_date) VALUES
('Laptop', 'Electronics', 5, 1200.00, '2024-01-15'),
('Smartphone', 'Electronics', 12, 800.00, '2024-01-16'),
('Book', 'Education', 20, 25.00, '2024-01-17'),
('Chair', 'Furniture', 8, 150.00, '2024-01-18'),
('Table', 'Furniture', 3, 300.00, '2024-01-19'),
('Monitor', 'Electronics', 7, 350.00, '2024-01-20'),
('Pen', 'Office', 100, 1.50, '2024-01-21'),
('Notebook', 'Office', 30, 5.00, '2024-01-22');
```

## 2. В ClickHouse (интеграция)

```sql
-- Подключение к ClickHouse
docker-compose exec clickhouse clickhouse-client

-- 1. Проверка подключения через функцию postgresql()
SELECT *
FROM postgresql(
    'postgres:5432',  -- host:port
    'test_db',        -- database
    'sales',          -- table
    'airflow',        -- username
    'airflow',        -- password
    'public'          -- schema
) LIMIT 5;

-- 2. Создание базы данных для интеграции
CREATE DATABASE pg_integration;

-- 3. Создание таблицы с движком PostgreSQL
CREATE TABLE pg_sales
(
    id UInt32,
    product_name String,
    category String,
    quantity Int32,
    price Decimal(10,2),
    sale_date Date
)
ENGINE = PostgreSQL(
    'postgres:5432',  -- host:port
    'test_db',        -- database
    'sales',          -- table
    'airflow',        -- username
    'airflow'         -- password
);

-- 4. Проверка данных из интегрированной таблицы
SELECT * FROM pg_sales ORDER BY id LIMIT 5;

-- 5. Агрегация данных из PostgreSQL в ClickHouse
SELECT
    category,
    sum(quantity) as total_quantity,
    sum(quantity * price) as total_revenue
FROM pg_sales
GROUP BY category
ORDER BY total_revenue DESC;

-- 6. Проверка структуры таблицы
DESCRIBE TABLE pg_sales;

-- 7. Проверка количества записей
SELECT count() FROM pg_sales;
```

## Итоговые результаты:

✅ **Базовая интеграция ClickHouse с PostgreSQL настроена успешно**
✅ **Данные из PostgreSQL доступны в ClickHouse в реальном времени**
✅ **Создана база данных `pg_integration` для интеграции**
✅ **Создана таблица `pg_sales` с движком PostgreSQL**
✅ **Выполнены тестовые запросы с агрегацией данных**

**Ключевые особенности вашей конфигурации:**

- Хост PostgreSQL: `postgres` (имя сервиса в docker-compose)
- Порт: `5432`
- База данных: `test_db`
- Пользователь: `airflow`
- Пароль: `airflow`
- Функция в ClickHouse: `postgresql()` (не `postgres()`)
- Формат параметров: `'host:port', 'database', 'table', 'username', 'password'`

Интеграция работает корректно! 🎉
