# Домашнее задание
## Настройка autovacuum с учетом особеностей производительности
 
Разворачиваем новый кластер PostgreSQL 16 на Ubuntu 22.04 LTS в WMware. Виртуальная машина будет состоять из 1 го процессора с 2 ядрами, 4Гб ОЗУ и 40Гб SSD диска.

Создадим БД для тестов и выполним инициализацию таблиц.

```bash
# В консоли psql
CREATE DATABASE testvacuum;
CREATE DATABASE

# В консоли
sudo -u postgres pgbench -i testvacuum
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.04 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.16 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.08 s, vacuum 0.03 s, primary keys 0.04 s).

# В консоли psql посмотрим размер БД после инициализации
\l+
...
-[ RECORD 4 ]-----+-------------------------------------------
Name              | testvacuum
Owner             | postgres
Encoding          | UTF8
Locale Provider   | libc
Collate           | en_US.UTF-8
Ctype             | en_US.UTF-8
ICU Locale        |
ICU Rules         |
Access privileges |
Size              | 23 MB
Tablespace        | pg_default
Description       |
```

Запустим pgbench, кластер с настройками по умолчанию

```bash
sudo -u postgres pgbench -P 6 -c 8 -j 4 -T 60 testvacuum
pgbench (16.6 (Ubuntu 16.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 1762.8 tps, lat 4.526 ms stddev 2.753, 0 failed
progress: 12.0 s, 1772.2 tps, lat 4.514 ms stddev 2.637, 0 failed
progress: 18.0 s, 1779.0 tps, lat 4.495 ms stddev 2.522, 0 failed
progress: 24.0 s, 1763.6 tps, lat 4.536 ms stddev 2.762, 0 failed
progress: 30.0 s, 1780.0 tps, lat 4.493 ms stddev 2.588, 0 failed
progress: 36.0 s, 1718.7 tps, lat 4.654 ms stddev 2.743, 0 failed
progress: 42.0 s, 1760.4 tps, lat 4.543 ms stddev 2.824, 0 failed
progress: 48.0 s, 1758.2 tps, lat 4.548 ms stddev 2.709, 0 failed
progress: 54.0 s, 1770.5 tps, lat 4.518 ms stddev 2.643, 0 failed
progress: 60.0 s, 1773.7 tps, lat 4.511 ms stddev 2.922, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 105842
number of failed transactions: 0 (0.000%)
latency average = 4.534 ms
latency stddev = 2.713 ms
initial connection time = 11.807 ms
tps = 1763.821351 (without initial connection time)

# В консоли psql посмотрим размер БД после 1го теста
-[ RECORD 4 ]-----+-------------------------------------------
Name              | testvacuum
Owner             | postgres
Encoding          | UTF8
Locale Provider   | libc
Collate           | en_US.UTF-8
Ctype             | en_US.UTF-8
ICU Locale        |
ICU Rules         |
Access privileges |
Size              | 28 MB
Tablespace        | pg_default
Description       |
```

Применим к кластеру следующие настройки из задания в ДЗ:

```bash
# DB Version: 11
# OS Type: linux
# DB Type: dw
# Total Memory (RAM): 4 GB
# CPUs num: 1
# Data Storage: hdd

sudo sh -c 'echo "
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 65536kB
min_wal_size = 4GB
max_wal_size = 16GB
" >> /etc/postgresql/16/main/postgresql.conf'

sudo systemctl restart postgresql@16-main
```

Проверим, что настройки применились

