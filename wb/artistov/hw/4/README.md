## Получаем дедлок

Чтобы получить блокировку, достаточно веести следующие запросы:


В первом терминале:

```sql
learn_locks=# begin;
BEGIN
learn_locks=*# UPDATE accounts SET ammount = ammount + 100 WHERE id = 1;
UPDATE 1
learn_locks=*# UPDATE accounts SET ammount = ammount + 200 WHERE id = 2;
UPDATE 1
learn_locks=*#
```

Во втором терминале:

```sql
learn_locks=# begin;
BEGIN
learn_locks=*#   UPDATE accounts SET ammount = ammount + 200 WHERE id = 2;
UPDATE 1
learn_locks=*#   UPDATE accounts SET ammount = ammount + 200 WHERE id = 1;
ERROR:  deadlock detected
DETAIL:  Process 2028 waits for ShareLock on transaction 967; blocked by process 1994.
Process 1994 waits for ShareLock on transaction 968; blocked by process 2028.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "accounts"
learn_locks=!#
```


### Подробно

1. Подключимся к отдельно созданной БД

```sql
postgres=# \c learn_locks ;
You are now connected to database "learn_locks" as user "postgres".
```


```sql
learn_locks=#  CREATE TABLE accounts (
       id serial,
       amount numeric
   );
```

```sql
learn_locks=# INSERT INTO accounts (id, amount) VALUES (1, 1000), (2, 2000), (3, 3000);
```


Посмотрим pid и txid в первом терминале:

```sql
learn_locks=# select pg_backend_pid();
 pg_backend_pid
----------------
           1994
(1 row)

learn_locks=# select txid_current();
 txid_current
--------------
          969
(1 row)
```

Посмотрим pid и txid во втором терминале:

```sql
learn_locks=# select pg_backend_pid();
 pg_backend_pid
----------------
           2028
(1 row)

learn_locks=# select txid_current();
 txid_current
--------------
          970
(1 row)
```

В первом терминале:

```sql
learn_locks=# begin;
BEGIN
learn_locks=*# UPDATE accounts SET ammount = ammount + 100 WHERE id = 1;
UPDATE 1
```

Во втором терминале:

```sql
learn_locks=# begin;
BEGIN
learn_locks=*#   UPDATE accounts SET ammount = ammount + 200 WHERE id = 2;
UPDATE 1
```

Посмотрим блокировки из вьюшки:
```sql
learn_locks=# select * from locks_v where pid <> pg_backend_pid() order by pid, locktype;
 pid  |   locktype    |  lockid  |       mode       | granted
------+---------------+----------+------------------+---------
 1994 | relation      | accounts | RowExclusiveLock | t
 1994 | transactionid | 971      | ExclusiveLock    | t
 1994 | virtualxid    | 5/18     | ExclusiveLock    | t
 2028 | relation      | accounts | RowExclusiveLock | t
 2028 | transactionid | 972      | ExclusiveLock    | t
 2028 | virtualxid    | 7/5      | ExclusiveLock    | t
(6 rows)
```

Видим, что транзакции взяли собственные строки в RowExclusiveLock, то есть запрещают доступ кому-либо.
Если мы попробуем обновить транзакцией 1994 строку, захваченную транзакцией 2028, то будем ожидать до тех пор, пока транзакция 2028 ее не освободит:


В первом терминале:

```sql
learn_locks=*# UPDATE accounts SET ammount = ammount + 200 WHERE id = 2;
UPDATE 1
learn_locks=*#
```

Посмотрим, кто блокирует нашу транзакцию 1994
```sql
learn_locks=# select pg_blocking_pids(1994);
 pg_blocking_pids
------------------
 {2028}
(1 row)
```

Также посмотрим на вьюшку еще раз:

```sql
learn_locks=# select * from locks_v where pid <> pg_backend_pid() order by pid, locktype;
 pid  |   locktype    |   lockid   |       mode       | granted
------+---------------+------------+------------------+---------
 1994 | relation      | accounts   | RowExclusiveLock | t
 1994 | transactionid | 972        | ShareLock        | f
 1994 | transactionid | 971        | ExclusiveLock    | t
 1994 | tuple         | accounts:2 | ExclusiveLock    | t    
 1994 | virtualxid    | 5/18       | ExclusiveLock    | t
 2028 | relation      | accounts   | RowExclusiveLock | t
 2028 | transactionid | 972        | ExclusiveLock    | t
 2028 | virtualxid    | 7/5        | ExclusiveLock    | t
(8 rows)
```

Обратите внимание на то, что 1994 ждет tuple accounts:2. 

И теперь, если попробуем транзакцией 2028 сделать любую эксклюзивную операцию, то получим дедлок. Например, следующая операция транзакции 2028 приведет к 
блокировке:
```sql
learn_locks=*# SELECT * from accounts where id = 1 FOR UPDATE;
```

Но, если вместо вышеприведенной эксклюзивной select for update мы попробуем выполнить следующее:
```sql
learn_locks=*# SELECT * from accounts where id = 1 FOR KEY SHARE;
 id | ammount
----+---------
  1 |    1000
(1 row)
```

То никакой блокировки не будет, а мы просто прочитаем актуальную версию строки, изменения которой транзакция 1994 не закоммитила
