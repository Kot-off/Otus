#Vagrant-CH

## Структура проекта

```
project/
├── Vagrantfile              # Конфигурация Vagrant для создания VM
├── provisioning/
│   ├── playbook.yml         # Ansible playbook для настройки
│   ├── roles/
│   │   ├── common/          # Общие настройки (пользователи, пакеты)
│   │   ├── docker/          # Установка Docker и Docker Compose
│   │   └── clickhouse/      # Настройка ClickHouse и Grafana
├── docker-compose.yml       # Docker Compose для ClickHouse и Grafana
└── README.md                # Инструкции по развертыванию
```

# Otus
