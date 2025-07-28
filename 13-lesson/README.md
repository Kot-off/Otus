# ClickHouse: Storage Policy и резервное копирование

## Содержание

- [ClickHouse: Storage Policy и резервное копирование](#clickhouse-storage-policy-и-резервное-копирование)
	- [Содержание](#содержание)
	- [Настройка окружения ](#настройка-окружения-)
		- [Развертывание MinIO ](#развертывание-minio-)
		- [Установка ClickHouse ](#установка-clickhouse-)
		- [Установка clickhouse-backup ](#установка-clickhouse-backup-)
	- [Конфигурация ](#конфигурация-)
		- [Настройка clickhouse-backup ](#настройка-clickhouse-backup-)
		- [Политики хранения ClickHouse ](#политики-хранения-clickhouse-)
	- [Тестирование ](#тестирование-)
		- [Создание тестовых данных ](#создание-тестовых-данных-)
		- [Резервное копирование ](#резервное-копирование-)
		- [Повреждение данных ](#повреждение-данных-)
		- [Восстановление ](#восстановление-)
		- [](#)

---

## Настройка окружения <a name="настройка-окружения"></a>

### Развертывание MinIO <a name="развертывание-minio"></a>

```bash
docker run -p 9000:9000 -p 9001:9001 \
  -e "MINIO_ROOT_USER=clickhouse" \
  -e "MINIO_ROOT_PASSWORD=clickhouse123" \
  minio/minio server /data --console-address ":9001"
```

### Установка ClickHouse <a name="установка-clickhouse"></a>

```bash
sudo apt-get install -y apt-transport-https ca-certificates dirmngr
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv E0C56BD4
echo "deb https://packages.clickhouse.com/deb stable main" | sudo tee /etc/apt/sources.list.d/clickhouse.list
sudo apt-get update
sudo apt-get install -y clickhouse-server clickhouse-client
```

### Установка clickhouse-backup <a name="установка-clickhouse-backup"></a>

```bash
wget https://github.com/AlexAkulov/clickhouse-backup/releases/download/v2.4.0/clickhouse-backup.tar.gz
tar -xvf clickhouse-backup.tar.gz
sudo mv clickhouse-backup /usr/local/bin/
```

## Конфигурация <a name="конфигурация"></a>

### Настройка clickhouse-backup <a name="настройка-clickhouse-backup"></a>

```yaml
# /etc/clickhouse-backup/config.yml
s3:
  access_key: clickhouse
  secret_key: clickhouse123
  bucket: clickhouse-backups
  endpoint: http://localhost:9000
```

### Политики хранения ClickHouse <a name="политики-хранения-clickhouse"></a>

```xml
<!-- /etc/clickhouse-server/config.d/storage.xml -->
<yandex>
  <storage_configuration>
    <disks>
      <s3_disk>
        <type>s3</type>
        <endpoint>http://localhost:9000/clickhouse-backups/</endpoint>
      </s3_disk>
    </disks>
  </storage_configuration>
</yandex>
```

## Тестирование <a name="тестирование"></a>

### Создание тестовых данных <a name="создание-тестовых-данных"></a>

```sql
CREATE DATABASE test_backup;
CREATE TABLE test_backup.users (id UInt32) ENGINE = MergeTree();
INSERT INTO test_backup.users VALUES (1), (2), (3);
```

### Резервное копирование <a name="резервное-копирование"></a>

```bash
clickhouse-backup create test_backup
clickhouse-backup upload test_backup
```

### Повреждение данных <a name="повреждение-данных"></a>

```sql
DROP TABLE test_backup.users;
```

### Восстановление <a name="восстановление"></a>

```bash
clickhouse-backup restore test_backup
```

###

```

```
