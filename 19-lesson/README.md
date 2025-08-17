# Руководство по развертыванию Apache Superset с подключением к ClickHouse

## Содержание

- [Руководство по развертыванию Apache Superset с подключением к ClickHouse](#руководство-по-развертыванию-apache-superset-с-подключением-к-clickhouse)
	- [Содержание](#содержание)
	- [Предварительные требования](#предварительные-требования)
	- [Установка Docker](#установка-docker)
	- [Настройка конфигурации](#настройка-конфигурации)
	- [Запуск и инициализация](#запуск-и-инициализация)
	- [Подключение ClickHouse](#подключение-clickhouse)
	- [Создание визуализаций](#создание-визуализаций)
		- [Line Chart](#line-chart)
		- [Bar Chart](#bar-chart)
	- [Создание дашбордов](#создание-дашбордов)
	- [Полезные команды](#полезные-команды)

---

## Предварительные требования

- Установленный Docker и Docker-compose
- Доступ к серверу ClickHouse (хост, порт, учетные данные)
- 4+ GB оперативной памяти

## Установка Docker

```bash
# Для Ubuntu/Debian
sudo apt-get update
sudo apt-get install docker.io docker-compose
sudo usermod -aG docker $USER
```

## Настройка конфигурации

Создайте `docker-compose.yml`:

```yaml
version: '3.8'
services:
  superset:
    image: apache/superset
    environment:
      SUPERSET_SECRET_KEY: your-secret-key
    ports:
      - '8088:8088'
    volumes:
      - superset_data:/app/superset_home

  redis:
    image: redis:latest

  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: superset
      POSTGRES_PASSWORD: superset
      POSTGRES_DB: superset

volumes:
  superset_data:
```

## Запуск и инициализация

```bash
docker-compose up -d
docker-compose exec superset superset db upgrade
docker-compose exec superset superset init
docker-compose exec superset superset fab create-admin \
  --username admin \
  --password admin
```

## Подключение ClickHouse

1. Установите драйвер:

```bash
docker-compose exec superset pip install clickhouse-sqlalchemy==0.2.4
```

2. В интерфейсе Superset:
   - **Database Name**: ClickHouse Production
   - **SQLAlchemy URI**:
     ```
     clickhouse://username:password@clickhouse-host:8123/default
     ```

## Создание визуализаций

### Line Chart

```sql
SELECT toDate(timestamp) AS day, COUNT(*)
FROM your_table
GROUP BY day
```

### Bar Chart

```sql
SELECT category, SUM(value)
FROM your_table
GROUP BY category
```

## Создание дашбордов

1. Создайте новый дашборд
2. Добавьте чарты через "Add Components"
3. Настройте фильтры и расположение

## Полезные команды

```bash
# Проверка логов
docker-compose logs superset

# Пересоздание контейнеров
docker-compose down && docker-compose up -d

# Доступ к CLI Superset
docker-compose exec superset superset shell
```

---