```sql
select * from pg_file_settings order by sourceline desc;

               sourcefile                | sourceline | seqno |             name             |                setting                 | applied | error
-----------------------------------------+------------+-------+------------------------------+----------------------------------------+---------+-------
 /etc/postgresql/16/main/postgresql.conf |        837 |    37 | max_wal_size                 | 16GB                                   | t       |
 /etc/postgresql/16/main/postgresql.conf |        836 |    36 | min_wal_size                 | 4GB                                    | t       |
 /etc/postgresql/16/main/postgresql.conf |        835 |    35 | work_mem                     | 65536kB                                | t       |
 /etc/postgresql/16/main/postgresql.conf |        834 |    34 | effective_io_concurrency     | 2                                      | t       |
 /etc/postgresql/16/main/postgresql.conf |        833 |    33 | random_page_cost             | 4                                      | t       |
 /etc/postgresql/16/main/postgresql.conf |        832 |    32 | default_statistics_target    | 500                                    | t       |
 /etc/postgresql/16/main/postgresql.conf |        831 |    31 | wal_buffers                  | 16MB                                   | t       |
 /etc/postgresql/16/main/postgresql.conf |        830 |    30 | checkpoint_completion_target | 0.9                                    | t       |
 /etc/postgresql/16/main/postgresql.conf |        829 |    29 | maintenance_work_mem         | 512MB                                  | t       |
 /etc/postgresql/16/main/postgresql.conf |        828 |    28 | effective_cache_size         | 3GB                                    | t       |
 /etc/postgresql/16/main/postgresql.conf |        827 |    27 | shared_buffers               | 1GB                                    | t       |
 /etc/postgresql/16/main/postgresql.conf |        826 |    26 | max_connections              | 40                                     | t       |
 ...
```

Выполним тест заново

```bash
sudo -u postgres pgbench -P 6 -c 8 -j 4 -T 60 testvacuum
pgbench (16.6 (Ubuntu 16.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 1704.2 tps, lat 4.682 ms stddev 3.022, 0 failed
progress: 12.0 s, 1760.2 tps, lat 4.544 ms stddev 2.579, 0 failed
progress: 18.0 s, 1807.1 tps, lat 4.427 ms stddev 2.579, 0 failed
progress: 24.0 s, 1756.5 tps, lat 4.552 ms stddev 2.735, 0 failed
progress: 30.0 s, 1761.0 tps, lat 4.543 ms stddev 2.819, 0 failed
progress: 36.0 s, 1743.9 tps, lat 4.585 ms stddev 2.763, 0 failed
progress: 42.0 s, 1773.8 tps, lat 4.510 ms stddev 2.652, 0 failed
progress: 48.0 s, 1768.2 tps, lat 4.524 ms stddev 2.648, 0 failed
progress: 54.0 s, 1795.9 tps, lat 4.454 ms stddev 2.652, 0 failed
progress: 60.0 s, 1774.4 tps, lat 4.507 ms stddev 2.659, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 105878
number of failed transactions: 0 (0.000%)
latency average = 4.532 ms
latency stddev = 2.713 ms
initial connection time = 12.033 ms
tps = 1764.587101 (without initial connection time)

# В консоли psql посмотрим размер БД после 2го теста
-[ RECORD 4 ]-----+-------------------------------------------
Name              | testvacuum
Owner             | postgres
Encoding          | UTF8
Locale Provider   | libc
Collate           | en_US.UTF-8
Ctype             | en_US.UTF-8
ICU Locale        |
ICU Rules         |
Access privileges |
Size              | 28 MB
Tablespace        | pg_default
Description       |
```

