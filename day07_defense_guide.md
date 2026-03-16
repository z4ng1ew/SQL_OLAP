# Day 07 — Разбор решений для защиты (P2P Review)

---

## Exercise 00 — Simple aggregated information

### Решение

```sql
SELECT
    person_id,                  -- 1. Выбираем столбец person_id из таблицы person_visits
    COUNT(*) AS count_of_visits -- 2. COUNT(*) считает количество ВСЕХ строк в каждой группе
                                --    AS count_of_visits — задаём псевдоним (alias) столбцу
FROM person_visits              -- 3. Источник данных — таблица журнала посещений
GROUP BY person_id              -- 4. Группируем строки по person_id:
                                --    все строки с одинаковым person_id схлопываются в одну,
                                --    а COUNT(*) подсчитывает, сколько их было
ORDER BY 2 DESC, 1 ASC;        -- 5. Сортировка:
                                --    2 DESC — по второму столбцу (count_of_visits) по убыванию
                                --    1 ASC  — при равенстве — по первому (person_id) по возрастанию
```

### Что спросят на защите

- **Зачем GROUP BY?** — Без GROUP BY агрегатная функция COUNT(*) посчитает ВСЕ строки таблицы и вернёт одно число. GROUP BY разбивает таблицу на группы, и COUNT считает строки внутри каждой группы отдельно.
- **Чем COUNT(*) отличается от COUNT(column)?** — COUNT(*) считает все строки, включая NULL. COUNT(column) считает только строки, где column НЕ NULL.
- **Что значит ORDER BY 2?** — Это ссылка по порядковому номеру столбца в SELECT. 2 = второй столбец = count_of_visits. Альтернатива — писать имя столбца явно.

### Альтернативное решение

```sql
-- Вариант: COUNT(person_id) вместо COUNT(*)
-- Результат тот же, потому что person_id не содержит NULL
SELECT
    person_id,
    COUNT(person_id) AS count_of_visits
FROM person_visits
GROUP BY person_id
ORDER BY count_of_visits DESC, person_id ASC;
-- Здесь сортировка по именам столбцов, а не по номерам — более читаемо
```

---

## Exercise 01 — Let's see real names

### Решение

```sql
SELECT
    prs.name,                   -- 1. Берём имя из таблицы person (через алиас prs)
    COUNT(*) AS count_of_visits -- 2. Считаем количество посещений в каждой группе
FROM person_visits              -- 3. Основная таблица — журнал посещений
    INNER JOIN person AS prs    -- 4. INNER JOIN — соединяем с таблицей person
        ON prs.id = person_visits.person_id
                                -- 5. Условие связи: person.id = person_visits.person_id
                                --    INNER JOIN отбросит строки, где нет совпадения
GROUP BY prs.name               -- 6. Группируем по имени человека
ORDER BY 2 DESC, 1 ASC         -- 7. Сортируем: сначала по количеству (убыв.), потом по имени (возр.)
LIMIT 4;                        -- 8. Берём только первые 4 строки (топ-4)
```

### Что спросят на защите

- **Почему INNER JOIN, а не LEFT JOIN?** — Нам нужны только посещения, у которых есть человек. INNER JOIN вернёт только совпавшие строки. LEFT JOIN вернул бы то же самое, потому что person_id — внешний ключ (foreign key) и не может ссылаться на несуществующую запись, но INNER JOIN семантически точнее.
- **Почему GROUP BY по name, а не по id?** — Потому что в SELECT стоит name. Всё, что не внутри агрегатной функции, ОБЯЗАНО быть в GROUP BY. Если два человека имеют одинаковое имя, их посещения сольются — но в нашей базе имена уникальны.
- **Что делает LIMIT?** — Ограничивает вывод N строками. LIMIT работает ПОСЛЕ ORDER BY, поэтому мы получаем именно топ-4.

### Альтернативное решение

```sql
-- Вариант с подзапросом: сначала считаем в person_visits, потом подставляем имена
SELECT
    prs.name,
    agg.count_of_visits
FROM (
    SELECT
        person_id,
        COUNT(*) AS count_of_visits
    FROM person_visits
    GROUP BY person_id
) AS agg
    INNER JOIN person AS prs ON prs.id = agg.person_id
ORDER BY agg.count_of_visits DESC, prs.name ASC
LIMIT 4;
-- Логика: подзапрос agg считает визиты по person_id,
-- затем JOIN подтягивает имена.
-- Результат тот же, но может быть производительнее на больших данных,
-- потому что группировка идёт по числовому id, а не по строке name.
```

