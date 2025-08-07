---
# Подробное руководство по мониторингу и оптимизации ClickHouse в Docker
---

## 1. Структура проекта и зачем она нужна

### Что мы делаем?

Создаём папки для удобной организации проекта:

- `config/clickhouse-backup/` — конфигурация для утилиты резервного копирования ClickHouse.
- `backups/` — здесь будут храниться ваши резервные копии данных.
- `scripts/` — место для полезных скриптов (например, для автоматизации задач).
- `config/prometheus/` — конфигурация для системы мониторинга Prometheus.

### Как создать?

Откройте терминал, перейдите в папку, где хотите хранить проект, и выполните:

```bash
mkdir -p docker/{config/clickhouse-backup,config/prometheus,scripts,backups}
cd docker
```

---

## 2. Docker Compose — сервисы и зачем они нужны

### Что такое Docker Compose?

Это файл, который описывает ваши контейнеры (сервисы) и как они связаны. Вместо запуска каждого вручную, достаточно одной команды.

### Какие сервисы у нас будут?

- **MinIO** — простой объектный S3-совместимый сторедж для хранения резервных копий.
- **ClickHouse** — ваша основная аналитическая база.
- **clickhouse-backup** — утилита для создания и восстановления резервных копий ClickHouse.
- **Prometheus** — система мониторинга, которая будет собирать метрики с ClickHouse.

### Как написать файл?

Создайте файл `docker-compose.yml` в папке `docker` с таким содержимым (текст длинный — посмотрите полный файл чуть ниже):

```yaml
version: '3.8'

services:
  minio:
    image: minio/minio
    ports:
      - '19000:9000' # S3 API
      - '9001:9001' # Web UI
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server /data --console-address ":9001"
    volumes:
      - minio_data:/data

  clickhouse:
    image: clickhouse/clickhouse-server:23.3-alpine
    ports:
      - '8123:8123'
      - '9000:9000'
      - '9363:9363' # порт для Prometheus метрик
    volumes:
      - clickhouse_data:/var/lib/clickhouse
      - ./config/clickhouse/config.xml:/etc/clickhouse-server/config.xml # если нужно добавить конфиг с метриками
    environment:
      CLICKHOUSE_DB: default
      CLICKHOUSE_USER: default
      CLICKHOUSE_PASSWORD: ''
    ulimits:
      nofile:
        soft: 262144
        hard: 262144

  clickhouse-backup:
    image: alexakulov/clickhouse-backup:latest
    restart: unless-stopped
    depends_on:
      clickhouse:
        condition: service_started
    volumes:
      - ./config/clickhouse-backup/config.yml:/etc/clickhouse-backup/config.yml
      - ./backups:/var/lib/clickhouse/backup
    command: ['sleep', 'infinity']

  prometheus:
    image: prom/prometheus
    ports:
      - '9090:9090'
    volumes:
      - ./config/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml

volumes:
  minio_data:
  clickhouse_data:
```

### Что здесь происходит?

- Для MinIO — мы пробрасываем порты и задаём логин/пароль.
- ClickHouse запускается с нужными портами, включая 9363 — это важно для метрик.
- Clickhouse-backup подключён к ClickHouse, чтобы управлять бэкапами.
- Prometheus будет опрашивать ClickHouse на метрики.
- Созданы отдельные тома для сохранения данных MinIO и ClickHouse.

---

## 3. Включаем метрики Prometheus в ClickHouse

### Почему это важно?

ClickHouse умеет экспортировать статистику и показатели своей работы (например, сколько памяти используется, сколько запросов в очереди и т.п.). Prometheus будет периодически забирать эти данные.

### Где настраивается?

В конфигурационном файле ClickHouse `config.xml`.

### Что сделать?

- Если у вас его нет, создайте файл `docker/config/clickhouse/config.xml` (отредактируйте существующий, если есть).
- В секции `<yandex>` добавьте:

```xml
<prometheus>
    <endpoint>/metrics</endpoint>         <!-- URL для метрик -->
    <port>9363</port>                     <!-- Порт, который мы пробросили -->
    <metrics>true</metrics>               <!-- Включаем сбор метрик -->
    <events>true</events>                 <!-- Включаем события -->
    <asynchronous_metrics>true</asynchronous_metrics> <!-- Асинхронные метрики -->
</prometheus>
```

