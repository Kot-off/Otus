# Домашнее задание: Движки MergeTree в ClickHouse

## 📑 Содержание

- [Цель](#-цель)
- [tbl1 — ReplacingMergeTree](#-таблица-tbl1)
- [tbl2 — SummingMergeTree](#-таблица-tbl2)
- [tbl3 — ReplacingMergeTree (без версии)](#-таблица-tbl3)
- [tbl4 — MergeTree](#-таблица-tbl4)
- [tbl5 — AggregatingMergeTree](#-таблица-tbl5)
- [tbl6 — CollapsingMergeTree](#-таблица-tbl6)
- [Проблемы и решения](#-проблемы-и-решения)
- [Полезные ссылки](#-ссылки-на-материалы)
- [Вывод](#вывод)

---

## 🎯 Цель

- Разобрать принципы работы MergeTree и его наследников.
- Выбрать подходящий движок для различных сценариев.
- Научиться дедуплицировать данные и заменять операции `DELETE`/`UPDATE`.

---

## 📊 Таблица `tbl1`

```sql
CREATE TABLE tbl1
(
    UserID UInt64,
    PageViews UInt8,
    Duration UInt8,
    Sign Int8,
    Version UInt8
)
ENGINE = ReplacingMergeTree(Version)
ORDER BY UserID;
```

````

- **Тип:** `ReplacingMergeTree`
- **Назначение:** Удаление дублей по `UserID`, сохранение самой свежей версии.
- **Почему:** Используется поле `Version`, а значит, логика замены версий актуальна.

➡️ [Документация ClickHouse: ReplacingMergeTree](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/replacingmergetree)

---

## 📊 Таблица `tbl2`

```sql
CREATE TABLE tbl2
(
    key UInt32,
    value UInt32
)
ENGINE = SummingMergeTree
ORDER BY key;
```

- **Тип:** `SummingMergeTree`
- **Назначение:** Автоматическая агрегация чисел по ключу.
- **Почему:** Повторяющиеся `key` автоматически суммируются по `value`.

➡️ [Документация ClickHouse: SummingMergeTree](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/summingmergetree)

---

## 📊 Таблица `tbl3`

```sql
CREATE TABLE tbl3
(
    id Int32,
    status String,
    price String,
    comment String
)
ENGINE = ReplacingMergeTree
ORDER BY (id, status);
```

- **Тип:** `ReplacingMergeTree` без `Version`
- **Назначение:** Схлопывание дубликатов строк.
- **Почему:** При отсутствии `Version` одна из строк случайно останется после `FINAL`.

---

## 📊 Таблица `tbl4`

```sql
CREATE TABLE tbl4
(
    CounterID UInt8,
    StartDate Date,
    UserID UInt64
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(StartDate)
ORDER BY (CounterID, StartDate);
```

- **Тип:** `MergeTree`
- **Назначение:** Базовое хранение без агрегации.
- **Почему:** Используется для промежуточных данных.

➡️ [Документация ClickHouse: MergeTree](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/mergetree)

---

## 📊 Таблица `tbl5`

```sql
CREATE TABLE tbl5
(
    CounterID UInt8,
    StartDate Date,
    UserID AggregateFunction(uniq, UInt64)
)
ENGINE = AggregatingMergeTree
PARTITION BY toYYYYMM(StartDate)
ORDER BY (CounterID, StartDate);
```

- **Тип:** `AggregatingMergeTree`
- **Назначение:** Сохранение агрегатных состояний.
- **Почему:** Используется для дальнейшего применения `uniqMerge`.

➡️ [Документация ClickHouse: AggregatingMergeTree](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/aggregatingmergetree)

---

## 📊 Таблица `tbl6`

```sql
CREATE TABLE tbl6
(
    id Int32,
    status String,
    price String,
    comment String,
    sign Int8
)
ENGINE = CollapsingMergeTree(sign)
ORDER BY (id, status);
```

- **Тип:** `CollapsingMergeTree`
- **Назначение:** Схлопывание строк с противоположным знаком `sign`.
- **Почему:** Эмуляция логического удаления.

➡️ [Документация ClickHouse: CollapsingMergeTree](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/collapsingmergetree)

---

## 🛠 Проблемы и решения

| Проблема                                                   | Решение                                                    |
| ---------------------------------------------------------- | ---------------------------------------------------------- |
| Данные не схлопываются без `FINAL`                         | Использовать `SELECT … FINAL` или `OPTIMIZE TABLE … FINAL` |
| Неопределённый результат в `ReplacingMergeTree` без версии | Добавление версии через поле `Version`                     |
| Неверная агрегация при вставке в `AggregatingMergeTree`    | Использовать `uniqState()` и `uniqMerge()`                 |

---

## 🔗 Ссылки на материалы

- [Обзор MergeTree-движков в документации ClickHouse](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/)
- [ReplacingMergeTree: дедупликация строк](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/replacingmergetree)
- [SummingMergeTree: автоматическое суммирование](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/summingmergetree)
- [AggregatingMergeTree и агрегатные функции](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/aggregatingmergetree)
- [CollapsingMergeTree: логическое удаление](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/collapsingmergetree)
- [FINAL модификатор: поведение и производительность](https://clickhouse.com/docs/en/sql-reference/statements/select/final)

---

## 🧾 Вывод

- **MergeTree** — универсальный базовый движок.
- **ReplacingMergeTree** — подходит для замены и дедупликации строк.
- **SummingMergeTree** — агрегирует числовые поля автоматически.
- **AggregatingMergeTree** — для агрегатных состояний (с `uniqState()` и `uniqMerge()`).
- **CollapsingMergeTree** — симулирует `DELETE` через схлопывание.

---
````
