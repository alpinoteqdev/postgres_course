# Протестировать падение производителности при исползовании pgbouncer в разнх
режимах: statement, transaction, session

Хар-ки ВМ-ки

| Характеристика | Значение                |
|----------------|-------------------------|
| CPU            | 2 cores (1 socket)      |
| Memory         | 4.00 GiB               |
| HDD            | 35G (SCSI, local-lvm)  |

----
## Session

```bash
@dev:~# /usr/lib/postgresql/17/bin/pgbench -c 10 -j 4 -T 10 -f /var/lib/postgresql/workload.sql -n -U postgres -p 6432 -h 127.0.0.1 thai
Password:
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 2175
number of failed transactions: 0 (0.000%)
latency average = 46.087 ms
initial connection time = 70.604 ms
tps = 216.979466 (without initial connection time)
```

---
## Transaction

```bash
@dev:~# /usr/lib/postgresql/17/bin/pgbench -c 10 -j 4 -T 10 -f /var/lib/postgresql/workload.sql -n -U postgres -p 6432 -h 127.0.0.1 thai
Password:
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 2490
number of failed transactions: 0 (0.000%)
latency average = 39.953 ms
initial connection time = 91.625 ms
tps = 250.294926 (without initial connection time)
```

---
## Statement

```bash
@dev:~# /usr/lib/postgresql/17/bin/pgbench -c 10 -j 4 -T 10 -f /var/lib/postgresql/workload.sql -n -U postgres -p 6432 -h 127.0.0.1 thai
Password:
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 2729
number of failed transactions: 0 (0.000%)
latency average = 36.527 ms
initial connection time = 71.758 ms
tps = 273.766471 (without initial connection time)
```



----
## Таблица

| Mode       | TPS               | Latency Average | Initial Connection Time |
|------------|-------------------|-----------------|-------------------------|
| Session    | 216.979466        | 46.087 ms       | 70.604 ms               |
| Transaction| 250.294926        | 39.953 ms       | 91.625 ms               |
| Statement  | 273.766471        | 36.527 ms       | 71.758 ms               |
