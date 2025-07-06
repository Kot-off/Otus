# Шардирование в ClickHouse

## Содержание

- [Шардирование в ClickHouse](#шардирование-в-clickhouse)
	- [Содержание](#содержание)
	- [Обзор проекта](#обзор-проекта)
	- [Топологии кластеров](#топологии-кластеров)
		- [Топология 1: 2 шарда с репликацией](#топология-1-2-шарда-с-репликацией)
		- [Топология 2: 3 шарда без репликации](#топология-2-3-шарда-без-репликации)
	- [Distributed-таблицы](#distributed-таблицы)
		- [Для топологии 1 (2 шарда × 2 реплики):](#для-топологии-1-2-шарда--2-реплики)
		- [Для топологии 2 (3 шарда × 1 реплика):](#для-топологии-2-3-шарда--1-реплика)
	- [Проверка работы](#проверка-работы)
		- [1. Просмотр информации о кластерах:](#1-просмотр-информации-о-кластерах)
		- [2. Проверка Distributed-таблиц:](#2-проверка-distributed-таблиц)
		- [3. Просмотр структуры таблиц:](#3-просмотр-структуры-таблиц)
	- [Конфигурационные файлы](#конфигурационные-файлы)

## Обзор проекта

Проект реализует:

1. Две различные топологии шардирования
2. Distributed-таблицы для каждой топологии
3. Проверку работы шардирования

## Топологии кластеров

### Топология 1: 2 шарда с репликацией

**Параметры:**

- 2 шарда
- Фактор репликации: 2
- Всего 4 сервера (2 шарда × 2 реплики)

```xml
<!-- config.xml -->
<remote_servers>
    <cluster_2s2r>
        <shard>
            <replica>
                <host>server1</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>server2</host>
                <port>9000</port>
            </replica>
        </shard>
        <shard>
            <replica>
                <host>server3</host>
                <port>9000</port>
            </replica>
            <replica>
                <host>server4</host>
                <port>9000</port>
            </replica>
        </shard>
    </cluster_2s2r>
</remote_servers>
```

### Топология 2: 3 шарда без репликации

**Параметры:**

- 3 шарда
- Без репликации
- Всего 3 сервера

```xml
<!-- config.xml -->
<remote_servers>
    <cluster_3s1r>
        <shard>
            <replica>
                <host>server5</host>
                <port>9000</port>
            </replica>
        </shard>
        <shard>
            <replica>
                <host>server6</host>
                <port>9000</port>
            </replica>
        </shard>
        <shard>
            <replica>
                <host>server7</host>
                <port>9000</port>
            </replica>
        </shard>
    </cluster_3s1r>
</remote_servers>
```

## Distributed-таблицы

### Для топологии 1 (2 шарда × 2 реплики):

```sql
CREATE TABLE distributed_2s2r AS system.one
ENGINE = Distributed('cluster_2s2r', 'system', 'one');
```

### Для топологии 2 (3 шарда × 1 реплика):

```sql
CREATE TABLE distributed_3s1r AS system.one
ENGINE = Distributed('cluster_3s1r', 'system', 'one');
```

## Проверка работы

### 1. Просмотр информации о кластерах:

```sql
SELECT * FROM system.clusters;
```

### 2. Проверка Distributed-таблиц:

```sql
-- Для топологии 1
SELECT *, hostName(), _shard_num
FROM distributed_2s2r
LIMIT 5;

-- Для топологии 2
SELECT *, hostName(), _shard_num
FROM distributed_3s1r
LIMIT 5;
```

### 3. Просмотр структуры таблиц:

```sql
SHOW CREATE TABLE distributed_2s2r;
SHOW CREATE TABLE distributed_3s1r;
```

## Конфигурационные файлы

- config.xml - основная конфигурация кластеров
- users.xml - настройки пользователей (при необходимости)
