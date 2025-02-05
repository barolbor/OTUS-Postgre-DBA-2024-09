# Домашнее задание
## Работа с индексами
 
 Скачиваем и устанавливаем демонстрационную базу данных авиаперевозки по России.
```bash
curl https://edu.postgrespro.ru/demo-medium-20161013.zip -o demo-medium.zip
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 61.5M  100 61.5M    0     0  5383k      0  0:00:11  0:00:11 --:--:-- 6038k

unzip demo-medium.zip
Archive:  demo-medium.zip
  inflating: demo_medium.sql

sudo -u postgres psql postgres < ~/demo_medium.sql
```

Посмотрим размер таблиц и количество строк

```sql
ALTER DATABASE demo SET search_path TO bookings, pg_catalog;

 SELECT
    t.tablename AS table_name,
    c.reltuples AS num_rows,
    pg_size_pretty(pg_total_relation_size(c.oid))AS total_size,
    pg_size_pretty(pg_table_size(c.oid)) AS data_size,
    pg_size_pretty(pg_indexes_size(c.oid)) AS index_size
FROM
    pg_catalog.pg_tables AS t
    LEFT JOIN pg_catalog.pg_class AS c ON c.relname = t.tablename
WHERE
    t.schemaname ='bookings';

   table_name    |   num_rows   | total_size | data_size | index_size
-----------------+--------------+------------+-----------+------------
 ticket_flights  | 2.360335e+06 | 246 MB     | 154 MB    | 91 MB
 boarding_passes | 1.894295e+06 | 264 MB     | 109 MB    | 155 MB
 aircrafts       |            9 | 32 kB      | 16 kB     | 16 kB
 flights         |        65664 | 10192 kB   | 6688 kB   | 3504 kB
 airports        |          104 | 72 kB      | 56 kB     | 16 kB
 seats           |         1339 | 144 kB     | 96 kB     | 48 kB
 tickets         |       829071 | 134 MB     | 109 MB    | 25 MB
 bookings        |       593433 | 43 MB      | 30 MB     | 13 MB
(8 rows)
```

Посмотрим "внутренне устройство" таблицы boarding_passes

```sql
\d boarding_passes
                  Table "bookings.boarding_passes"
   Column    |         Type         | Collation | Nullable | Default
-------------+----------------------+-----------+----------+---------
 ticket_no   | character(13)        |           | not null |
 flight_id   | integer              |           | not null |
 boarding_no | integer              |           | not null |
 seat_no     | character varying(4) |           | not null |
Indexes:
    "boarding_passes_pkey" PRIMARY KEY, btree (ticket_no, flight_id)
    "boarding_passes_flight_id_boarding_no_key" UNIQUE CONSTRAINT, btree (flight_id, boarding_no)
    "boarding_passes_flight_id_seat_no_key" UNIQUE CONSTRAINT, btree (flight_id, seat_no)
Foreign-key constraints:
    "boarding_passes_ticket_no_fkey" FOREIGN KEY (ticket_no, flight_id) REFERENCES ticket_flights(ticket_no, flight_id)

-- Удалим ограничение
ALTER TABLE boarding_passes DROP CONSTRAINT boarding_passes_flight_id_boarding_no_key;

-- Отключим распараллеливание запросов
SET max_parallel_workers_per_gather = 0;

SHOW max_parallel_workers_per_gather;
 max_parallel_workers_per_gather
---------------------------------
 0
(1 row)
```

Найдем все посадочные с номером 154 и посмотрим на план выполнения запроса

```sql
EXPLAIN ANALYZE
SELECT boarding_no FROM boarding_passes bp WHERE bp.boarding_no = 154;
                                                      QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------
 Seq Scan on boarding_passes bp  (cost=0.00..37630.69 rows=1207 width=4) (actual time=0.072..90.393 rows=1797 loops=1)
   Filter: (boarding_no = 154)
   Rows Removed by Filter: 1892498
 Planning Time: 0.168 ms
 Execution Time: 90.472 ms
(5 rows)
```

Запрос выполнялся 90.472мс, при этом последовательно сканировалась все строки таблицы и к каждой применялся фильтр на искомое значение, найдено 1 797 подходящих строк, 1 892 498 строк не удовлетворило условию фильтра.

Запрос на все поля

```sql
EXPLAIN ANALYZE
SELECT * FROM boarding_passes bp WHERE bp.boarding_no = 154;
                                                       QUERY PLAN
------------------------------------------------------------------------------------------------------------------------
 Seq Scan on boarding_passes bp  (cost=0.00..37630.69 rows=1207 width=25) (actual time=0.094..89.232 rows=1797 loops=1)
   Filter: (boarding_no = 154)
   Rows Removed by Filter: 1892498
 Planning Time: 0.121 ms
 Execution Time: 89.317 ms
(5 rows)
```

