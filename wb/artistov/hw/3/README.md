1. Создать таблицу с текстовым полем и заполнить случайными или сгенерированными
данным в размере 1 млн строк

Создадим отдельную базу и заполним таблицу `vacuum_test`:
```sql
postgres=# CREATE DATABASE VACUUM_LEARN;
CREATE DATABASE
postgres=# \c vacuum_learn;
You are now connected to database "vacuum_learn" as user "postgres".
vacuum_learn=# create table vacuum_test(id serial, some_text text);
CREATE TABLE
vacuum_learn=# INSERT INTO vacuum_test (some_text) SELECT md5(random()::text) FROM generate_series(1, 1000000);
INSERT 0 1000000
vacuum_learn=#
vacuum_learn=# select count(id) from vacuum_test;
  count
---------
 1000000
(1 row)
```

3. Посмотреть размер файла с таблицей

Таблица занимает всего 65МБ
```sql
vacuum_learn=# \dt+
                                      List of relations
 Schema |    Name     | Type  |  Owner   | Persistence | Access method | Size  | Description
--------+-------------+-------+----------+-------------+---------------+-------+-------------
 public | vacuum_test | table | postgres | permanent   | heap          | 65 MB |
```

4. 5 раз обновить все строчки и добавить к каждой строчке любой символ

Обновляем текстовое поле все таблицы, добавив случайный символ:
```sql
vacuum_learn=# UPDATE vacuum_test set some_text = some_text || floor(random()*26+65)::text;
 UPDATE vacuum_test set some_text = some_text || floor(random()*26+65)::text;
 UPDATE vacuum_test set some_text = some_text || floor(random()*26+65)::text;
 UPDATE vacuum_test set some_text = some_text || floor(random()*26+65)::text;
 UPDATE vacuum_test set some_text = some_text || floor(random()*26+65)::text;
```

5. Посмотреть количество мертвых строчек в таблице и когда последний раз приходил
автовакуум

Последний автовакум был в 13:07:05
```sql
vacuum_learn=# SELECT relname, n_live_tup, n_dead_tup,
trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
FROM pg_stat_user_tables WHERE relname = 'vacuum_test';
   relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
-------------+------------+------------+--------+-------------------------------
 vacuum_test |     998767 |    1000000 |    100 | 2025-03-14 13:07:05.903205-04
```

6. Подождать некоторое время, проверяя, пришел ли автовакуум

Повторил 2 минуты запрос и увидели, что автовакуум пришел:
```sql
vacuum_learn=# UPDATE vacuum_test set some_text = some_text || floor(random()*26+65)::text;
   relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
-------------+------------+------------+--------+-------------------------------
 vacuum_test |     999441 |          0 |      0 | 2025-03-14 13:08:02.074574-04
(1 row)

```

7. 5 раз обновить все строчки и добавить к каждой строчке любой символ

```sql
vacuum_learn=# UPDATE vacuum_test set some_text = some_text || floor(random()*26+65)::text;
UPDATE 1000000
vacuum_learn=# UPDATE vacuum_test set some_text = some_text || floor(random()*26+65)::text;
UPDATE 1000000
vacuum_learn=# UPDATE vacuum_test set some_text = some_text || floor(random()*26+65)::text;
UPDATE 1000000
vacuum_learn=# UPDATE vacuum_test set some_text = some_text || floor(random()*26+65)::text;
UPDATE 1000000
vacuum_learn=# UPDATE vacuum_test set some_text = some_text || floor(random()*26+65)::text;
UPDATE 1000000
```

8. Посмотреть размер файла с таблицей

```sql
                                      List of relations
 Schema |    Name     | Type  |  Owner   | Persistence | Access method |  Size  | Description
--------+-------------+-------+----------+-------------+---------------+--------+-------------
 public | vacuum_test | table | postgres | permanent   | heap          | 484 MB |
(1 row)
```

9. Отключить Автовакуум на конкретной таблице

Отключим автовакуум:
```sql
vacuum_learn=# ALTER TABLE vacuum_test SET (autovacuum_enabled = off);
ALTER TABLE
```

