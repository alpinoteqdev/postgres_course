# Сравниваем производительность сингл инстанса и реплики

Сингл инстанс:
```bash
postgres@dev-postgres-2:~$ /usr/lib/postgresql/17/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload2.sql -n -U postgres -p 5432 thai
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload2.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 4084
number of failed transactions: 0 (0.000%)
latency average = 19.584 ms
initial connection time = 15.414 ms
tps = 408.489582 (without initial connection time)
```

С репликой асинхронной:
```bash
postgres@dev-postgres-0:~$ /usr/lib/postgresql/17/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload2.sql -n -U postgres -p 5432 thai
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload2.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 4763
number of failed transactions: 0 (0.000%)
latency average = 16.801 ms
initial connection time = 17.481 ms
tps = 476.174433 (without initial connection time)
```


Включим синхронную реплику. Я настроил через postgresl.conf, указав стендба и имя кластеры указал со звездочкой (ради простоты выполнения дз).
```sql
postgres=# SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 29227
usesysid         | 16509
usename          | replicator
application_name | 17/main
client_addr      | 10.22.22.2
client_hostname  |
client_port      | 59968
backend_start    | 2025-03-27 10:34:44.013068-04
backend_xmin     |
state            | streaming
sent_lsn         | 0/22000130
write_lsn        | 0/22000130
flush_lsn        | 0/22000130
replay_lsn       | 0/22000130
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 1
sync_state       | sync
reply_time       | 2025-03-27 10:35:54.059776-04
```

И ради интереса репликой синхронной:
```bash
postgres@dev-postgres-0:~$ /usr/lib/postgresql/17/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload2.sql -n -U postgres -p 5432 thai
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload2.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 1647
number of failed transactions: 0 (0.000%)
latency average = 49.115 ms
initial connection time = 16.666 ms
tps = 162.881848 (without initial connection time)
```

Итоговая таблица
| Configuration         | Transactions Processed | Average Latency (ms) | TPS       | Performance Impact       |
|-----------------------|------------------------|----------------------|-----------|--------------------------|
| Single Instance       | 4,084                  | 19.584               | 408.49    | Baseline                 |
| Async Replication     | 4,763                  | 16.801               | 476.17    | +16.6% faster            |
| Sync Replication      | 1,647                  | 49.115               | 162.88    | -60.1% slower            |