Создадим индекс для оптимизации выполнения запроса и повторим запрос

```sql
--Создадим индекс
CREATE INDEX boarding_passes_boarding_no ON boarding_passes(boarding_no);
CREATE INDEX

-- Добавим комментарий
COMMENT ON INDEX  boarding_passes_boarding_no IS 'Индекс для поиска по номеру посадочного талона';

-- Посмотрим описание индекса
\di+ boarding_passes_boarding_no
List of relations
-[ RECORD 1 ]-+-----------------------------------------------
Schema        | bookings
Name          | boarding_passes_boarding_no
Type          | index
Owner         | postgres
Table         | boarding_passes
Persistence   | permanent
Access method | btree
Size          | 13 MB
Description   | Индекс для поиска по номеру посадочного талона

-- Выполним запрос
SELECT pg_size_pretty(pg_relation_size('boarding_passes_boarding_no')) AS index_size;
 index_size
------------
 13 MB
(1 row)

EXPLAIN ANALYZE
SELECT boarding_no FROM boarding_passes bp WHERE bp.boarding_no = 154;
                                                                         QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------------------------
 Index Only Scan using boarding_passes_boarding_no on boarding_passes bp  (cost=0.43..37.54 rows=1207 width=4) (actual time=0.048..0.155 rows=1797 loops=1)
   Index Cond: (boarding_no = 154)
   Heap Fetches: 0
 Planning Time: 0.350 ms
 Execution Time: 0.241 ms
(5 rows)
```

Время выполнения запроса сократилось с 90.472мс до 0.241мс т.е. в 375 раз. Т.к. мы выбираем только одно поле boarding_no, индекс "не ходит" в таблицу, а значения выбираются прямо из индекса.

Запрос на все поля

```sql
EXPLAIN ANALYZE
SELECT * FROM boarding_passes bp WHERE bp.boarding_no = 154;
                                                                QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on boarding_passes bp  (cost=17.78..3661.32 rows=1207 width=25) (actual time=0.385..3.264 rows=1797 loops=1)
   Recheck Cond: (boarding_no = 154)
   Heap Blocks: exact=1797
   ->  Bitmap Index Scan on boarding_passes_boarding_no  (cost=0.00..17.48 rows=1207 width=0) (actual time=0.178..0.178 rows=1797 loops=1)
         Index Cond: (boarding_no = 154)
 Planning Time: 0.082 ms
 Execution Time: 3.329 ms
(7 rows)
```
При выборке на все поля, планировщик создает битовую карту на основе индекса фильтруя записи по заданному условию, после чего читает страницы, используя полученную битовую карту и перепроверяет условие выборки. Время выполнения запроса сократилось с 90.472мс до 3.329мс (в 27 раз), но по сравнению с выборкой только индексного поля увеличилось с 0.241мс до 3.329мс (в 13 раз).

Удалим индекс boarding_passes_boarding_no и пересоздадим его на часть таблицы с посадочными номерами до 40 и от 40 включительно (972000 и 922295 записи соответственно).

```sql
DROP INDEX boarding_passes_boarding_no;
DROP INDEX

CREATE INDEX boarding_passes_boarding_no ON boarding_passes(boarding_no) WHERE boarding_no < 40;
CREATE INDEX

COMMENT ON INDEX  boarding_passes_boarding_no IS 'Индекс для поиска по номеров посадочных талонов меньше 40';
COMMENT

\di+ boarding_passes_boarding_no
List of relations
-[ RECORD 1 ]-+----------------------------------------------------------
Schema        | bookings
Name          | boarding_passes_boarding_no
Type          | index
Owner         | postgres
Table         | boarding_passes
Persistence   | permanent
Access method | btree
Size          | 6624 kB
Description   | Индекс для поиска по номеров посадочных талонов меньше 40

SELECT pg_size_pretty(pg_relation_size('boarding_passes_boarding_no')) AS index_size;
 index_size
------------
 6624 kB
(1 row)

EXPLAIN ANALYZE
SELECT boarding_no FROM boarding_passes bp WHERE bp.boarding_no = 33;
                                                                          QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 Index Only Scan using boarding_passes_boarding_no on boarding_passes bp  (cost=0.42..516.52 rows=20585 width=4) (actual time=0.021..0.866 rows=19804 loops=1)
   Index Cond: (boarding_no = 33)
   Heap Fetches: 1
 Planning Time: 0.218 ms
 Execution Time: 1.336 ms
(5 rows)

EXPLAIN ANALYZE
SELECT boarding_no FROM boarding_passes bp WHERE bp.boarding_no = 154;
                                                      QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------
 Seq Scan on boarding_passes bp  (cost=0.00..37630.69 rows=1207 width=4) (actual time=0.293..87.770 rows=1797 loops=1)
   Filter: (boarding_no = 154)
   Rows Removed by Filter: 1892498
 Planning Time: 0.360 ms
 Execution Time: 87.881 ms
(5 rows)
```