### После изменений

Перезапустите ClickHouse:

```bash
docker-compose restart clickhouse
```

---

## 4. Настраиваем Prometheus для сбора метрик ClickHouse

### Что такое Prometheus?

Это система мониторинга, которая регулярно опрашивает сервисы и собирает данные о состоянии.

### Как настроить?

Создайте файл конфигурации `docker/config/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s # Как часто собирать метрики

scrape_configs:
  - job_name: 'clickhouse' # Имя задачи
    static_configs:
      - targets: ['clickhouse:9363'] # Где брать метрики
```

Здесь мы говорим Prometheus, что нужно забирать метрики каждые 15 секунд с адреса `clickhouse:9363` (имя контейнера и порт).

---

## 5. Создаём базу и таблицу для персонализированного мониторинга в ClickHouse

### Зачем?

Вместо постоянного копирования запросов, удобно хранить типовые запросы мониторинга в отдельной таблице.

### Как создать?

Выполните в ClickHouse (через контейнер):

```bash
docker exec -it docker-clickhouse-1 clickhouse-client --query="
CREATE DATABASE IF NOT EXISTS monitoring;

CREATE TABLE IF NOT EXISTS monitoring.queries (
    query_name String,
    query_text String,
    update_interval UInt32,
    last_executed DateTime DEFAULT now()
) ENGINE = MergeTree()
ORDER BY query_name;
"
```

### Что это значит?

- `monitoring` — база данных для мониторинга.
- Таблица `queries` — в ней храним запросы, их описание и частоту обновления.

---

## 6. Добавляем типовые запросы мониторинга

### Почему?

Это примеры запросов, которые помогут следить за состоянием сервера.

### Добавляем:

```bash
docker exec -it docker-clickhouse-1 clickhouse-client --query="
INSERT INTO monitoring.queries (query_name, query_text, update_interval)
VALUES
('Active Queries', 'SELECT count() FROM system.processes', 5),
('Memory Usage', 'SELECT formatReadableSize(sum(memory_usage)) FROM system.processes', 10),
('Slow Queries', 'SELECT query, elapsed FROM system.query_log WHERE elapsed > 1 ORDER BY elapsed DESC LIMIT 5', 60),
('Replication Status', 'SELECT table, absolute_delay FROM system.replicas WHERE is_readonly', 30);
"
```

---

## 7. Оптимизация параметров ClickHouse

### Почему?

Чтобы ClickHouse не использовал слишком много памяти и не запускал слишком много параллельных запросов.

### Пример установки:

```bash
docker exec -it docker-clickhouse-1 clickhouse-client --query="
SET max_memory_usage = 10000000000;  -- 10 ГБ памяти
SET max_concurrent_queries = 100;    -- максимум 100 параллельных запросов
"
```

_Можно вынести в конфигурацию, чтобы применялось по умолчанию._

---

## 8. Проверка использования ресурсов вручную

### Для диагностики:

```bash
docker exec -it docker-clickhouse-1 clickhouse-client --query="
SELECT metric, value
FROM system.asynchronous_metrics
WHERE metric LIKE '%Memory%' OR metric LIKE '%CPU%';
"
```

---

## 9. Запуск всей системы

- Запустите все сервисы:

```bash
docker-compose up -d
```

- Проверьте метрики ClickHouse (можно из браузера или терминала):

```bash
curl http://localhost:9363/metrics
```

- Откройте Prometheus в браузере: `http://localhost:9090`

- Посмотрите запросы мониторинга:

```bash
docker exec -it docker-clickhouse-1 clickhouse-client --query="SELECT * FROM monitoring.queries;"
```

---

## 10. Что делать дальше?

- Можно создать Grafana и подключить её к Prometheus для красивых дашбордов.
- Написать скрипты для автоматического обновления мониторинговых запросов.
- Настроить алерты в Prometheus.

---

Если хочешь, могу сделать подробное описание для настройки Grafana или помощь с автоматизацией!

---

Если нужно, могу оформить этот текст в PDF/Markdown файл — чтобы было удобно сохранить.