---

## Exercise 02 — Restaurants statistics

### Решение

```sql
(                                           -- Первый подзапрос в скобках (для UNION)
    SELECT
        pzr.name,                           -- 1. Название пиццерии
        COUNT(*) AS count,                  -- 2. Количество посещений
        'visit' AS action_type              -- 3. Литеральная строка 'visit' — метка типа действия
    FROM person_visits
        INNER JOIN pizzeria AS pzr          -- 4. Соединяем с pizzeria, чтобы получить имя
            ON pzr.id = person_visits.pizzeria_id
    GROUP BY pzr.name                       -- 5. Группируем по названию пиццерии
    ORDER BY 2 DESC                         -- 6. Сортируем по количеству (убыв.) ВНУТРИ этого подзапроса
    LIMIT 3                                 -- 7. Топ-3 по посещениям
)
UNION                                       -- 8. UNION объединяет два набора строк
                                            --    и удаляет полные дубликаты (UNION ALL — не удаляет)
(                                           -- Второй подзапрос
    SELECT
        pzr.name,                           -- 9.  Название пиццерии
        COUNT(*) AS count,                  -- 10. Количество заказов
        'order' AS action_type              -- 11. Метка 'order'
    FROM person_order
        INNER JOIN menu AS mn               -- 12. person_order → menu (чтобы узнать pizzeria_id)
            ON person_order.menu_id = mn.id
        INNER JOIN pizzeria AS pzr          -- 13. menu → pizzeria (чтобы узнать имя)
            ON mn.pizzeria_id = pzr.id
    GROUP BY pzr.name                       -- 14. Группируем по названию
    ORDER BY 2 DESC                         -- 15. Сортируем по количеству (убыв.)
    LIMIT 3                                 -- 16. Топ-3 по заказам
)
ORDER BY 3, 2 DESC;                         -- 17. Финальная сортировка объединённого результата:
                                            --     3 = action_type (ASC по умолчанию, 'order' < 'visit')
                                            --     2 = count (DESC)
```

### Что спросят на защите

- **Зачем скобки вокруг каждого SELECT?** — В PostgreSQL скобки позволяют использовать ORDER BY и LIMIT внутри каждой «ветки» UNION. Без скобок ORDER BY применялся бы ко всему объединённому результату.
- **Почему UNION, а не UNION ALL?** — UNION убирает дубликаты. Здесь дубликатов не будет (action_type различается), так что UNION ALL тоже подошёл бы и работал бы чуть быстрее. Но оба варианта корректны.
- **Почему два JOIN для заказов?** — Таблица person_order хранит menu_id, а не pizzeria_id. Путь: person_order → menu → pizzeria. Для визитов путь короче: person_visits → pizzeria.

### Альтернативное решение

```sql
-- Вариант с UNION ALL (быстрее, дубликатов всё равно нет из-за action_type)
(
    SELECT pzr.name, COUNT(*) AS count, 'visit' AS action_type
    FROM person_visits
        INNER JOIN pizzeria AS pzr ON pzr.id = person_visits.pizzeria_id
    GROUP BY pzr.name
    ORDER BY count DESC
    LIMIT 3
)
UNION ALL
(
    SELECT pzr.name, COUNT(*) AS count, 'order' AS action_type
    FROM person_order
        INNER JOIN menu AS mn ON person_order.menu_id = mn.id
        INNER JOIN pizzeria AS pzr ON mn.pizzeria_id = pzr.id
    GROUP BY pzr.name
    ORDER BY count DESC
    LIMIT 3
)
ORDER BY action_type, count DESC;
```

---

## Exercise 03 — Restaurants statistics #2

### Решение