Запомним, что последний авктовакум был в 13:16:02:
```sql
vacuum_learn=# SELECT
  schemaname, relname,
  last_vacuum, last_autovacuum,
  vacuum_count, autovacuum_count  -- not available on 9.0 and earlier
FROM pg_stat_user_tables;
 schemaname |   relname   | last_vacuum |        last_autovacuum        | vacuum_count | autovacuum_count
------------+-------------+-------------+-------------------------------+--------------+------------------
 public     | vacuum_test |             | 2025-03-14 13:16:02.985533-04 |            0 |                6
(1 row)
```

10. 10 раз обновить все строчки и добавить к каждой строчке любой символ


```sql
vacuum_learn=# UPDATE vacuum_test set some_text = some_text || floor(random()*26+65)::text;
UPDATE 1000000
vacuum_learn=# UPDATE vacuum_test set some_text = some_text || floor(random()*26+65)::text;
UPDATE 1000000
vacuum_learn=# UPDATE vacuum_test set some_text = some_text || floor(random()*26+65)::text;
UPDATE 1000000
vacuum_learn=# UPDATE vacuum_test set some_text = some_text || floor(random()*26+65)::text;
UPDATE 1000000
vacuum_learn=# UPDATE vacuum_test set some_text = some_text || floor(random()*26+65)::text;
UPDATE 1000000
vacuum_learn=# UPDATE vacuum_test set some_text = some_text || floor(random()*26+65)::text;
UPDATE 1000000
vacuum_learn=# UPDATE vacuum_test set some_text = some_text || floor(random()*26+65)::text;
UPDATE 1000000
vacuum_learn=# UPDATE vacuum_test set some_text = some_text || floor(random()*26+65)::text;
UPDATE 1000000
vacuum_learn=# UPDATE vacuum_test set some_text = some_text || floor(random()*26+65)::text;
UPDATE 1000000
vacuum_learn=# UPDATE vacuum_test set some_text = some_text || floor(random()*26+65)::text;
UPDATE 1000000
vacuum_learn=# \dt+
                                       List of relations
 Schema |    Name     | Type  |  Owner   | Persistence | Access method |  Size   | Description
--------+-------------+-------+----------+-------------+---------------+---------+-------------
 public | vacuum_test | table | postgres | permanent   | heap          | 1158 MB |
(1 row)

```

Обращаю внимание, что последний автовакум был в 13:16:02, то есть он не приходил

```sql
vacuum_learn=# SELECT relname, n_live_tup, n_dead_tup,
trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
FROM pg_stat_user_tables WHERE relname = 'vacuum_test';
   relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
-------------+------------+------------+--------+-------------------------------
 vacuum_test |    1650289 |   10997070 |    666 | 2025-03-14 13:16:02.985533-04
(1 row)
```

И ради интереса запустил explain:
```sql
vacuum_learn=# explain(analyze) select * from vacuum_test;
                                                       QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------
 Seq Scan on vacuum_test  (cost=0.00..187598.70 rows=3946770 width=57) (actual time=9.932..219.810 rows=1000000 loops=1)
 Planning Time: 0.037 ms
 Execution Time: 245.443 ms
(3 rows)
```

11. Посмотреть размер файла с таблицей
12. Объясните полученный результат

Поскольку автовакуум не придет, а делать 2млрд транзакций, чтобы "сломать" постгрес было лень, попробовал сделать автовакум сам.
```sql
vacuum_learn=# VACUUM(ANALYZE) vacuum_test ;
VACUUM
vacuum_learn=# \dt+
                                       List of relations
 Schema |    Name     | Type  |  Owner   | Persistence | Access method |  Size   | Description
--------+-------------+-------+----------+-------------+---------------+---------+-------------
 public | vacuum_test | table | postgres | permanent   | heap          | 1158 MB |
(1 row)
```

Меня удивило, что размер таблички не изменился после автовакума, хотя мертвые строчки были удалены:

```sql
vacuum_learn=# SELECT relname, n_live_tup, n_dead_tup,
trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
FROM pg_stat_user_tables WHERE relname = 'vacuum_test';
   relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
-------------+------------+------------+--------+-------------------------------
 vacuum_test |     999553 |          0 |      0 | 2025-03-14 13:16:02.985533-04
(1 row)
```