Мы сделали два запроса: 1ый с номером посадочного 33, который покрывает индекс, а 2ой - с номером 154, он в индекс не входит. Время выполнения запросов 1.336мс против 87.881мс, разница в 65 раз.

Индекс на часть таблицы имен размер 6624 kB = 6.5MB что в два раза меньше (мы ведь подобрали значения примерно, чтобы в индекс вошло пол таблицы) меньше, чем на всю таблицу 13MB.

Удалим индекс. Создадим индекс на несколько полей, аналогично ограничению, удаленному в самом начале ДЗ.

```sql
CREATE INDEX boarding_passes_flight_id_boarding_no ON boarding_passes(flight_id, boarding_no);
CREATE INDEX

COMMENT ON INDEX  boarding_passes_flight_id_boarding_no IS 'Индекс для поиска по номеру рейса и посадочного талона';
COMMENT

demo=# \di+ boarding_passes_flight_id_boarding_no
List of relations
-[ RECORD 1 ]-+-------------------------------------------------------
Schema        | bookings
Name          | boarding_passes_flight_id_boarding_no
Type          | index
Owner         | postgres
Table         | boarding_passes
Persistence   | permanent
Access method | btree
Size          | 41 MB
Description   | Индекс для поиска по номеру рейса и посадочного талона
```

Запрос с прямым порядком следования параметров (как в индексе)

```sql
EXPLAIN ANALYZE
SELECT * FROM boarding_passes WHERE flight_id=677 AND boarding_no = 154;
                                                                       QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using boarding_passes_flight_id_boarding_no on boarding_passes  (cost=0.43..8.45 rows=1 width=25) (actual time=0.053..0.055 rows=1 loops=1)
   Index Cond: ((flight_id = 677) AND (boarding_no = 154))
 Planning Time: 0.279 ms
 Execution Time: 0.083 ms
(4 rows)
```

Запрос с обратным порядком следования параметров

```sql
demo=# EXPLAIN ANALYZE
SELECT * FROM boarding_passes WHERE boarding_no = 154 AND flight_id=677;
                                                                       QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using boarding_passes_flight_id_boarding_no on boarding_passes  (cost=0.43..8.45 rows=1 width=25) (actual time=0.058..0.060 rows=1 loops=1)
   Index Cond: ((flight_id = 677) AND (boarding_no = 154))
 Planning Time: 0.156 ms
 Execution Time: 0.094 ms
(4 rows)
```

А ничего не изменилось, оптимизатор ставит условия в "правильном" порядке и применяет индекс.

Сделаем запрос с условием только на одно - первое поле в индексе
```sql
EXPLAIN ANALYZE
SELECT * FROM boarding_passes WHERE flight_id=677;
                                                                   QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on boarding_passes  (cost=5.06..311.55 rows=81 width=25) (actual time=0.123..0.149 rows=166 loops=1)
   Recheck Cond: (flight_id = 677)
   Heap Blocks: exact=2
   ->  Bitmap Index Scan on boarding_passes_flight_id_boarding_no  (cost=0.00..5.04 rows=81 width=0) (actual time=0.103..0.103 rows=166 loops=1)
         Index Cond: (flight_id = 677)
 Planning Time: 0.138 ms
 Execution Time: 0.183 ms
(7 rows)
```

Оптимизатор по-прежнему использует индекс. 

Сделаем запрос с условием только на одно - второе поле в индексе

```sql
EXPLAIN ANALYZE
SELECT * FROM boarding_passes WHERE boarding_no = 154;
                                                     QUERY PLAN
---------------------------------------------------------------------------------------------------------------------
 Seq Scan on boarding_passes  (cost=0.00..37630.69 rows=1207 width=25) (actual time=0.109..91.410 rows=1797 loops=1)
   Filter: (boarding_no = 154)
   Rows Removed by Filter: 1892498
 Planning Time: 0.205 ms
 Execution Time: 91.483 ms
(5 rows)
```

Оптимизатор уже не использует индекс, а последовательно сканирует всю таблицу.

Чуть модифицируем наш запрос, будем выбирать не все поля, а только одно, что есть в индексе.

```sql
EXPLAIN ANALYZE
SELECT boarding_no FROM boarding_passes WHERE boarding_no = 154;
                                                                              QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Index Only Scan using boarding_passes_flight_id_boarding_no on boarding_passes  (cost=0.43..35015.71 rows=1207 width=4) (actual time=0.579..30.876 rows=1797 loops=1)
   Index Cond: (boarding_no = 154)
   Heap Fetches: 0
 Planning Time: 0.076 ms
 Execution Time: 30.952 ms
(5 rows)
```

