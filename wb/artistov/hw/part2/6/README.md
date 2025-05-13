# Домашнее задание: Оптимизация запроса и производительность с партициями

## Часть 1: Оптимизация запроса

Изначально был выполнен запрос подсчёта среднего процента чаевых по типам оплаты:

```sql
SELECT
    payment_type,
    ROUND(SUM(tips) / SUM(tips + fare) * 100) AS tips_percent,
    COUNT(*)
FROM
    taxi_trips
GROUP BY
    payment_type
ORDER BY
    3 DESC;
```

Время выполнения: **272 сек** (04:32.202)


Проапгрейдим немного:
```sql
SET work_mem = '64MB';
SET enable_parallel_hash = on;
SET enable_partition_pruning = on;
```

После включения настроек и повторного выполнения того же запроса — время снизилось до **165 сек** (02:45.722).

### Дальнейшая оптимизация: фильтрация по `payment_type`

Посмотрим на распределение данных:

```sql
postgres=# SELECT
    payment_type,
    ROUND(SUM(tips) / SUM(tips + fare) * 100) AS tips_percent,
    COUNT('')
FROM
    taxi_trips
GROUP BY
    payment_type
ORDER BY
    3 DESC;
 payment_type | tips_percent |  count
--------------+--------------+----------
 Cash         |            0 | 17231871
 Credit Card  |           18 |  9224956
 Unknown      |            0 |   103869
 Prcard       |            1 |    86053
 Mobile       |           17 |    61256
 No Charge    |            0 |    26294
 Pcard        |            2 |    13575
 Dispute      |            0 |     5596
 Split        |           19 |      180
 Way2ride     |           14 |       27
 Prepaid      |            0 |        6
```

Если посмотреть на соотношение:

| Payment_Type | Count |  Amount  | Percentage |
|--------------|-------|----------|------------|
| Cash         |     0 | 17231871 |   64.409%  |
| Credit Card  |    18 |  9224956 |   34.481%  |
| Unknown      |     0 |   103869 |    0.388%  |
| Prcard       |     1 |    86053 |    0.322%  |
| Mobile       |    17 |    61256 |    0.229%  |
| No Charge    |     0 |    26294 |    0.098%  |
| Pcard        |     2 |    13575 |    0.051%  |
| Dispute      |     0 |     5596 |    0.021%  |
| Split        |    19 |      180 |    0.001%  |
| Way2ride     |    14 |       27 |    0.000%  |
| Prepaid      |     0 |        6 |    0.000%  |


То первые две категории занимают 99%. Выделим три группы: Cash, Credit Card, и остальные. 
Как мне кажется, то тут очень хорошо подойдут предикатные индексы по payment_type и перепишем немного запрос таким образом, 
чтобы каждый тип payment_type считалс параллельно:

---

План выполнения нового запроса:

```sql
postgres=# explain SELECT * FROM (
    SELECT
        'Cash' AS payment_type,
        ROUND(SUM(tips) / SUM(tips + fare) * 100) AS tips_percent,
        COUNT(*) AS cnt
    FROM taxi_trips
    WHERE payment_type = 'Cash'

    UNION ALL

    SELECT
        'Credit Card',
        ROUND(SUM(tips) / SUM(tips + fare) * 100),
        COUNT(*)
    FROM taxi_trips
    WHERE payment_type = 'Credit Card'

    UNION ALL

    SELECT
        payment_type,
        ROUND(SUM(tips) / SUM(tips + fare) * 100),
        COUNT(*)
    FROM taxi_trips
    WHERE payment_type NOT IN ('Cash', 'Credit Card') group by payment_type
) sub
ORDER BY cnt DESC;
                                                              QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------
 Gather Merge  (cost=1066577.65..1066578.58 rows=8 width=52)
   Workers Planned: 2
   ->  Sort  (cost=1065577.63..1065577.64 rows=4 width=52)
         Sort Key: (count(*)) DESC
         ->  Parallel Append  (cost=0.42..1065577.59 rows=4 width=52)
               ->  GroupAggregate  (cost=0.42..856999.76 rows=9 width=47)
                     Group Key: taxi_trips.payment_type
                     ->  Index Scan using idx_other on taxi_trips  (cost=0.42..853053.38 rows=315694 width=16)
               ->  Aggregate  (cost=692017.37..692017.39 rows=1 width=72)
                     ->  Index Only Scan using idx_taxi_cash on taxi_trips taxi_trips_1  (cost=0.56..519973.35 rows=17204402 width=9)
               ->  Aggregate  (cost=373560.16..373560.18 rows=1 width=72)
                     ->  Index Only Scan using idx_taxi_card on taxi_trips taxi_trips_2  (cost=0.56..281224.27 rows=9233588 width=9)
 JIT:
   Functions: 9
   Options: Inlining true, Optimization true, Expressions true, Deforming true
(15 rows)

Time: 2.657 ms
```

Теперль мы наблюдаем параллельное использование индексов.
Сам запрос:

```sql
postgres=#  SELECT * FROM (
    SELECT
        'Cash' AS payment_type,
        ROUND(SUM(tips) / SUM(tips + fare) * 100) AS tips_percent,
        COUNT(*) AS cnt
    FROM taxi_trips
    WHERE payment_type = 'Cash'

    UNION ALL

    SELECT
        'Credit Card',
        ROUND(SUM(tips) / SUM(tips + fare) * 100),
        COUNT(*)
    FROM taxi_trips
    WHERE payment_type = 'Credit Card'

    UNION ALL

    SELECT
        payment_type,
        ROUND(SUM(tips) / SUM(tips + fare) * 100),
        COUNT(*)
    FROM taxi_trips
    WHERE payment_type NOT IN ('Cash', 'Credit Card') group by payment_type
) sub
ORDER BY cnt DESC;
 payment_type | tips_percent |   cnt
--------------+--------------+----------
 Cash         |            0 | 17231871
 Credit Card  |           18 |  9224956
 Unknown      |            0 |   103869
 Prcard       |            1 |    86053
 Mobile       |           17 |    61256
 No Charge    |            0 |    26294
 Pcard        |            2 |    13575
 Dispute      |            0 |     5596
 Split        |           19 |      180
 Way2ride     |           14 |       27
 Prepaid      |            0 |        6
(11 rows)

Time: 4039.382 ms (00:04.039)
```

Запрос бы исполнилс еще быстрее, но я столкнулся с той проблемой, что упирался в IO/Wait, так как используется в качестве хранения старый hdd.

---

## Часть 2: Проверка производительности с партиционированием

Создание партиций по месяцам с 2013 по 2023 год. Я сделал с помощью баш-скрипта:
```bash
for year in {2013..2023}; do
  for month in {01..12}; do
    start="${year}-${month}-01"
    end=$(date -d "$start +1 month" +%F)
    suffix=$(date -d "$start" +%Y%m)

    echo "
DROP TABLE IF EXISTS taxi_trips_range_p${suffix};
CREATE TABLE taxi_trips_range_p${suffix} PARTITION OF taxi_trips_range
FOR VALUES FROM ('$start') TO ('$end');"
  done
done > script.sql

psql -f script.sql
```

Перенос данных:

```sql
INSERT INTO taxi_trips_range (
    unique_key,
    taxi_id,
    trip_start_timestamp,
    trip_end_timestamp,
    trip_seconds,
    trip_miles,
    pickup_census_tract,
    dropoff_census_tract,
    pickup_community_area,
    dropoff_community_area,
    fare,
    tips,
    tolls,
    extras,
    trip_total,
    payment_type,
    company,
    pickup_latitude,
    pickup_longitude,
    pickup_location,
    dropoff_latitude,
    dropoff_longitude,
    dropoff_location
)
SELECT
    unique_key,
    taxi_id,
    trip_start_timestamp,
    trip_end_timestamp,
    trip_seconds::INTEGER,
    trip_miles::FLOAT,
    pickup_census_tract::BIGINT,
    dropoff_census_tract::BIGINT,
    pickup_community_area::INTEGER,
    dropoff_community_area::INTEGER,
    fare::NUMERIC,
    tips::NUMERIC,
    tolls::NUMERIC,
    extras::NUMERIC,
    trip_total::NUMERIC,
    payment_type,
    company,
    pickup_latitude::FLOAT,
    pickup_longitude::FLOAT,
    POINT(
        (regexp_match(pickup_location, 'POINT\\((-?[0-9.]+) (-?[0-9.]+)\\)'))[1]::FLOAT8,
        (regexp_match(pickup_location, 'POINT\\((-?[0-9.]+) (-?[0-9.]+)\\)'))[2]::FLOAT8
    ),
    dropoff_latitude::FLOAT,
    dropoff_longitude::FLOAT,
    POINT(
        (regexp_match(dropoff_location, 'POINT\\((-?[0-9.]+) (-?[0-9.]+)\\)'))[1]::FLOAT8,
        (regexp_match(dropoff_location, 'POINT\\((-?[0-9.]+) (-?[0-9.]+)\\)'))[2]::FLOAT8
    )
FROM taxi_trips;
```

## Проверка общей производительности (pgbench)

Прогоним тесты. До создания партиций:

```bash
postgres@dev-postgres-0:~$ pgbench -c 10 -j 2 -T 60 postgres
pgbench (17.5 (Debian 17.5-1.pgdg120+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 10
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 15236
number of failed transactions: 0 (0.000%)
latency average = 39.386 ms
initial connection time = 10.620 ms
tps = 253.897411 (without initial connection time)
```

И результаты после создания партиций:

```bash
postgres@dev-postgres-0:~$ pgbench -c 10 -j 2 -T 60 postgres
pgbench (17.5 (Debian 17.5-1.pgdg120+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 10
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 3785
number of failed transactions: 0 (0.000%)
latency average = 159.924 ms
initial connection time = 19.306 ms
tps = 62.529768 (without initial connection time)
```

Для наглядности
| Metric                            | Before Partitions | After Partitions |
|-----------------------------------|--------------------|------------------|
| Transactions Processed            | 15,236             | 3,785            |
| Failed Transactions               | 0                  | 0                |
| Latency Average                   | 39.386 ms          | 159.924 ms       |
| TPS (Throughput)                  | 253.90             | 62.53            |

---