Т.к. в настройках из ДЗ значения параметров random_page_cost и effective_io_concurrency соответствуют рекомендованным для HDD, а у нас ВМ на SSD, да и версия PostgeSQL отличается, сделаем расчет на калькуляторе PGTune (судя по всему, ДЗ`шные настройки из него) для более входных параметров и протестируем с ними.

```bash
# DB Version: 16
# OS Type: linux
# DB Type: dw
# Total Memory (RAM): 4 GB
# CPUs num: 2
# Connections num: 40
# Data Storage: ssd

 sudo -u postgres pgbench -P 6 -c 8 -j 4 -T 60 testvacuum
pgbench (16.6 (Ubuntu 16.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 1767.0 tps, lat 4.515 ms stddev 2.805, 0 failed
progress: 12.0 s, 1825.3 tps, lat 4.382 ms stddev 2.645, 0 failed
progress: 18.0 s, 1761.9 tps, lat 4.539 ms stddev 2.907, 0 failed
progress: 24.0 s, 1792.1 tps, lat 4.463 ms stddev 2.731, 0 failed
progress: 30.0 s, 1779.7 tps, lat 4.494 ms stddev 2.800, 0 failed
progress: 36.0 s, 1758.5 tps, lat 4.548 ms stddev 2.750, 0 failed
progress: 42.0 s, 1743.4 tps, lat 4.588 ms stddev 2.764, 0 failed
progress: 48.0 s, 1839.6 tps, lat 4.349 ms stddev 2.475, 0 failed
progress: 54.0 s, 1713.8 tps, lat 4.667 ms stddev 2.811, 0 failed
progress: 60.0 s, 1685.0 tps, lat 4.747 ms stddev 3.153, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 106004
number of failed transactions: 0 (0.000%)
latency average = 4.527 ms
latency stddev = 2.790 ms
initial connection time = 14.200 ms
tps = 1766.454705 (without initial connection time)
```

Создадим сводную таблицу настроек и результатов тестирования.

|||||
|-|-|-|-|
|параметр|default|PG 11, НВВ|PG 16, SSD|
|max_connections|100|40|40|
|shared_buffers|128MB|1GB|1GB|
|effective_cache_size|3GB|3GB|3GB|
|maintenance_work_mem|64MB|512MB|512MB|
|checkpoint_completion_target|0.9|0.9|0.9|
|wal_buffers|4MB|16MB|16MB|
|default_statistics_target|100|500|500|
|random_page_cost|4|4|1.1|
|effective_io_concurrency|1|2|200|
|work_mem|4MB|65536kB|65536kB|
|min_wal_size|80MB|4GB|4GB|
|max_wal_size|1GB|16GB|16GB|
|результаты||||
|tps|1763|1764|1766|

По сравнению с настройками по умолчанию, в тестовых увеличен hared_buffers в 8 раз, что должно было сказаться на производительности, но т.к. у нас тестовая база размером 28GB, что в 4 раза меньше буферного кэша, то мы не увидели никакого прироста производительности.

Так же в 8 раз увеличен размер maintenance_work_mem, но эта память нужна для обслуживающих процессов, типа VACUUM, она выделяется сразу в полном объеме, и если в процессе тестирования к нам не приходит autovacuum, то и его влияния мы не увидим.


Создадим таблицу с текстовым полем и заполним ее сгенерированными данным в размере 1млн строк.

```sql
CREATE TABLE tv(id serial, str char(100));
CREATE TABLE

INSERT INTO tv(str) SELECT 'AAA' FROM generate_series(1, 1000000);
INSERT 0 1000000

# Размер файла с таблицей
SELECT pg_size_pretty(pg_total_relation_size('tv'));
 pg_size_pretty
----------------
 135 MB
(1 row)
```

Создадим процедуру для обновления всех строк в тестовой таблице заданное число раз

```sql
CREATE OR REPLACE FUNCTION update_tv_tbl(cnt INT)
RETURNS VOID
LANGUAGE plpgsql
AS $$
DECLARE
    i INT := 1;
BEGIN
    FOR i IN 1..cnt LOOP
        UPDATE tv  tv SET str=str||' '||chr(65+i);
        RAISE NOTICE 'Шаг цикла №: %', i;
    END LOOP;
END;
$$;
CREATE FUNCTION
```

Создадим представление для просмотра информации по автовакууму и мертвых строках.
```sql
CREATE VIEW tv_vac as
    SELECT 
        relname,
        n_live_tup,
        n_dead_tup,
        trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%",
        last_autovacuum
    FROM 
        pg_stat_user_tables
    WHERE 
        relname = 'tv';
```

Пять раз обновим все строки и посмотрим количество мертвых строк в таблице, а так же когда последний раз приходил автовакуум и подождем прихода автовакуума.

```sql
SELECT update_tv_tbl(5);
NOTICE:  Шаг цикла №: 1
NOTICE:  Шаг цикла №: 2
NOTICE:  Шаг цикла №: 3
NOTICE:  Шаг цикла №: 4
NOTICE:  Шаг цикла №: 5
 update_tv_tbl
---------------

(1 row)

SELECT * FROM tv_vac;
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 tv      |    1000000 |    5000000 |    499 | 2025-01-28 15:50:05.284646+00
(1 row)

SELECT * FROM tv_vac;
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 tv      |    1000000 |          0 |      0 | 2025-01-28 15:54:08.774861+00
(1 row)
```

Пять раз обновим все строки и посмотрим размер файла с таблицей

```sql
SELECT update_tv_tbl(5);
NOTICE:  Шаг цикла №: 1
NOTICE:  Шаг цикла №: 2
NOTICE:  Шаг цикла №: 3
NOTICE:  Шаг цикла №: 4
NOTICE:  Шаг цикла №: 5
 update_tv_tbl
---------------

(1 row)

SELECT pg_size_pretty(pg_total_relation_size('tv'));
 pg_size_pretty
----------------
 808 MB
(1 row)
```

Отключим автовакуум на нашей тестовой таблице tv, 10 раз обновим все строки и посмотрим размер файла с таблицей

```sql
ALTER TABLE tv SET (autovacuum_enabled = false);
ALTER TABLE

ALTER TABLE tv SET (autovacuum_enabled = false);
ALTER TABLE
testvacuum=# SELECT update_tv_tbl(10);
NOTICE:  Шаг цикла №: 1
NOTICE:  Шаг цикла №: 2
NOTICE:  Шаг цикла №: 3
NOTICE:  Шаг цикла №: 4
NOTICE:  Шаг цикла №: 5
NOTICE:  Шаг цикла №: 6
NOTICE:  Шаг цикла №: 7
NOTICE:  Шаг цикла №: 8
NOTICE:  Шаг цикла №: 9
NOTICE:  Шаг цикла №: 10
 update_tv_tbl
---------------

(1 row)

SELECT pg_size_pretty(pg_total_relation_size('tv'));
 pg_size_pretty
----------------
 1482 MB
(1 row)
```

При выполнении команды UPDATE, PostgreSQL создает новую версию строки, при этом, старая версия строки не удаляется, т.к. может использоваться в других транзакциях, а помечается как мертвая. Размер таблицы увеличивается.

Отключение автовакуума позволяет убрать нагрузку с дисков, но как только его выключаем, таблицы и индексы перестают чиститься. И они начинают распухать. В них появляются мертвые строки. Следствием этого является то, что область shared buffers, где размещаются все оперативные данные для работы базы данных (это таблицы, индексы), начинает использоваться неэффективно. В ней находятся те самые мусорные строки. И чтобы запросу прочитать какие-то данные, PostgreSQL нужно загружать страницу с мусорными данными и помещать её в shared buffers. Производительность запросов снижается. Статистика планировщика перестаёт собираться. Потому что автовакуум не только чистит таблицы, а ещё и собирает статистику о распределении данных. И эта статистика используется планировщиком для того, чтобы строить оптимальные планы для запросов.

Если файлы таблицы или индекса по каким-то причинам сильно выросли в размерах, то обычная очистка освобождает место внутри существующих страниц, но число страниц в большинстве случаев не уменьшается. Единственная ситуация, при которой очистка возвращает место операционной системе, — образование полностью пустых страниц в самом конце файла, что происходит нечасто.

Если плотность информации в таблице очень низкая, можно выполнить полную очистку командой VACUUM FULL. При этом таблица и все ее индексы перестраиваются с нуля, а данные упаковываются максимально компактно (с учетом параметра fillfactor). Во время перестроения таблица полностью блокируется и для записи, и для чтения, а на диске хранятся файлы как старой так и новой таблицы. 

*****
Анонимная процедура, в которой в цикле 10 раз обновятся все строчки в нашей тестовой таблице tv и выводится шаг цикла

```sql
DO $$
BEGIN
        FOR i IN 1..10 LOOP
                UPDATE tv  tv SET str=str||' '||chr(65+i);
                RAISE NOTICE 'Шаг цикла №: %', i;
        END LOOP;
END $$;
NOTICE:  Шаг цикла №: 1
NOTICE:  Шаг цикла №: 2
NOTICE:  Шаг цикла №: 3
NOTICE:  Шаг цикла №: 4
NOTICE:  Шаг цикла №: 5
NOTICE:  Шаг цикла №: 6
NOTICE:  Шаг цикла №: 7
NOTICE:  Шаг цикла №: 8
NOTICE:  Шаг цикла №: 9
NOTICE:  Шаг цикла №: 10
DO
```
