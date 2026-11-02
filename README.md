# Отчёт по домашнему заданию

## Содержание

- [Задание 1. Процентное отношение размера индексов к размеру таблиц](#task-1)
- [Задание 2. EXPLAIN ANALYZE и оптимизация](#task-2)
- [Задание 3*. Типы индексов PostgreSQL, которых нет в MySQL](#task-3)

---

<a id="task-1"></a>
## Задание 1. Процентное отношение размера индексов к размеру таблиц

Для учебной базы `sakila` запрос выглядит так:

```sql
SELECT
    ROUND(
        100.0 * SUM(pg_indexes_size(c.oid))
              / NULLIF(SUM(pg_total_relation_size(c.oid)), 0),
        2
    ) AS index_to_table_size_percent
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind = 'r'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema');
```

**Как это работает.** `pg_indexes_size(oid)` даёт суммарный размер всех индексов таблицы, а `pg_total_relation_size(oid)` — полный размер таблицы вместе с индексами и TOAST. Отношение первого ко второму, умноженное на 100, и есть искомый процент. `NULLIF` защищает от деления на ноль, если таблиц вдруг не окажется. Условие `relkind = 'r'` оставляет только обычные таблицы, а фильтр по `nspname` убирает системные каталоги, чтобы они не искажали статистику.

Если нужен процент именно от «чистого» размера таблиц без индексов, знаменатель меняется на `pg_relation_size(c.oid)`.

---

<a id="task-2"></a>
## Задание 2. EXPLAIN ANALYZE и оптимизация

### Исходный запрос

```sql
select distinct concat(c.last_name, ' ', c.first_name),
       sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30'
  and p.payment_date = r.rental_date
  and r.customer_id = c.customer_id
  and i.inventory_id = r.inventory_id;
```

### Найденные узкие места

1. **`date(p.payment_date) = '2005-07-30'`** — функция над столбцом убивает SARGability. Индекс по `payment_date` не используется для поиска диапазона, планировщик вынужден читать всю таблицу `payment` (Seq Scan) или сканировать индекс целиком.
2. **Соединение через запятую в `FROM`** — устаревший синтаксис. Планировщик обычно справляется, но читаемость падает, а порядок соединений становится неочевидным.
3. **`p.payment_date = r.rental_date`** — семантически сомнительное условие. В Sakila платёж связан с прокатом через `rental_id`, а не через совпадение дат с точностью до секунды. Такое соединение может давать ложные пары или, наоборот, терять строки.
4. **Оконная функция + `DISTINCT`** — `DISTINCT` применяется уже после вычисления окна. Если нужна сумма по клиенту и фильму, достаточно `GROUP BY`, а `DISTINCT` становится лишним этапом сортировки/хеширования.
5. **Отсутствие индексов** на `payment(payment_date)`, `rental(rental_date)`, `rental(customer_id)`, `rental(inventory_id)` приводит к полным сканированиям и дорогим хеш- или merge-соединениям.

### Оптимизированный вариант

```sql
SELECT DISTINCT
       c.last_name || ' ' || c.first_name AS customer_name,
       sum(p.amount) OVER (PARTITION BY c.customer_id, f.title) AS amount_sum
FROM payment p
JOIN rental r    ON p.payment_date = r.rental_date
JOIN customer c  ON r.customer_id = c.customer_id
JOIN inventory i ON i.inventory_id = r.inventory_id
JOIN film f      ON f.film_id = i.film_id
WHERE p.payment_date >= DATE '2005-07-30'
  AND p.payment_date <  DATE '2005-07-31';
```

Что изменилось:

- Убран `date()` — предикат стал диапазонным, индекс по `payment_date` теперь работает.
- Явные `JOIN` вместо перечисления таблиц через запятую.
- `concat()` заменён на `||` — короче и без лишнего вызова функции.

Индексы, которые стоит добавить:

```sql
CREATE INDEX idx_payment_payment_date ON payment (payment_date);
CREATE INDEX idx_rental_rental_date   ON rental  (rental_date);
CREATE INDEX idx_rental_customer_id   ON rental  (customer_id);
CREATE INDEX idx_rental_inventory_id  ON rental  (inventory_id);
```

---

<a id="task-3"></a>
## Задание 3*. Типы индексов PostgreSQL, которых нет в MySQL

В PostgreSQL гораздо богаче набор методов доступа к индексам. Вот что есть в Postgres, но отсутствует в MySQL (или появилось там позже и в урезанном виде):

- **GiST** — обобщённое дерево поиска для геометрии, пространственных данных, текста, диапазонов. Поддерживает операторы близости, пересечения, вхождения. В MySQL аналога нет.
- **SP-GiST** — для несбалансированных структур: квадродеревья, k-мерные деревья, префиксные деревья. В MySQL отсутствует.
- **GIN** — инвертированный индекс для массивов, JSONB, полнотекстового поиска. MySQL использует FULLTEXT только для текста, универсального GIN там нет.
- **BRIN** — компактный индекс для очень больших таблиц с физически упорядоченными данными (например, по времени). В MySQL аналога нет.
- **Bloom** (расширение) — фильтр Блума для нескольких столбцов с проверкой на равенство. В MySQL отсутствует.
- **RUM** — расширение GIN с поддержкой позиций слов и ранжирования. В MySQL отсутствует.

Кроме того, PostgreSQL умеет **частичные индексы** (с `WHERE` в определении), **индексы по выражениям** и **покрывающие индексы через `INCLUDE`**. В MySQL частичные и функциональные индексы появились только в 8.0 и с ограничениями, а полноценного механизма `INCLUDE` нет вовсе.