```sql
SELECT
    visits_stats.name,          -- 1. Имя пиццерии (из подзапроса визитов)
    COALESCE(visits_stats.count, 0)
        + COALESCE(orders_stats.count, 0)
        AS total_count          -- 2. COALESCE(x, 0) — если x = NULL, подставляет 0
                                --    Это нужно, потому что FULL JOIN может дать NULL
                                --    с одной из сторон (пиццерия есть в визитах, но не в заказах)
FROM
    (                           -- 3. Подзапрос: количество визитов по каждой пиццерии
        SELECT
            pzr.name,
            COUNT(*) AS count
        FROM person_visits
            INNER JOIN pizzeria AS pzr ON pzr.id = person_visits.pizzeria_id
        GROUP BY pzr.name
    ) AS visits_stats
    FULL JOIN                   -- 4. FULL JOIN — берём ВСЕ записи из обеих сторон.
                                --    Если пиццерия есть только в визитах — count из orders будет NULL
                                --    Если только в заказах — count из visits будет NULL
    (                           -- 5. Подзапрос: количество заказов по каждой пиццерии
        SELECT
            pzr.name,
            COUNT(*) AS count
        FROM person_order
            INNER JOIN menu AS mn ON person_order.menu_id = mn.id
            INNER JOIN pizzeria AS pzr ON mn.pizzeria_id = pzr.id
        GROUP BY pzr.name
    ) AS orders_stats
        ON visits_stats.name = orders_stats.name
                                -- 6. Связываем по имени пиццерии
ORDER BY 2 DESC, 1 ASC;        -- 7. Сортировка: total_count убыв., имя возр.
```

### Что спросят на защите

- **Зачем FULL JOIN, а не INNER JOIN?** — INNER JOIN потеряет пиццерии, которые есть только в одной из таблиц. Например, DoDo Pizza имеет визиты, но нет заказов. FULL JOIN сохраняет всё.
- **Зачем COALESCE?** — При FULL JOIN одна сторона может быть NULL. NULL + 5 = NULL (любая арифметика с NULL даёт NULL). COALESCE(NULL, 0) = 0, и сложение работает корректно.
- **Почему нельзя просто UNION и потом GROUP BY?** — Можно! Это альтернативный вариант (см. ниже).

### Альтернативное решение

```sql
-- Вариант: UNION ALL + внешняя группировка (без FULL JOIN)
SELECT
    name,
    SUM(count) AS total_count
FROM (
    SELECT pzr.name, COUNT(*) AS count
    FROM person_visits
        INNER JOIN pizzeria AS pzr ON pzr.id = person_visits.pizzeria_id
    GROUP BY pzr.name

    UNION ALL

    SELECT pzr.name, COUNT(*) AS count
    FROM person_order
        INNER JOIN menu AS mn ON person_order.menu_id = mn.id
        INNER JOIN pizzeria AS pzr ON mn.pizzeria_id = pzr.id
    GROUP BY pzr.name
) AS combined
GROUP BY name
ORDER BY total_count DESC, name ASC;
-- Проще и понятнее: объединяем оба набора, потом суммируем по имени.
-- UNION ALL обязателен, потому что UNION удалил бы строки
-- с одинаковым name + count (если count совпадёт).
```

---

## Exercise 04 — Clause for groups

### Решение

```sql
SELECT
    prs.name,                   -- 1. Имя человека
    COUNT(*) AS count_of_visits -- 2. Количество посещений
FROM person_visits
    INNER JOIN person AS prs    -- 3. JOIN с person для получения имени
        ON prs.id = person_visits.person_id
GROUP BY prs.name               -- 4. Группируем по имени
HAVING COUNT(*) > 3;            -- 5. HAVING — фильтр ПОСЛЕ группировки
                                --    Оставляем только группы, где количество > 3
                                --    WHERE тут ЗАПРЕЩЕНО условием задания
```

### Что спросят на защите

- **Чем HAVING отличается от WHERE?** — WHERE фильтрует строки ДО группировки (до GROUP BY). HAVING фильтрует группы ПОСЛЕ группировки. В HAVING можно использовать агрегатные функции (COUNT, SUM и т.д.), в WHERE — нельзя.
- **Порядок выполнения SQL:** FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT. HAVING идёт после GROUP BY, поэтому он «видит» результаты агрегации.
- **Почему нет ORDER BY?** — Задание не требует сортировки, и в результате всего одна строка (Dmitriy с 4 визитами).

