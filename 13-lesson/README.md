# 🚀 Полное руководство: ClickHouse + MinIO + clickhouse-backup

Это пошаговая инструкция по развёртыванию системы резервного копирования для ClickHouse с использованием MinIO в качестве хранилища и [clickhouse-backup](https://github.com/AlexAkulov/clickhouse-backup) — утилиты для создания и восстановления бэкапов.

Подходит для локальной разработки, тестирования, CI/CD и образовательных проектов.

---

## 🧰 1. Подготовка структуры проекта

Создаём структуру каталогов для конфигураций и хранения бэкапов.

```bash
mkdir -p docker/{config/clickhouse-backup,scripts,backups}
cd docker
```

- `config/clickhouse-backup/` — конфиг `clickhouse-backup`
- `backups/` — локальное хранилище резервных копий
- `scripts/` — можно положить вспомогательные bash-скрипты

---

## ⚙️ 2. Docker Compose: запуск всех сервисов

Создаём `docker-compose.yml`, который поднимет 3 сервиса:

- **ClickHouse** — аналитическая СУБД
- **MinIO** — объектное хранилище, совместимое с Amazon S3
- **clickhouse-backup** — утилита для создания, загрузки и восстановления бэкапов

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
      - '8123:8123' # HTTP API
      - '9000:9000' # Native protocol
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

volumes:
  minio_data:
  clickhouse_data:
```

---

## 📝 3. Конфигурация clickhouse-backup

Создаём файл `config/clickhouse-backup/config.yml`, где указываем, куда сохранять бэкапы и как подключаться к ClickHouse и MinIO.

```yaml
general:
  remote_storage: s3
  backups_to_keep_remote: 2 # храним максимум 2 бэкапа на S3

clickhouse:
  username: default
  password: ''
  host: clickhouse
  port: 9000

s3:
  access_key: 'minioadmin'
  secret_key: 'minioadmin'
  bucket: 'clickhouse-backups'
  endpoint: 'http://minio:9000'
  region: 'us-east-1'
  force_path_style: true
  disable_ssl: true
  part_size: 5242880
```

---

## ▶️ 4. Запуск всех сервисов

```bash
docker-compose up -d
```

- Все контейнеры запустятся в фоне.
- Через несколько секунд ClickHouse будет доступен на порту `8123`, MinIO — на `9001`.

---

## 🌐 5. Настройка хранилища MinIO

Теперь нужно создать бакет (аналог S3-папки) в MinIO, куда будут загружаться бэкапы.

```bash
# Настройка alias
docker exec -it docker-minio-1 mc alias set local http://minio:9000 minioadmin minioadmin

# Создание бакета
docker exec -it docker-minio-1 mc mb local/clickhouse-backups

# Делаем бакет публичным (опционально)
docker exec -it docker-minio-1 mc anonymous set public local/clickhouse-backups
```

---

## 🧪 6. Создание тестовой базы и таблицы

```bash
docker exec -it docker-clickhouse-1 clickhouse-client --query "CREATE DATABASE IF NOT EXISTS test"
docker exec -it docker-clickhouse-1 clickhouse-client --query "CREATE TABLE test.data (id Int32, name String) ENGINE = MergeTree() ORDER BY id"
docker exec -it docker-clickhouse-1 clickhouse-client --query "INSERT INTO test.data VALUES (1, 'Alice'), (2, 'Bob')"
```

- Создаётся база `test` и таблица `data`.
- Вставляются два тестовых значения.

---

## 💾 7. Создание резервной копии

```bash
# Создаём локальный бэкап с уникальным именем
docker exec -it docker-clickhouse-backup-1 clickhouse-backup create backup_$(date +%Y%m%d_%H%M%S)

# Загружаем бэкап в MinIO
docker exec -it docker-clickhouse-backup-1 clickhouse-backup upload backup_*
```

---

## 🔁 8. Восстановление данных из бэкапа

### ✅ Стандартное восстановление (из локального бэкапа):

```bash
docker exec -it docker-clickhouse-backup-1 clickhouse-backup restore backup_YYYYMMDD_HHMMSS
```

### ❗ Ошибка восстановления: UUID / "directory already exists"

Если таблица уже была удалена, но физические данные остались в volume:

1. Остановите ClickHouse:

   ```bash
   docker-compose stop clickhouse
   ```

2. Удалите данные:

   ```bash
   docker volume rm docker_clickhouse_data
   ```

3. Перезапустите ClickHouse:

   ```bash
   docker-compose up -d clickhouse
   sleep 30  # подождите запуска
   ```

4. Повторите восстановление.

### 🛠 Восстановление отдельно схемы и данных:

Иногда удобно восстановить только структуру таблиц (без данных), а затем отдельно подгрузить данные:

```bash
# Только структура таблиц
docker exec -it docker-clickhouse-backup-1 clickhouse-backup restore --schema backup_*

# Только данные
docker exec -it docker-clickhouse-backup-1 clickhouse-backup restore --data backup_*
```

---

## 🔍 9. Проверка и отладка

```bash
# Просмотр доступных бэкапов
docker exec -it docker-clickhouse-backup-1 clickhouse-backup list
docker exec -it docker-clickhouse-backup-1 clickhouse-backup list remote

# Проверка данных в таблице
docker exec -it docker-clickhouse-1 clickhouse-client --query "SELECT * FROM test.data"
```

---

## 🛠 10. Полезные адреса и команды

- **MinIO Web UI**: [http://localhost:9001](http://localhost:9001)
  Логин: `minioadmin`, Пароль: `minioadmin`

- **ClickHouse HTTP API**: [http://localhost:8123](http://localhost:8123)

### Логи контейнеров при ошибках:

```bash
docker logs docker-clickhouse-1
docker logs docker-minio-1
docker logs docker-clickhouse-backup-1
```

---
