# Триггеры в PostgreSQL

## Шаг 1: Добавил колонку в таблицу ride

Чтобы не пересчитывать занятые места каждый раз, я добавил новое поле `occupied_seats` в таблицу `ride`.
Насколько это хорошее или плохое решение -не могу сказать, имею минимумт опыта.

```sql
ALTER TABLE book.ride ADD COLUMN occupied_seats INTEGER DEFAULT 0;
```

---

## Шаг 2: Создание триггера на вставку

Я написал функцию и триггер, который увеличивает количество занятых мест при добавлении билета в таблицу `tickets`. 
Также добавлена проверка, чтобы не продать больше мест, чем есть в автобусе. 

### Функция триггера на вставку

```sql
CREATE OR REPLACE FUNCTION book.safe_increment_occupied_seats()
RETURNS TRIGGER AS $$
DECLARE
    max_seats INTEGER;
    current_occupied INTEGER;
BEGIN
    SELECT COUNT(*) INTO max_seats
    FROM book.seat
    WHERE fkbus = (
        SELECT fkbus FROM book.ride WHERE id = NEW.fkride
    );

    SELECT occupied_seats INTO current_occupied
    FROM book.ride WHERE id = NEW.fkride;

    IF current_occupied >= max_seats THEN
        RAISE EXCEPTION 'Sold more than we have';
    END IF;

    UPDATE book.ride
    SET occupied_seats = occupied_seats + 1
    WHERE id = NEW.fkride;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### Привязка триггера

```sql
CREATE TRIGGER trg_increment_seats
BEFORE INSERT ON book.tickets
FOR EACH ROW
EXECUTE FUNCTION book.safe_increment_occupied_seats();
```

---

## Шаг 3: Триггер на удаление

Я также добавил триггер, который уменьшает количество занятых мест при удалении билета. При этом добавлена защита от отрицательных значений.

### Функция триггера на удаление

```sql
CREATE OR REPLACE FUNCTION book.decrement_occupied_seats()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE book.ride
    SET occupied_seats = GREATEST(0, occupied_seats - 1)
    WHERE id = OLD.fkride;
    RETURN OLD;
END;
$$ LANGUAGE plpgsql;
```

### Привязка триггера

```sql
CREATE TRIGGER trg_decrement_seats
AFTER DELETE ON book.tickets
FOR EACH ROW
EXECUTE FUNCTION book.decrement_occupied_seats();
```

---

## Шаг 4: Измерение производительности

Я замерил время вставки 1000 билетов в таблицу с отключённым триггером и с включённым. 
Чтобы соблюсти ограничения внешних ключей, использовал реальные значения `fkride` и `fkseat`.
Перед выполнением лямбда-функции я выключал и включал триггер.

### Код вставки билетов

```sql
DO $$
DECLARE
  seat_id INTEGER;
BEGIN
  FOR i IN 1..1000 LOOP
    seat_id := floor(random() * 199 + 1); -- примерный диапазон допустимых значений

    INSERT INTO book.tickets (fkride, fkseat, fio, contact)
    VALUES (
      1,
      seat_id,
      'User ' || i,
      '{"email": "contact@example.com"}'
    );
  END LOOP;
END;
$$;
```

### Результаты

| Условие       | Время выполнения |
|---------------|------------------|
| Без триггера  | 55.730 мс        |
| С триггером   | 133.304 мс       |

Триггер замедлил вставку примерно в 2.4 раза. Это ожидаемо, так как при каждой вставке происходит дополнительное обновление в таблице `ride`.
С одной стороны триггер позволяет держать логику в одном местет, но мы наблюдаем серьезные издержки за это.