### Альтернативное решение

```sql
-- Если бы WHERE был разрешён, можно было бы использовать подзапрос:
SELECT name, count_of_visits
FROM (
    SELECT
        prs.name,
        COUNT(*) AS count_of_visits
    FROM person_visits
        INNER JOIN person AS prs ON prs.id = person_visits.person_id
    GROUP BY prs.name
) AS stats
WHERE count_of_visits > 3;
-- Но в данном задании WHERE запрещён, поэтому это НЕ подходит.
-- Единственный правильный путь — HAVING.
```

---

## Exercise 05 — Person's uniqueness

### Решение

```sql
SELECT DISTINCT
    prs.name                    -- 1. DISTINCT убирает дубликаты имён
                                --    Если человек сделал 5 заказов, его имя появится 1 раз
FROM person_order
    INNER JOIN person AS prs    -- 2. Соединяем заказы с таблицей person
        ON prs.id = person_order.person_id
ORDER BY 1;                     -- 3. Сортировка по имени (по возрастанию, ASC по умолчанию)
```

### Что спросят на защите

- **Почему не GROUP BY?** — Задание ЗАПРЕЩАЕТ GROUP BY. DISTINCT — альтернативный способ убрать дубликаты без группировки.
- **Чем DISTINCT отличается от GROUP BY?** — GROUP BY создаёт группы, к которым можно применить агрегатные функции. DISTINCT просто убирает повторяющиеся строки из результата. Если агрегация не нужна — DISTINCT проще.
- **Почему нельзя UNION?** — Задание запрещает любые операции с множествами (UNION, INTERSECT, EXCEPT).

### Альтернативное решение

```sql
-- Если бы GROUP BY был разрешён:
SELECT prs.name
FROM person_order
    INNER JOIN person AS prs ON prs.id = person_order.person_id
GROUP BY prs.name
ORDER BY 1;
-- GROUP BY автоматически убирает дубликаты, потому что
-- все одинаковые name схлопываются в одну группу.
-- Но в этом задании GROUP BY запрещён.
```

---

## Exercise 06 — Restaurant metrics

### Решение

```sql
SELECT
    pzr.name,                               -- 1. Название пиццерии
    COUNT(*) AS count_of_orders,             -- 2. Общее количество заказов
    ROUND(AVG(mn.price), 2) AS average_price,-- 3. Средняя цена заказанных пицц
                                             --    ROUND(..., 2) — округляем до 2 знаков после запятой
    MAX(mn.price) AS max_price,              -- 4. Максимальная цена
    MIN(mn.price) AS min_price               -- 5. Минимальная цена
FROM person_order
    INNER JOIN menu AS mn                    -- 6. Соединяем заказы с меню (для получения цены)
        ON person_order.menu_id = mn.id
    INNER JOIN pizzeria AS pzr               -- 7. Соединяем меню с пиццерией (для получения названия)
        ON mn.pizzeria_id = pzr.id
GROUP BY pzr.name                            -- 8. Группируем по названию пиццерии
ORDER BY 1;                                  -- 9. Сортировка по имени (алфавит)
```

### Что спросят на защите

- **Зачем ROUND?** — AVG возвращает число с большим количеством знаков после запятой (например, 933.333333...). ROUND(x, 2) обрезает до 2 знаков: 933.33.
- **Почему два JOIN?** — person_order содержит menu_id, не pizzeria_id. Цепочка: person_order → menu (для price) → pizzeria (для name).
- **COUNT(*) считает заказы или пиццы?** — Здесь это одно и то же: каждая строка в person_order — это один заказ одной пиццы.

### Альтернативное решение

```sql
-- Вариант: группировка по pizzeria_id (числовой ключ — теоретически эффективнее),
-- имя подтягиваем отдельно
SELECT
    pzr.name,
    COUNT(*) AS count_of_orders,
    ROUND(AVG(mn.price), 2) AS average_price,
    MAX(mn.price) AS max_price,
    MIN(mn.price) AS min_price
FROM person_order
    INNER JOIN menu AS mn ON person_order.menu_id = mn.id
    INNER JOIN pizzeria AS pzr ON mn.pizzeria_id = pzr.id
GROUP BY pzr.id, pzr.name
ORDER BY pzr.name;
-- GROUP BY pzr.id, pzr.name — группировка по первичному ключу + имени.
-- PostgreSQL позволяет: если id в GROUP BY, то name (зависящий от id) можно не добавлять,
-- но для ANSI SQL совместимости лучше указать оба.
```

