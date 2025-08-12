# Загрузка данных в ClickHouse через Airflow

## Содержание

- [Загрузка данных в ClickHouse через Airflow](#загрузка-данных-в-clickhouse-через-airflow)
  - [Содержание](#содержание)
  - [Цель](#цель)
  - [Что уже есть](#что-уже-есть)
    - [1. Запуск инфраструктуры](#1-запуск-инфраструктуры)
    - [2. Подготовка DAG в Airflow](#2-подготовка-dag-в-airflow)
    - [3. Проверка работы](#3-проверка-работы)
  - [SQL для проверки данных](#sql-для-проверки-данных)
  - [Файл `docker-compose.yml`](#файл-docker-composeyml)

---

## Цель

- Настроить инструмент ETL для загрузки данных в ClickHouse.
- Создать пайплайн для одноразовой или регулярной загрузки данных из внешнего источника.

---

## Что уже есть

- **ClickHouse** (контейнер `clickhouse` в `docker-compose`) — готов для подключения.
- **Airflow** — развернут в контейнере `airflow` и готов к запуску DAG'ов.
- **PostgreSQL** — используется Airflow как база метаданных.
- **MinIO** и **clickhouse-backup** — для хранения и резервного копирования (по заданию напрямую не используются).

---

### 1. Запуск инфраструктуры

Запустите все сервисы:

```bash
docker-compose up -d
```

Убедитесь, что запущены контейнеры:

```bash
docker ps
```

---

### 2. Подготовка DAG в Airflow

Создайте файл `Airflow/dags/clickhouse_etl.py` со следующим содержимым:

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python_operator import PythonOperator
from clickhouse_driver import Client

default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'start_date': datetime(2023, 1, 1),
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

def load_data_to_clickhouse():
    ch_client = Client(
        host='clickhouse',
        user='default',
        password='',
        database='default'
    )

    # Создание таблицы
    ch_client.execute('''
    CREATE TABLE IF NOT EXISTS users (
        id Int32,
        name String,
        age Int32,
        timestamp DateTime
    ) ENGINE = MergeTree()
    ORDER BY id
    ''')

    # Пример тестовых данных
    test_data = [
        {'id': 1, 'name': 'Alice', 'age': 25, 'timestamp': datetime.now()},
        {'id': 2, 'name': 'Bob', 'age': 30, 'timestamp': datetime.now()},
        {'id': 3, 'name': 'Charlie', 'age': 35, 'timestamp': datetime.now()}
    ]

    # Вставка данных
    ch_client.execute('INSERT INTO users VALUES', test_data)

with DAG(
    'clickhouse_etl',
    default_args=default_args,
    schedule_interval='@daily',
    catchup=False,
) as dag:

    load_task = PythonOperator(
        task_id='load_data_to_clickhouse',
        python_callable=load_data_to_clickhouse,
    )

    load_task
```

> **Важно:** Убедитесь, что в контейнере `airflow` установлен пакет `clickhouse-driver`:

```bash
docker exec -it airflow python -m pip install clickhouse-driver
```

---

### 3. Проверка работы

1. Перейдите в веб-интерфейс Airflow: [http://localhost:8080](http://localhost:8080)
2. Найдите DAG `clickhouse_etl` и включите его (ползунок слева).
3. Нажмите **"Trigger DAG"** для запуска вручную.
4. Откройте логи задачи, чтобы убедиться, что данные загружены.

---

## SQL для проверки данных

Подключитесь к ClickHouse и выполните:

```sql
SELECT * FROM default.users;
```

---

## Файл `docker-compose.yml`

```yaml
version: '3.8'

services:
  minio:
    image: minio/minio
    ports:
      - '19000:9000'
      - '9001:9001'
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
    volumes:
      - clickhouse_data:/var/lib/clickhouse
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

  postgres:
    image: postgres:13
    environment:
      - POSTGRES_USER=airflow
      - POSTGRES_PASSWORD=airflow
      - POSTGRES_DB=airflow
    volumes:
      - postgres_data:/var/lib/postgresql/data

  airflow:
    image: apache/airflow:2.6.1
    depends_on:
      - clickhouse
      - postgres
    environment:
      - AIRFLOW__CORE__EXECUTOR=LocalExecutor
      - AIRFLOW__DATABASE__SQL_ALCHEMY_CONN=postgresql+psycopg2://airflow:airflow@postgres/airflow
      - AIRFLOW__CORE__LOAD_EXAMPLES=False
    volumes:
      - ./Airflow/dags:/opt/airflow/dags
      - ./Airflow/logs:/opt/airflow/logs
      - ./Airflow/plugins:/opt/airflow/plugins
    ports:
      - '8080:8080'
    command: airflow standalone

volumes:
  minio_data:
  clickhouse_data:
  postgres_data:
```

```

```