Обратившись к документации, понял в чем дело:
```
Простая команда VACUUM (без FULL) только высвобождает пространство и делает его доступным для повторного использования.
Эта форма команды может работать параллельно с обычными операциями чтения и записи таблицы, так она не требует исключительной блокировки.
Однако освобождённое место не возвращается операционной системе (в большинстве случаев); оно просто остаётся доступным для размещения данных этой же таблицы.
VACUUM FULL переписывает всё содержимое таблицы в новый файл на диске, не содержащий ничего лишнего, что позволяет возвратить неиспользованное пространство операционной системе
```

Убедимся на практике:
```sql
vacuum_learn=# VACUUM FULL vacuum_test
vacuum_learn-# ;
VACUUM
vacuum_learn=# \dt+ vacuum_test ;
                                      List of relations
 Schema |    Name     | Type  |  Owner   | Persistence | Access method | Size  | Description
--------+-------------+-------+----------+-------------+---------------+-------+-------------
 public | vacuum_test | table | postgres | permanent   | heap          | 35 MB |
(1 row)
```
И действительноя, табличка уменьшилась в размере. (Табличка стала еще меньше, чем в самом начале (65МБ было), так как в процессе изучения модифицировал строки по разному).


И ради интереса запустил explain без мертвых строк:
```sql
vacuum_learn=# explain(analyze) select * from vacuum_test;
                                                       QUERY PLAN
------------------------------------------------------------------------------------------------------------------------
 Seq Scan on vacuum_test  (cost=0.00..158126.53 rows=999553 width=79) (actual time=8.719..215.303 rows=1000000 loops=1)
 Planning Time: 0.102 ms
 Execution Time: 245.370 ms
(3 rows)
```

Табличка, для удобства сравнения

|Значние|With dirty tuples|Without dirty tuples|
|--------|-----------------|--------------------|
|cost|  0.00..187598.70 | 0.00..158126.53|
| rows|  3946770 | 999553|
|actual time|9.932..219.810|  8.719..215.303|
|execution time| 245.443 ms | 245.370 ms|

Я бы обратил внимание на значение rows - 3 миллиона в случае мертвых строк, против 1 миллиона без, но execution time остался тем же.
Хотел убедиться, что без очищения мертвых строк действительно запросы будут хуже работать и, возможно, на более объемных данных разница была бы заметнее.
Да и опыта в explain, как и во всем postgres у меня минимальный, мб что-то упускаю.

-----

Ради интереса повторил весь эксперимент, но уже сделал вакум ручками с VERBOSE и увидел, что действительно строчки удаляются, вот часть лога:
```sql
vacuum_learn=# VACUUM VERBOSE;
INFO:  finished vacuuming "vacuum_learn.pg_catalog.pg_sequence": index scans: 0
pages: 0 removed, 1 remain, 1 scanned (100.00% of total)
tuples: 1 removed, 1 remain, 0 are dead but not yet removable   <<<--------
```


13. Не забудьте включить автовакуум)
Задание со *:
Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в
искомой таблице.
Не забыть вывести номер шага цикла.

ЕСли я правильно понял, что подразумевается под анонимной процедурой:
```sql

vacuum_learn=# DO $$
vacuum_learn$# DECLARE
vacuum_learn$# step INT;
vacuum_learn$# BEGIN
vacuum_learn$# FOR step in 1..10 LOOP
vacuum_learn$# UPDATE vacuum_test set some_text = some_text || floor(random() * 26 + 65)::text;
vacuum_learn$# RAISE NOTICE 'Iterator % has ended', step;
vacuum_learn$# END LOOP;
vacuum_learn$# RAISE NOTICE 'done';
vacuum_learn$# END $$;
NOTICE:  Iterator 1 has ended
NOTICE:  Iterator 2 has ended
NOTICE:  Iterator 3 has ended
NOTICE:  Iterator 4 has ended
NOTICE:  Iterator 5 has ended
NOTICE:  Iterator 6 has ended
NOTICE:  Iterator 7 has ended
NOTICE:  Iterator 8 has ended
NOTICE:  Iterator 9 has ended
NOTICE:  Iterator 10 has ended
NOTICE:  done
```