---

## Exercise 07 — Average global rating

### Решение

```sql
SELECT
    ROUND(AVG(rating), 4) AS global_rating  -- 1. Средний рейтинг по ВСЕМ пиццериям
                                            --    ROUND(..., 4) — округляем до 4 знаков
                                            --    Нет GROUP BY — значит AVG считает по всей таблице
FROM pizzeria;                              -- 2. Источник — таблица пиццерий
```

### Что спросят на защите

- **Почему нет GROUP BY?** — Потому что нам нужно ОДНО число по всей таблице. GROUP BY разбил бы на группы, а мы хотим глобальное среднее.
- **Что будет, если rating содержит NULL?** — AVG игнорирует NULL значения. Если из 6 пиццерий у 2 рейтинг NULL, среднее посчитается по оставшимся 4.

### Альтернативное решение

```sql
-- Вариант: явное приведение типа для точности
SELECT
    ROUND(AVG(rating)::NUMERIC, 4) AS global_rating
FROM pizzeria;
-- ::NUMERIC — явное приведение к типу NUMERIC перед округлением.
-- В PostgreSQL rating уже numeric/float, так что результат тот же,
-- но приведение делает намерение явным.
```

---

## Exercise 08 — Find pizzeria's restaurant locations

### Решение

```sql
SELECT
    prs.address,                -- 1. Адрес человека (= город, где он живёт)
    pzr.name,                   -- 2. Название пиццерии
    COUNT(*) AS count_of_orders -- 3. Количество заказов из этой пиццерии по этому адресу
FROM person_order
    INNER JOIN menu AS mn       -- 4. person_order → menu (для pizzeria_id)
        ON person_order.menu_id = mn.id
    INNER JOIN pizzeria AS pzr  -- 5. menu → pizzeria (для name)
        ON mn.pizzeria_id = pzr.id
    INNER JOIN person AS prs    -- 6. person_order → person (для address)
        ON prs.id = person_order.person_id
GROUP BY prs.address, pzr.name -- 7. Группируем по ПАРЕ (адрес, пиццерия)
                                --    Каждая уникальная пара = отдельная строка в результате
ORDER BY 1, 2;                  -- 8. Сортировка: сначала по адресу, потом по названию
```

### Что спросят на защите

- **Почему GROUP BY по двум столбцам?** — Нам нужна статистика для каждой КОМБИНАЦИИ «город + пиццерия». GROUP BY address, name создаёт группу для каждой уникальной пары.
- **Допущение задания** — адрес считается городом. В реальности адрес мог бы быть полным (улица, дом), но здесь address хранит только название города.
- **Три JOIN — зачем?** — Три таблицы соединяют person_order с тремя справочниками: menu (цена/pizzeria_id), pizzeria (название), person (адрес). Без любого из них запрос не сработает.

### Альтернативное решение

```sql
-- Вариант с другим порядком JOIN (результат тот же, порядок JOIN не влияет на результат)
SELECT
    prs.address,
    pzr.name,
    COUNT(*) AS count_of_orders
FROM person AS prs
    INNER JOIN person_order AS po ON po.person_id = prs.id
    INNER JOIN menu AS mn ON po.menu_id = mn.id
    INNER JOIN pizzeria AS pzr ON mn.pizzeria_id = pzr.id
GROUP BY prs.address, pzr.name
ORDER BY 1, 2;
-- Здесь начинаем с person и присоединяем остальное.
-- Оптимизатор PostgreSQL сам выберет лучший план выполнения,
-- поэтому порядок записи JOIN не влияет на производительность.
```

---

## Exercise 09 — Explicit type transformation

### Решение

