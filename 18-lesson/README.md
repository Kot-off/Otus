# Интеграция Apache Kafka с ClickHouse

---

## 📑 Содержание

- [Интеграция Apache Kafka с ClickHouse](#интеграция-apache-kafka-с-clickhouse)
	- [📑 Содержание](#-содержание)
	- [1. Что потребуется от ClickHouse](#1-что-потребуется-от-clickhouse)
	- [2. Добавление Apache Kafka в docker-compose](#2-добавление-apache-kafka-в-docker-compose)
	- [3. Настройка ClickHouse для работы с Kafka](#3-настройка-clickhouse-для-работы-с-kafka)
	- [4. Тестирование интеграции](#4-тестирование-интеграции)
	- [5. Дополнительные настройки](#5-дополнительные-настройки)
	- [6. Проверка корректности работы](#6-проверка-корректности-работы)
	- [7. Полезные команды для мониторинга](#7-полезные-команды-для-мониторинга)

---

## 1. Что потребуется от ClickHouse

- ClickHouse уже должен быть в вашем `docker-compose.yml` и работать.
- Необходимо, чтобы он был доступен внутри сети Docker по имени контейнера `clickhouse` (или изменить в командах ниже).
- Порты и конфигурация ClickHouse остаются прежними.

---

## 2. Добавление Apache Kafka в docker-compose

Добавьте в `docker-compose.yml` сервисы **Zookeeper** и **Kafka**. Остальные ваши сервисы (ClickHouse, MinIO и т. д.) оставьте без изменений.

```yaml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.3.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-kafka:7.3.0
    depends_on:
      - zookeeper
    ports:
      - '9092:9092'
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'
```

---

## 3. Настройка ClickHouse для работы с Kafka

Подключитесь к ClickHouse:

```bash
docker-compose exec clickhouse clickhouse-client
```

Создайте таблицы и представление:

1. **MergeTree-таблица для хранения данных:**

```sql
CREATE TABLE test_data (
    id UInt32,
    message String,
    timestamp DateTime
) ENGINE = MergeTree()
ORDER BY (id, timestamp);
```

2. **Kafka Engine-таблица для чтения данных:**

```sql
CREATE TABLE kafka_test (
    id UInt32,
    message String,
    timestamp DateTime
) ENGINE = Kafka()
SETTINGS
    kafka_broker_list = 'kafka:29092',
    kafka_topic_list = 'test_topic',
    kafka_group_name = 'clickhouse_consumer_group',
    kafka_format = 'JSONEachRow',
    kafka_row_delimiter = '\n',
    kafka_num_consumers = 1;
```

3. **Материализованное представление для записи в MergeTree:**

```sql
CREATE MATERIALIZED VIEW kafka_to_merge_tree TO test_data AS
SELECT id, message, timestamp
FROM kafka_test;
```

---

## 4. Тестирование интеграции

1. Отправьте тестовые сообщения в Kafka:

```bash
docker-compose exec kafka kafka-console-producer \
  --broker-list localhost:9092 \
  --topic test_topic
```

Вставьте строки в формате JSON:

```json
{"id":1,"message":"Hello ClickHouse","timestamp":"2023-06-01 12:00:00"}
{"id":2,"message":"Another message","timestamp":"2023-06-01 12:01:00"}
```

Нажмите `Ctrl+D` для завершения ввода.

2. Проверьте, что данные появились в ClickHouse:

```sql
SELECT * FROM test_data;
```

---

## 5. Дополнительные настройки

- **Параллельное потребление:**
  увеличить `kafka_num_consumers` для повышения производительности.
- **Пропуск поврежденных сообщений:**
  добавить `kafka_skip_broken_messages = 1`.
- **Несколько топиков:**
  перечислить их в `kafka_topic_list` через запятую.

---

## 6. Проверка корректности работы

1. Убедитесь, что данные из Kafka появляются в MergeTree-таблице.
2. Проверьте, что новые сообщения также обрабатываются.
3. Убедитесь, что при перезапуске потребителя нет дублирования.

---

## 7. Полезные команды для мониторинга

Проверка потребителей Kafka:

```sql
SELECT * FROM system.kafka_consumers;
```

Просмотр ошибок обработки:

```sql
SELECT * FROM system.errors WHERE name LIKE '%Kafka%';
```

---

✅ Теперь у нас настроен полный пайплайн:
Kafka → ClickHouse Kafka Engine → Materialized View → MergeTree.

```
Kafka Producer → Kafka Broker → Kafka Engine (CH) → Materialized View → MergeTree
```
