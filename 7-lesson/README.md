# Работа со словарями и оконными функциями в ClickHouse

- [Работа со словарями и оконными функциями в ClickHouse](#работа-со-словарями-и-оконными-функциями-в-clickhouse)
  - [1. Обзор проекта](#1-обзор-проекта)
  - [2. Инструкции по настройке](#2-инструкции-по-настройке)
    - [2.1. Создание таблиц](#21-создание-таблиц)
    - [2.2 Создание словаря user_emails_dict](#22-создание-словаря-user_emails_dict)
    - [2.3. Загрузка тестовых данных](#23-загрузка-тестовых-данных)
  - [3. Примеры запросов](#3-примеры-запросов)
    - [3.1. Получение email с аккумулятивной суммой расходов](#31-получение-email-с-аккумулятивной-суммой-расходов)
    - [Результат:](#результат)
    - [3.2. Альтернативные варианты запросов](#32-альтернативные-варианты-запросов)
    - [3.3. Анализ расходов по действиям](#33-анализ-расходов-по-действиям)

## 1. Обзор проекта

Проект демонстрирует работу со словарями и оконными функциями в ClickHouse. Основные задачи:

- Создание таблицы для хранения действий пользователей
- Организация словаря для быстрого доступа к email пользователей
- Использование оконных функций для анализа накопленных расходов

## 2. Инструкции по настройке

### 2.1. Создание таблиц

```sql
CREATE TABLE user_actions (
    user_id UInt64,
    action String,
    expense UInt64
) ENGINE = MergeTree()
ORDER BY (user_id, action);
```

### 2.2 Создание словаря user_emails_dict

```sql
-- Создаем словарь для хранения email пользователей
CREATE DICTIONARY user_emails_dict (
    user_id UInt64,
    email String
)
PRIMARY KEY user_id
SOURCE(FILE(
    path '/path/to/user_emails.csv'
    format 'CSVWithNames'
))
LIFETIME(MIN 300 MAX 360)
LAYOUT(HASHED());
```

### 2.3. Загрузка тестовых данных

```sql
INSERT INTO user_actions VALUES
(1, 'click', 10),
(1, 'view', 5),
(2, 'click', 15),
(2, 'purchase', 200),
(3, 'view', 8),
(3, 'click', 20),
(1, 'purchase', 100),
(2, 'view', 8),
(3, 'purchase', 150),
(1, 'click', 12);
```

## 3. Примеры запросов

### 3.1. Получение email с аккумулятивной суммой расходов

```sql
SELECT
    dictGet('user_emails_dict', 'email', user_id) AS email,
    action,
    expense,
    sum(expense) OVER (PARTITION BY action ORDER BY email) AS cumulative_expense
FROM user_actions
ORDER BY email;
```

### Результат:

```sql
Результат:
email	action	expense	cumulative_expense
user1@example.com	click	10	42
user1@example.com	view	5	13
user1@example.com	purchase	100	450
user2@example.com	click	15	42
user2@example.com	view	8	13
user2@example.com	purchase	200	450
user3@example.com	click	20	42
user3@example.com	view	8	13
user3@example.com	purchase	150	450
```

### 3.2. Альтернативные варианты запросов

```sql
SELECT
    dictGet('user_emails_dict', 'email', user_id) AS email,
    sum(expense) AS total_expense
FROM user_actions
GROUP BY email
ORDER BY total_expense DESC;
```

### 3.3. Анализ расходов по действиям

```sql
SELECT *
FROM (
    SELECT
        dictGet('user_emails_dict', 'email', user_id) AS email,
        action,
        expense,
        row_number() OVER (PARTITION BY email ORDER BY expense DESC) AS rank
    FROM user_actions
)
WHERE rank <= 3
ORDER BY email, rank;
```