Оптимизатор использовал индекс для поиска только по второму параметру индекса, время выполнения запроса по-прежнему велико, хотя и в 3 раза меньше, чем при выборке по всем полям, когда применялось последовательное сканирование всей таблицы.

 Создадим индекс для полнотекстового поиска

```sql
CREATE INDEX tickets_passenger_name ON tickets USING gin(to_tsvector('english', passenger_name));
CREATE INDEX

COMMENT ON INDEX  tickets_passenger_name IS 'Индекс для полнотекстового поиска по имени пассажира';
COMMENT

demo=# \di+ tickets_passenger_name
List of relations
-[ RECORD 1 ]-+-----------------------------------------------------
Schema        | bookings
Name          | tickets_passenger_name
Type          | index
Owner         | postgres
Table         | tickets
Persistence   | permanent
Access method | gin
Size          | 5376 kB
Description   | Индекс для полнотекстового поиска по имени пассажира
```

Выполним поиск пассажиров с именем oleg или фамилией baranov

```sql
EXPLAIN ANALYZE
SELECT * FROM tickets WHERE to_tsvector('english', passenger_name) @@ to_tsquery('oleg | baranov');
                                                              QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on tickets  (cost=77.52..16874.24 rows=8270 width=104) (actual time=1.924..13.355 rows=10975 loops=1)
   Recheck Cond: (to_tsvector('english'::regconfig, passenger_name) @@ to_tsquery('oleg | baranov'::text))
   Heap Blocks: exact=7651
   ->  Bitmap Index Scan on tickets_passenger_name  (cost=0.00..75.45 rows=8270 width=0) (actual time=1.224..1.225 rows=10975 loops=1)
         Index Cond: (to_tsvector('english'::regconfig, passenger_name) @@ to_tsquery('oleg | baranov'::text))
 Planning Time: 0.143 ms
 Execution Time: 13.626 ms
(7 rows)

SELECT * FROM tickets WHERE to_tsvector('english', passenger_name) @@ to_tsquery('oleg | baranov') LIMIT 10;
   ticket_no   | book_ref | passenger_id |  passenger_name   |                                contact_data
---------------+----------+--------------+-------------------+-----------------------------------------------------------------------------
 0005432000971 | A8ADCE   | 6780 391925  | ALEKSANDR BARANOV | {"phone": "+70299141945"}
 0005432001026 | F5F784   | 2765 289085  | OLEG KUZNECOV     | {"email": "oleg.kuznecov-10101967@postgrespro.ru", "phone": "+70154552799"}
 0005432001063 | ACCA92   | 7650 555895  | ROMAN BARANOV     | {"email": "roman_baranov-1973@postgrespro.ru", "phone": "+70002968931"}
 0005432002091 | 04E1A6   | 6285 568179  | OLEG NOVIKOV      | {"email": "olegnovikov09081960@postgrespro.ru", "phone": "+70118631206"}
 0005432003511 | 6409CA   | 4263 146430  | YURIY BARANOV     | {"email": "baranov_yuriy_07091966@postgrespro.ru", "phone": "+70939047627"}
 0005432003546 | 186DD8   | 7001 204079  | OLEG ROMANOV      | {"email": "oleg-romanov-041985@postgrespro.ru", "phone": "+70102814628"}
 0005432003593 | B2EBF7   | 0294 173850  | ANTON BARANOV     | {"email": "a.baranov.1972@postgrespro.ru", "phone": "+70205764897"}
 0005432003673 | C7D1C1   | 0750 738975  | VADIM BARANOV     | {"phone": "+70718887901"}
 0005432003787 | 2CF94D   | 8280 468376  | OLEG MAKAROV      | {"phone": "+70480427913"}
 0005432005154 | C36130   | 5817 090451  | OLEG KISELEV      | {"phone": "+70127405954"}
(10 rows)
```

И тот же запрос, но без индекса

```sql
 EXPLAIN ANALYZE
SELECT * FROM tickets WHERE to_tsvector('english', passenger_name) @@ to_tsquery('oleg | baranov');
                                                    QUERY PLAN
------------------------------------------------------------------------------------------------------------------
 Seq Scan on tickets  (cost=0.00..438818.89 rows=8270 width=104) (actual time=7.289..1774.678 rows=10975 loops=1)
   Filter: (to_tsvector('english'::regconfig, passenger_name) @@ to_tsquery('oleg | baranov'::text))
   Rows Removed by Filter: 818096
 Planning Time: 0.166 ms
 JIT:
   Functions: 2
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 0.249 ms, Inlining 0.000 ms, Optimization 0.253 ms, Emission 6.670 ms, Total 7.172 ms
 Execution Time: 1805.114 ms
(9 rows)
```

Без индекса 1805.114мс., а с индексом 13.626, разница в 132 раза.

Индекс штука хороша, но его нужно использовать с умом, дабы не получить просадку производительности при DML операциях.