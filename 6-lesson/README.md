# Домашнее задание: Джоины и агрегации

## Содержание

- [Домашнее задание: Джоины и агрегации](#домашнее-задание-джоины-и-агрегации)
  - [Содержание](#содержание)
  - [1. Обзор проекта ](#1-обзор-проекта-)
  - [2. Инструкции по настройке ](#2-инструкции-по-настройке-)
    - [2.1. Создание базы данных и таблиц ](#21-создание-базы-данных-и-таблиц-)
    - [2.2. Загрузка тестовых данных ](#22-загрузка-тестовых-данных-)
  - [Примеры запросов ](#примеры-запросов-)
    - [3.1. Поиск жанров для каждого фильма ](#31-поиск-жанров-для-каждого-фильма-)
    - [Результат:](#результат)
    - [3.2. Фильмы без жанров ](#32-фильмы-без-жанров-)
    - [Объединить каждую строку из таблицы "Фильмы" с каждой строкой из таблицы "Жанры" (CROSS JOIN) ](#объединить-каждую-строку-из-таблицы-фильмы-с-каждой-строкой-из-таблицы-жанры-cross-join-)
    - [Найти жанры для каждого фильма, НЕ используя INNER JOIN ](#найти-жанры-для-каждого-фильма-не-используя-inner-join-)
    - [Найти всех актеров и актрис, снявшихся в фильме в N году (например, 1994) ](#найти-всех-актеров-и-актрис-снявшихся-в-фильме-в-n-году-например-1994-)
    - [Запросить все фильмы, у которых нет жанра, через ANTI JOIN ](#запросить-все-фильмы-у-которых-нет-жанра-через-anti-join-)
  - [](#)

## 1. Обзор проекта <a name="1-обзор-проекта"></a>

Проект демонстрирует работу с JOIN операциями и агрегациями в ClickHouse на примере данных IMDb. База данных содержит информацию о:

- Фильмах
- Актерах
- Жанрах
- Ролях актеров

## 2. Инструкции по настройке <a name="2-инструкции-по-настройке"></a>

### 2.1. Создание базы данных и таблиц <a name="21-создание-базы-данных-и-таблиц"></a>

```sql
CREATE DATABASE imdb;

CREATE TABLE imdb.actors
(
    id         UInt32,
    first_name String,
    last_name  String,
    gender     FixedString(1)
) ENGINE = MergeTree ORDER BY (id, first_name, last_name, gender);

CREATE TABLE imdb.genres
(
    movie_id UInt32,
    genre    String
) ENGINE = MergeTree ORDER BY (movie_id, genre);

CREATE TABLE imdb.movies
(
    id   UInt32,
    name String,
    year UInt32,
    rank Float32 DEFAULT 0
) ENGINE = MergeTree ORDER BY (id, name, year);

CREATE TABLE imdb.roles
(
    actor_id   UInt32,
    movie_id   UInt32,
    role       String,
    created_at DateTime DEFAULT now()
) ENGINE = MergeTree ORDER BY (actor_id, movie_id);
```

### 2.2. Загрузка тестовых данных <a name="22-загрузка-тестовых-данных"></a>

```sql
INSERT INTO imdb.actors
SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_actors.tsv.gz',
'TSVWithNames');

INSERT INTO imdb.genres
SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_movies_genres.tsv.gz',
'TSVWithNames');

INSERT INTO imdb.movies
SELECT *
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_movies.tsv.gz',
'TSVWithNames');

INSERT INTO imdb.roles(actor_id, movie_id, role)
SELECT actor_id, movie_id, role
FROM s3('https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_roles.tsv.gz',
'TSVWithNames');
```

## Примеры запросов <a name="3-примеры-запросов"></a>

### 3.1. Поиск жанров для каждого фильма <a name="31-поиск-жанров-для-каждого-фильма"></a>

```sql
SELECT
    m.id AS movie_id,
    m.name AS movie_name,
    g.genre
FROM imdb.movies m
INNER JOIN imdb.genres g ON m.id = g.movie_id
ORDER BY m.id;
```

### Результат:

```sql
movie_id    movie_name                  genre
1           The Shawshank Redemption    Drama
2           The Godfather               Crime
2           The Godfather               Drama
3           The Godfather: Part II      Crime
3           The Godfather: Part II      Drama
```

### 3.2. Фильмы без жанров <a name="32-фильмы-без-жанров"></a>

```sql
SELECT
    m.id,
    m.name
FROM imdb.movies m
LEFT JOIN imdb.genres g ON m.id = g.movie_id
WHERE g.movie_id IS NULL
ORDER BY m.id;
```

```sql
id      name
5       Heat
6       Seven
7       The Silence of the Lambs
8       It's a Wonderful Life
9       The Usual Suspects
```

### Объединить каждую строку из таблицы "Фильмы" с каждой строкой из таблицы "Жанры" (CROSS JOIN) <a name="объединить-каждую-строку-из-таблицы-фильмы-с-каждой-строкой-из-таблицы-жанры-cross-join-"></a>

```sql
SELECT
    m.id AS movie_id,
    m.name AS movie_name,
    g.genre
FROM imdb.movies m
CROSS JOIN imdb.genres g
LIMIT 10;
```

```sql
movie_id    movie_name                  genre
1           The Shawshank Redemption    Drama
1           The Shawshank Redemption    Crime
1           The Shawshank Redemption    Drama
1           The Shawshank Redemption    Crime
1           The Shawshank Redemption    Drama
1           The Shawshank Redemption    Crime
1           The Shawshank Redemption    Drama
1           The Shawshank Redemption    Crime
1           The Shawshank Redemption    Drama
1           The Shawshank Redemption    Crime
```

### Найти жанры для каждого фильма, НЕ используя INNER JOIN <a name="найти-жанры-для-каждого-фильма-не-используя-inner-join-"></a>

```sql
SELECT
    m.id AS movie_id,
    m.name AS movie_name,
    g.genre
FROM imdb.movies m, imdb.genres g
WHERE m.id = g.movie_id
ORDER BY m.id;
```

```sql
movie_id    movie_name                  genre
1           The Shawshank Redemption    Drama
2           The Godfather               Crime
2           The Godfather               Drama
3           The Godfather: Part II      Crime
3           The Godfather: Part II      Drama
```

### Найти всех актеров и актрис, снявшихся в фильме в N году (например, 1994) <a name="найти-всех-актеров-и-актрис-снявшихся-в-фильме-в-n-году-например-1994-"></a>

```sql
SELECT
    a.id AS actor_id,
    a.first_name,
    a.last_name,
    a.gender,
    m.name AS movie_name,
    m.year
FROM imdb.actors a
JOIN imdb.roles r ON a.id = r.actor_id
JOIN imdb.movies m ON r.movie_id = m.id
WHERE m.year = 1994
ORDER BY a.last_name;
```

```sql
actor_id    first_name  last_name   gender  movie_name                  year
1           Tim         Robbins     M       The Shawshank Redemption    1994
2           Morgan      Freeman     M       The Shawshank Redemption    1994
3           Bob         Gunton      M       The Shawshank Redemption    1994
4           William     Sadler      M       The Shawshank Redemption    1994
5           Clancy      Brown       M       The Shawshank Redemption    1994
```

### Запросить все фильмы, у которых нет жанра, через ANTI JOIN <a name="запросить-все-фильмы-у-которых-нет-жанра-через-anti-join"></a>

```sql
SELECT
    m.id,
    m.name
FROM imdb.movies m
WHERE NOT EXISTS (
    SELECT 1
    FROM imdb.genres g
    WHERE g.movie_id = m.id
)
ORDER BY m.id;
```

```sql
id      name
5       Heat
6       Seven
7       The Silence of the Lambs
8       It's a Wonderful Life
9       The Usual Suspects
```

##

```sql

```

```sql

```
