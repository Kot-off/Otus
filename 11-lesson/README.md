# Мутации данных и манипуляции с партициями в ClickHouse

## Содержание

- [Мутации данных и манипуляции с партициями в ClickHouse](#мутации-данных-и-манипуляции-с-партициями-в-clickhouse)
	- [Содержание](#содержание)
	- [ Цель](#-цель)
	- [ Выполнение задания](#-выполнение-задания)
		- [ 1. Создание таблицы](#-1-создание-таблицы)
		- [ 2. Заполнение таблицы](#-2-заполнение-таблицы)
		- [ 3. Выполнение мутаций](#-3-выполнение-мутаций)
		- [ 4. Проверка результатов](#-4-проверка-результатов)
		- [ 5. Манипуляции с партициями](#-5-манипуляции-с-партициями)
		- [ 6. Проверка состояния таблицы](#-6-проверка-состояния-таблицы)
	- [ Дополнительные задания (по желанию)](#-дополнительные-задания-по-желанию)
		- [ 1. Создание новой партиции и вставка данных](#-1-создание-новой-партиции-и-вставка-данных)
		- [ 2. Использование TTL](#-2-использование-ttl)
		- [ 3. Другие типы мутаций](#-3-другие-типы-мутаций)
	- [ Полезные команды для мониторинга](#-полезные-команды-для-мониторинга)

## <a name="цель"></a> Цель

- Понять, как работают мутации данных в ClickHouse
- Научиться управлять партициями таблиц и выполнять операции с ними

## <a name="выполнение-задания"></a> Выполнение задания

### <a name="1-создание-таблицы"></a> 1. Создание таблицы

```sql
CREATE TABLE user_activity
(
    user_id UInt32,
    activity_type String,
    activity_date DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(activity_date)
ORDER BY (user_id, activity_date);
```

### <a name="2-заполнение-таблицы"></a> 2. Заполнение таблицы

```sql
INSERT INTO user_activity VALUES
(1, 'login', '2023-01-01 10:00:00'),
(2, 'logout', '2023-01-15 11:30:00'),
(3, 'purchase', '2023-02-05 14:15:00'),
(1, 'purchase', '2023-02-10 09:45:00'),
(2, 'login', '2023-03-20 16:20:00');
```

### <a name="3-выполнение-мутации"> 3. Выполнение мутаций

```sql
ALTER TABLE user_activity
UPDATE activity_type = 'view'
WHERE user_id = 1 AND activity_date = '2023-01-01 10:00:00';
```

### <a name="4-проверка-результатов"> 4. Проверка результатов

```sql
SELECT * FROM user_activity WHERE user_id = 1;

-- Проверка статуса мутации
SELECT *
FROM system.mutations
WHERE table = 'user_activity';
```

### <a name="5-манипуляции-с-партициями"> 5. Манипуляции с партициями

```sql
-- Удаление партиции за февраль 2023 (202302)
ALTER TABLE user_activity DROP PARTITION 202302;
```

### <a name="6-Проверка-состояния-таблицы"> 6. Проверка состояния таблицы

```sql
SELECT * FROM user_activity;

-- Проверка оставшихся партиций
SELECT partition, name, active
FROM system.parts
WHERE table = 'user_activity';
```

## <a name="Дополнительные-задания-по-желанию"> Дополнительные задания (по желанию)

### <a name="1-Создание-новой-партиции-и-вставка-данных"> 1. Создание новой партиции и вставка данных

```sql
-- Вставка автоматически создаст новую партицию
INSERT INTO user_activity VALUES
(4, 'login', '2023-04-01 08:00:00');
```

### <a name="2-Использование-TTL"> 2. Использование TTL

```sql
-- Создание таблицы с TTL
CREATE TABLE user_activity_ttl
(
    user_id UInt32,
    activity_type String,
    activity_date DateTime
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(activity_date)
ORDER BY (user_id, activity_date)
TTL activity_date + INTERVAL 3 MONTH
SETTINGS merge_with_ttl_timeout = 86400; -- Проверка TTL раз в день
```

### <a name="3-Другие-типы-мутаций"> 3. Другие типы мутаций

```sql
-- Удаление строк
ALTER TABLE user_activity DELETE WHERE user_id = 2;

-- Добавление столбца
ALTER TABLE user_activity ADD COLUMN device String DEFAULT 'unknown';
```

## <a name="Полезные-команды-для-мониторинга"> Полезные команды для мониторинга

```sql
-- Просмотр всех партиций
SELECT partition, count() as parts_count, sum(rows) as rows_count
FROM system.parts
WHERE table = 'user_activity' AND active
GROUP BY partition;

-- Просмотр всех мутаций
SELECT *
FROM system.mutations
WHERE table = 'user_activity';
```

<a name="примечания"></a> Примечания

- Мутации в ClickHouse выполняются асинхронно
- Для проверки статуса мутации используйте system.mutations
- Операции с партициями выполняются мгновенно
