---
# ✅ Полное руководство: ClickHouse + MinIO + clickhouse-backup

Пошаговая инструкция по развёртыванию системы резервного копирования ClickHouse с использованием MinIO и [clickhouse-backup](https://github.com/AlexAkulov/clickhouse-backup).

Подходит для локальной разработки, CI/CD и обучения.
---

## 🧰 1. Структура проекта

Создаём структуру:

```bash
mkdir -p docker/{config/clickhouse-backup,scripts,backups}
cd docker
```

- `config/clickhouse-backup/` — конфиг clickhouse-backup
- `backups/` — локальное хранилище бэкапов
- `scripts/` — вспомогательные скрипты

---

## ⚙️ 2. Docker Compose

Создаём `docker-compose.yml`:

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

## 📝 3. Конфигурация `clickhouse-backup`

Создаём `config/clickhouse-backup/config.yml`:

```yaml
general:
  remote_storage: s3
  backups_to_keep_remote: 2

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

## ▶️ 4. Запуск

```bash
docker-compose up -d
```

- ClickHouse будет на `http://localhost:8123`
- MinIO Web UI — `http://localhost:9001` (логин/пароль: `minioadmin`)

---

## 🌐 5. Настройка MinIO

```bash
docker exec -it docker-minio-1 mc alias set local http://minio:9000 minioadmin minioadmin
docker exec -it docker-minio-1 mc mb local/clickhouse-backups
docker exec -it docker-minio-1 mc anonymous set public local/clickhouse-backups
```

---

## 🧪 6. Тестовые данные

```bash
docker exec -it docker-clickhouse-1 clickhouse-client --query "CREATE DATABASE IF NOT EXISTS test"
docker exec -it docker-clickhouse-1 clickhouse-client --query "CREATE TABLE test.data (id Int32, name String) ENGINE = MergeTree() ORDER BY id"
docker exec -it docker-clickhouse-1 clickhouse-client --query "INSERT INTO test.data VALUES (1, 'Alice'), (2, 'Bob')"
```

---

## 💾 7. Создание резервной копии

```bash
# 1. Локальный бэкап (данные + схема)
docker exec -it docker-clickhouse-backup-1 clickhouse-backup create backup_$(date +%Y%m%d_%H%M%S)

# 2. Загрузка в MinIO
docker exec -it docker-clickhouse-backup-1 clickhouse-backup upload backup_*
```

> ⚠️ Никаких `--with-data` — бэкап с данными создаётся по умолчанию.

---

## 🧹 8. Удаление ClickHouse + volume

Чтобы эмулировать потерю данных и протестировать восстановление:

```bash
# Остановить и удалить контейнеры
docker-compose down

# Удалить volume с данными
docker volume rm docker_clickhouse_data

# Запустить ClickHouse заново
docker-compose up -d
sleep 30  # Подождать запуска
```

---

## ♻️ 9. Восстановление из MinIO

```bash
# 1. Скачиваем нужный бэкап из MinIO
docker exec -it docker-clickhouse-backup-1 clickhouse-backup download backup_YYYYMMDD_HHMMSS

# 2. Восстанавливаем
docker exec -it docker-clickhouse-backup-1 clickhouse-backup restore backup_YYYYMMDD_HHMMSS
```

🎯 После этого база и таблица полностью восстановятся вместе с данными.

---

## 🔍 10. Проверка

```bash
# Список локальных и удалённых бэкапов
docker exec -it docker-clickhouse-backup-1 clickhouse-backup list
docker exec -it docker-clickhouse-backup-1 clickhouse-backup list remote

# Проверка данных
docker exec -it docker-clickhouse-1 clickhouse-client --query "SELECT * FROM test.data"
```

---

## 🛠 11. Полезные адреса и команды

- **MinIO UI**: [http://localhost:9001](http://localhost:9001)
- **ClickHouse HTTP API**: [http://localhost:8123](http://localhost:8123)

### Логи:

```bash
docker logs docker-clickhouse-1
docker logs docker-clickhouse-backup-1
docker logs docker-minio-1
```

---
