# Контроль доступа в PostgreSQL

## Цель

Освоить базовые операции управления пользователями и ролями.

## Выполнение задания

### 1. Создание пользователя

```sql
CREATE USER anton WITH PASSWORD 'anton';
```

### 2. Создание роли

```sql
CREATE ROLE otus;
```

### 3. Выдача прав роли

```sql
GRANT SELECT ON TABLE your_table_name TO otus;
-- Замените your_table_name на имя конкретной таблицы
```

### 4. Назначение роли пользователю

```sql
GRANT otus TO anton;
```

## 5 Проверка созданных сущностей

### 5.1 Просмотр пользователей

```sql
SELECT * FROM pg_user WHERE usename = 'anton';
```

usename | usesysid | usecreatedb | usesuper | userepl | usebypassrls | passwd | valuntil | useconfig
---------+----------+-------------+----------+---------+--------------+----------+----------+-----------
anton | 16384 | f | f | f | f | **\*\*\*\*** | |

### 5.2 Просмотр ролей

```sql
SELECT * FROM pg_roles WHERE rolname = 'otus';
```

rolname | rolsuper | rolinherit | rolcreaterole | rolcreatedb | rolcanlogin | rolreplication | rolconnlimit | rolpassword | rolvaliduntil | rolbypassrls | rolconfig | oid  
---------+----------+------------+---------------+-------------+-------------+----------------+--------------+-------------+---------------+--------------+-----------+------
otus | f | t | f | f | f | f | -1 | **\*\*\*\*** | | f | | 16385

### 5.3 Просмотр привилегий

```sql
SELECT * FROM information_schema.role_table_grants
WHERE grantee = 'otus' AND table_name = 'your_table_name';
-- Замените your_table_name на имя таблицы, которой выдавали права
```

grantor | grantee | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy
---------+---------+---------------+--------------+------------+----------------+--------------+----------------
postgres| otus | my_database | public | employees | SELECT | NO | NO

### Проверка назначения ролей

```sql
SELECT * FROM pg_auth_members WHERE roleid = (SELECT oid FROM pg_roles WHERE rolname = 'otus');
```

roleid | member | grantor | admin_option
--------+--------+---------+--------------
16385 | 16384 | 10 | f

## Результаты

### Все операции выполнены успешно:

- Создан пользователь 'anton'
- Создана роль 'otus'
- Роли 'otus' выданы права SELECT на таблицу 'employees'
- Роль 'otus' назначена пользователю 'anton'
- Проверка через системные таблицы подтверждает создание всех сущностей и корректность назначений
