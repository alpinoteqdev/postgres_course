# Домашнее задание 3. Работа с JSONB, индексами и TOAST в PostgreSQL

## 1. Cоздаём новую таблицу и заполняем таблицу миллионом записей с данными о пользователях: email, телефон (массив), имя пользователя, дата регистраци


```sql
postgres=# CREATE TABLE user_registrations (
    id SERIAL PRIMARY KEY,
    registration_data JSONB
);
CREATE TABLE
postgres=# INSERT INTO user_registrations (registration_data)
SELECT jsonb_build_object(
    'email', 'user' || gs || '@example.com',
    'phone', array['+1' || lpad((trunc(random() * 1000000000))::text, 10, '0')],
    'username', 'user_' || gs,
    'registered_at', to_char(now() - (random() * interval '5 years'), 'YYYY-MM-DD"T"HH24:MI:SS'),
    'active', (random() > 0.2)
)
FROM generate_series(1, 1000000) AS gs;
INSERT 0 1000000
```

---

## 4. Создаём GIN-индекс на поле JSONB

```sql
postgres=# CREATE INDEX idx_user_registrations_data ON user_registrations USING gin (registration_data);
CREATE INDEX
```
---

## 5. Обновляем поле `phone` для части записей

```sql
postgres=# UPDATE user_registrations
SET registration_data = jsonb_set(registration_data, '{phone,0}', to_jsonb('+1555' || id::text))
WHERE id % 1000 = 0;
UPDATE 1000
```

---

## 6. Проверяем размеры таблицы и TOAST

```sql
postgres=# SELECT
    n.nspname || '.' || c.relname AS table_name,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size,
    pg_size_pretty(pg_relation_size(c.oid)) AS table_size,
    pg_size_pretty(pg_total_relation_size(c.reltoastrelid)) AS toast_size
FROM pg_class c
JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE c.relname = 'user_registrations';
        table_name         | total_size | table_size | toast_size
---------------------------+------------+------------+------------
 public.user_registrations | 487 MB     | 191 MB     | 8192 bytes
```


Видим, что таблица в целом занимает 487 MB, но основной объём без TOAST — 191 MB.

Посмотрим на размер индекса
```sql
postgres=# SELECT
    indexrelid::regclass AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE relname = 'user_registrations';
         index_name          | index_size
-----------------------------+------------
 user_registrations_pkey     | 21 MB
 idx_user_registrations_data | 275 MB
(2 rows)

Time: 0.955 ms
```

И сравним с размером таблицы:
```sql
postgres=# SELECT
    pg_size_pretty (pg_relation_size('user_registrations')) size;
  size
--------
 191 MB
(1 row)
```

Индекс `idx_user_registrations_data` занимает 275 MB, при размере таблица в 191 MB.

---

## 7. Очистка блоатинга индекса (REINDEX)

```sql
postgres=# REINDEX INDEX CONCURRENTLY idx_user_registrations_data;
REINDEX
```

---

## 9. Проверяем размер индекса после REINDEX

```sql
postgres=# SELECT
    indexrelid::regclass AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE relname = 'user_registrations';
         index_name          | index_size
-----------------------------+------------
 user_registrations_pkey     | 21 MB
 idx_user_registrations_data | 275 MB
(2 rows)
```

Почему не изменился? Я обновлял каждую 1000 запись и мусора было минимум. Похоже повезло. Исправим ситуацию и обновим всю таблицу:

```sql
postgres=# UPDATE user_registrations
SET registration_data = jsonb_set(registration_data, '{phone,0}', to_jsonb('+15325093246' || id::text));
```

Посмотрим еще раз на размер таблицы:

```sql
postgres=# SELECT
    n.nspname || '.' || c.relname AS table_name,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size,
    pg_size_pretty(pg_relation_size(c.oid)) AS table_size,
    pg_size_pretty(pg_total_relation_size(c.reltoastrelid)) AS toast_size
FROM pg_class c
JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE c.relname = 'user_registrations';
        table_name         | total_size | table_size | toast_size
---------------------------+------------+------------+------------
 public.user_registrations | 801 MB     | 381 MB     | 8192 bytes
(1 row)
```

И на размер индекса:

```sql
postgres=#  SELECT
    indexrelid::regclass AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE relname = 'user_registrations';
         index_name          | index_size
-----------------------------+------------
 user_registrations_pkey     | 43 MB
 idx_user_registrations_data | 377 MB
(2 rows)
```

И сделаем *VACUUM FULL* чтобы наверняка:

```sql
postgres=# VACUUM FULL user_registrations;
VACUUM
Time: 41327.510 ms (00:41.328)
postgres=#  SELECT
    indexrelid::regclass AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE relname = 'user_registrations';
         index_name          | index_size
-----------------------------+------------
 user_registrations_pkey     | 21 MB
 idx_user_registrations_data | 301 MB
(2 rows)

Time: 1.306 ms
```

Конечно в хайлоаде такого нам никто не позволит сделать, так что попробуем еще раз параллельно переиндексировать:

```sql
postgres=# UPDATE user_registrations
SET registration_data = jsonb_set(registration_data, '{phone,0}', to_jsonb('+15312413246' || id::text))
;
UPDATE 1000000
Time: 104745.552 ms (01:44.746)
postgres=# SELECT
    indexrelid::regclass AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE relname = 'user_registrations';
         index_name          | index_size
-----------------------------+------------
 user_registrations_pkey     | 43 MB
 idx_user_registrations_data | 402 MB
(2 rows)

Time: 1.016 ms
postgres=#  REINDEX INDEX CONCURRENTLY idx_user_registrations_data;
REINDEX
Time: 45752.289 ms (00:45.752)
postgres=# SELECT
    indexrelid::regclass AS index_name,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE relname = 'user_registrations';
         index_name          | index_size
-----------------------------+------------
 user_registrations_pkey     | 43 MB
 idx_user_registrations_data | 301 MB
(2 rows)

Time: 1.196 ms
```

Как можем наблюдать, это действительно помогол