```sql
SELECT
    address,                                    -- 1. Адрес (город)
    ROUND(
        MAX(age) - MIN(age) / MAX(age::NUMERIC) -- 2. Формула: max_age - (min_age / max_age)
                                                --    age::NUMERIC — приведение типа (cast)
                                                --    из INTEGER в NUMERIC, чтобы деление
                                                --    было дробным, а не целочисленным.
                                                --    Без приведения: 10 / 30 = 0 (integer division)
                                                --    С приведением: 10 / 30.0 = 0.333...
    , 2) AS formula,                            -- 3. Округляем до 2 знаков
    ROUND(AVG(age), 2) AS average,              -- 4. Средний возраст по адресу, 2 знака
    ROUND(
        MAX(age) - MIN(age) / MAX(age::NUMERIC)
    , 2) > ROUND(AVG(age), 2)                   -- 5. Сравнение: formula > average
                                                --    Результат — булево значение: true или false
        AS comparison                           --    PostgreSQL может вернуть boolean напрямую
FROM person                                     -- 6. Источник — таблица person
GROUP BY address                                -- 7. Группируем по адресу
ORDER BY 1;                                     -- 8. Сортировка по адресу (алфавит)
```

### Что спросят на защите

- **Зачем `age::NUMERIC`?** — В PostgreSQL деление двух INTEGER даёт INTEGER (целочисленное деление, дробная часть отбрасывается). Например: 21 / 25 = 0 (а не 0.84). Приведение к NUMERIC (числовой тип с дробной частью) даёт правильный результат: 21 / 25.0 = 0.84. Это и есть «Explicit type transformation» из названия задания.
- **Приоритет операций в формуле** — `MAX(age) - MIN(age) / MAX(age::NUMERIC)` — деление выполняется РАНЬШЕ вычитания (как в математике). Поэтому это: `max - (min / max)`, а НЕ `(max - min) / max`.
- **Почему формула повторяется дважды?** — SQL не позволяет использовать алиас (formula) в том же SELECT, где он определён. Поэтому выражение приходится писать дважды: один раз для столбца formula, второй — для comparison.
- **Можно ли не повторять?** — Да, через подзапрос или CTE (Common Table Expression — обобщённое табличное выражение). См. альтернативу.

### Альтернативное решение

```sql
-- Вариант 1: CTE (WITH), чтобы не дублировать формулу
WITH stats AS (
    SELECT
        address,
        ROUND(MAX(age) - MIN(age) / MAX(age::NUMERIC), 2) AS formula,
        ROUND(AVG(age), 2) AS average
    FROM person
    GROUP BY address
)
SELECT
    address,
    formula,
    average,
    formula > average AS comparison
FROM stats
ORDER BY 1;
-- CTE (Common Table Expression) — именованный подзапрос.
-- WITH stats AS (...) создаёт временную «таблицу» stats,
-- к которой обращается основной SELECT.
-- Формула написана один раз, comparison ссылается на алиас.

-- Вариант 2: CAST вместо ::
SELECT
    address,
    ROUND(MAX(age) - MIN(age) / MAX(CAST(age AS NUMERIC)), 2) AS formula,
    ROUND(AVG(age), 2) AS average,
    ROUND(MAX(age) - MIN(age) / MAX(CAST(age AS NUMERIC)), 2)
        > ROUND(AVG(age), 2) AS comparison
FROM person
GROUP BY address
ORDER BY 1;
-- CAST(age AS NUMERIC) — стандарт ANSI SQL.
-- age::NUMERIC — синтаксис PostgreSQL (короче, но не переносим на другие СУБД).
-- Оба делают одно и то же.
```

---

## Шпаргалка: порядок выполнения SQL-запроса

Для защиты полезно знать, в каком порядке PostgreSQL реально обрабатывает запрос:

```
1. FROM / JOIN    — определяет таблицы и соединения
2. WHERE          — фильтрует строки ДО группировки
3. GROUP BY       — группирует строки
4. HAVING         — фильтрует группы ПОСЛЕ группировки
5. SELECT         — вычисляет выражения и алиасы
6. DISTINCT       — убирает дубликаты
7. ORDER BY       — сортирует результат
8. LIMIT / OFFSET — ограничивает количество строк
```

Это объясняет, почему:
- В WHERE нельзя использовать алиасы из SELECT (WHERE выполняется раньше)
- В HAVING можно использовать агрегатные функции (HAVING выполняется после GROUP BY)
- ORDER BY может ссылаться на алиасы (ORDER BY выполняется после SELECT)
