## Создание функций 

Создадим таблицу:
```sql
CREATE TABLE sales (
  id         SERIAL PRIMARY KEY,
  sale_date  DATE,
  amount     NUMERIC(10,2)
);
```

И заполним данными, включая null:
```sql
INSERT INTO sales (sale_date, amount) VALUES
  ('2025-01-10',  150.00),
  ('2025-04-27',   75.50),
  ('2025-05-12',  200.00),
  ('2025-08-01',  125.25),
  ('2025-09-30',  300.00),
  (NULL,           50.00);
```

----

Создадим 3 функции: одну через case и две "математические".

#### функция с case

```sql
CREATE OR REPLACE FUNCTION third_of_year_case(p_date DATE)
  RETURNS INTEGER
  LANGUAGE SQL AS
$$
  SELECT
    CASE
      WHEN p_date IS NULL THEN NULL
      WHEN EXTRACT(MONTH FROM p_date)::int BETWEEN 1 AND 4  THEN 1
      WHEN EXTRACT(MONTH FROM p_date)::int BETWEEN 5 AND 8  THEN 2
      ELSE 3
    END;
$$;
```

#### Функция с целочисленным делением

Используем целочисленное деление: (month + 3) / 4 даёт 1 для 1–4, 2 для 5–8 и 3 для 9–12.

```sql
CREATE OR REPLACE FUNCTION third_of_year_math1(p_date DATE)
  RETURNS INTEGER
  LANGUAGE SQL AS
$$
  SELECT
    CASE
      WHEN p_date IS NULL THEN NULL
      ELSE (EXTRACT(MONTH FROM p_date)::int + 3) / 4
    END;
$$;
```

#### Функция с использованием ceil

Также можно использовать CEIL(month/4):
```sql
CREATE OR REPLACE FUNCTION third_of_year_math2(p_date DATE)
  RETURNS INTEGER
  LANGUAGE SQL AS
$$
  SELECT
    CASE
      WHEN p_date IS NULL THEN NULL
      ELSE CEIL(EXTRACT(MONTH FROM p_date)::numeric / 4)::int
    END;
$$;
```

----

Проверим работу функций одним запросов
```sql
postgres=# SELECT
  id,
  sale_date,
  amount,
  third_of_year_case(sale_date)         AS case,
  third_of_year_math1(sale_date)        AS divis,
  third_of_year_math2(sale_date)        AS ceil
FROM sales
ORDER BY id;
 id | sale_date  | amount | case | divis | ceil
----+------------+--------+------+-------+------
  1 | 2025-01-10 | 150.00 |    1 |     1 |    1
  2 | 2025-04-27 |  75.50 |    1 |     1 |    1
  3 | 2025-05-12 | 200.00 |    2 |     2 |    2
  4 | 2025-08-01 | 125.25 |    2 |     2 |    2
  5 | 2025-09-30 | 300.00 |    3 |     3 |    3
  6 |            |  5
```
