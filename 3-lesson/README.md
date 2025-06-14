# Домашнее задание: Язык запросов SQL

## Содержание

- [Домашнее задание: Язык запросов SQL](#домашнее-задание-язык-запросов-sql)
	- [Содержание](#содержание)
	- [Шаг 1. Создание базы данных](#шаг-1-создание-базы-данных)
	- [Шаг 2. Создание таблицы "Автомобили"](#шаг-2-создание-таблицы-автомобили)
	- [Шаг 3. Наполнение таблицы данными](#шаг-3-наполнение-таблицы-данными)
	- [Шаг 4. Тестирование CRUD операций](#шаг-4-тестирование-crud-операций)
	- [Шаг 5. Изменение структуры таблицы](#шаг-5-изменение-структуры-таблицы)
	- [Шаг 6. Выборка данных из sample dataset](#шаг-6-выборка-данных-из-sample-dataset)
	- [Шаг 7. Материализация таблицы](#шаг-7-материализация-таблицы)
	- [Шаг 8. Работа с партициями](#шаг-8-работа-с-партициями)

## Шаг 1. Создание базы данных

```
sql
CREATE DATABASE auto_db;
USE auto_db;
```

## Шаг 2. Создание таблицы "Автомобили"

```
CREATE TABLE cars (
    car_id UInt32,
    brand LowCardinality(String) COMMENT 'Марка автомобиля',
    model String COMMENT 'Модель автомобиля',
    year UInt16 COMMENT 'Год выпуска',
    price Decimal(12, 2) COMMENT 'Цена в рублях',
    -- остальные поля
) ENGINE = MergeTree()
ORDER BY (brand, year, car_id);
```

## Шаг 3. Наполнение таблицы данными

```
INSERT INTO cars VALUES
(1, 'Toyota', 'Camry', 2020, 2500000.00, ...),
(2, 'BMW', 'X5', 2022, 6500000.00, ...);
```

## Шаг 4. Тестирование CRUD операций

```
-- CREATE
INSERT INTO cars VALUES (6, 'Audi', 'A6', ...);

-- READ
SELECT * FROM cars WHERE price BETWEEN 1000000 AND 3000000;

-- UPDATE
ALTER TABLE cars UPDATE price = 950000.00 WHERE car_id = 3;

-- DELETE
ALTER TABLE cars DELETE WHERE brand = 'Lada';
```

## Шаг 5. Изменение структуры таблицы

```
-- Добавление полей
ALTER TABLE cars
    ADD COLUMN color LowCardinality(String),
    ADD COLUMN warranty_until Nullable(Date);

-- Удаление полей
ALTER TABLE cars
    DROP COLUMN is_used,
    DROP COLUMN features;
```

## Шаг 6. Выборка данных из sample dataset

```
USE default;
SELECT database, table, formatReadableSize(sum(bytes)) AS size
FROM system.parts
WHERE active
GROUP BY database, table;
```

## Шаг 7. Материализация таблицы

```
CREATE TABLE cars_copy AS cars
ENGINE = MergeTree()
ORDER BY (brand, year, car_id);

INSERT INTO cars_copy SELECT * FROM cars;
```

## Шаг 8. Работа с партициями

```
-- Создание таблицы с партициями
CREATE TABLE cars_partitioned (...) ENGINE = MergeTree()
PARTITION BY year ORDER BY (brand, car_id);

-- Операции с партициями
ALTER TABLE cars_partitioned DETACH PARTITION 2020;
ALTER TABLE cars_partitioned ATTACH PARTITION 2020;
ALTER TABLE cars_partitioned DROP PARTITION 2019;
```
