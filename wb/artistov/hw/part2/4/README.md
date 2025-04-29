# Оконные функциии


## 1. Создадим таблицы и заполним данными с ЗП [script](https://www.db-fiddle.com/f/eQ8zNtAFY88i8nB4GRB65V/0)

И убедимся, что данные существуют:

```sql
postgres=# SELECT * FROM salary;
 fk_employee | amount | from_date  |  to_date   | fk_grade
-------------+--------+------------+------------+----------
           1 | 100000 | 2024-01-01 | 2024-01-31 |        1
           1 | 200000 | 2024-02-01 | 2024-02-29 |        2
           1 | 300000 | 2024-03-01 | 2099-12-31 |        3
           2 | 200000 | 2023-01-01 | 2024-01-31 |        2
           3 | 200000 | 2024-03-01 | 2024-01-31 |        2
           1 | 350000 | 2024-04-01 | 2024-04-30 |        4
           2 | 250000 | 2024-02-01 | 2024-02-29 |        3
           2 | 300000 | 2024-03-01 | 2099-12-31 |        4
           3 | 150000 | 2024-04-01 | 2024-04-30 |        1
(9 rows)
```


## 2. Пишем запрос с оконном функцией постепенно.


Используем LAG():
```sql
postgres=# SELECT
  fk_employee,
  from_date,
  amount,
  LAG(amount) OVER (ORDER BY from_date) AS previous_amount
FROM salary;
 fk_employee | from_date  | amount | previous_amount
-------------+------------+--------+-----------------
           2 | 2023-01-01 | 200000 |
           1 | 2024-01-01 | 100000 |          200000
           2 | 2024-02-01 | 250000 |          100000
           1 | 2024-02-01 | 200000 |          250000
           3 | 2024-03-01 | 200000 |          200000
           1 | 2024-03-01 | 300000 |          200000
           2 | 2024-03-01 | 300000 |          300000
           3 | 2024-04-01 | 150000 |          300000
           1 | 2024-04-01 | 350000 |          150000
(9 rows)
```

Здесь `LAG()` просто берет предыдущую зарплату в общем порядке — смешиваются сотрудники. Это неправильно.

----

Добавим к LAG  PARTITION BY
```sql
postgres=# SELECT
  fk_employee,
  from_date,
  amount,
  LAG(amount) OVER (PARTITION BY fk_employee ORDER BY from_date) AS previous_amount,
  amount - LAG(amount) OVER (PARTITION BY fk_employee ORDER BY from_date) AS difference
FROM salary
ORDER BY fk_employee, from_date;
 fk_employee | from_date  | amount | previous_amount | difference
-------------+------------+--------+-----------------+------------
           1 | 2024-01-01 | 100000 |                 |
           1 | 2024-02-01 | 200000 |          100000 |     100000
           1 | 2024-03-01 | 300000 |          200000 |     100000
           1 | 2024-04-01 | 350000 |          300000 |      50000
           2 | 2023-01-01 | 200000 |                 |
           2 | 2024-02-01 | 250000 |          200000 |      50000
           2 | 2024-03-01 | 300000 |          250000 |      50000
           3 | 2024-03-01 | 200000 |                 |
           3 | 2024-04-01 | 150000 |          200000 |     -50000
(9 rows)
```

Теперь:
- `PARTITION BY fk_employee` — для каждого сотрудника отдельно
- `ORDER BY from_date` — по времени
- `difference` — разница между текущей и предыдущей зарплатой

---

И напишем полную версию с JOIN-ами, учитывая null (первую ЗП):

```sql
postgres=# SELECT
    e.id AS employee_id,
    e.first_name || ' ' || e.last_name AS employee_name,
    e.gender,
    s.from_date,
    s.amount AS current_salary,
    g.value AS grade,
    COALESCE(s.amount - LAG(s.amount) OVER (PARTITION BY s.fk_employee ORDER BY s.from_date), 0) AS salary_difference
FROM
    salary s
JOIN
    employee e ON s.fk_employee = e.id
JOIN
    grade g ON s.fk_grade = g.id
ORDER BY
    e.id, s.from_date;
 employee_id | employee_name  | gender | from_date  | current_salary | grade  | salary_difference
-------------+----------------+--------+------------+----------------+--------+-------------------
           1 | Eugene Aristov | M      | 2024-01-01 |         100000 | junior |                 0
           1 | Eugene Aristov | M      | 2024-02-01 |         200000 | middle |            100000
           1 | Eugene Aristov | M      | 2024-03-01 |         300000 | senoir |            100000
           1 | Eugene Aristov | M      | 2024-04-01 |         350000 | lead   |             50000
           2 | Ivan Ivanov    | M      | 2023-01-01 |         200000 | middle |                 0
           2 | Ivan Ivanov    | M      | 2024-02-01 |         250000 | senoir |             50000
           2 | Ivan Ivanov    | M      | 2024-03-01 |         300000 | lead   |             50000
           3 | Petr Petrov    | M      | 2024-03-01 |         200000 | middle |                 0
           3 | Petr Petrov    | M      | 2024-04-01 |         150000 | junior |            -50000
(9 rows)
```

Учитываем также понижение (просто ради интереса).
