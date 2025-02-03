# Домашнее задание
## Работа с журналами
 
Настроим выполнение контрольной точки раз в 30 секунд.


```sql
-- Посмотрим текущие настройки, обратим внимание на ед. измерения unit

SELECT * FROM pg_settings WHERE name='checkpoint_timeout' \gx
-[ RECORD 1 ]---+---------------------------------------------------------
name            | checkpoint_timeout
setting         | 300
unit            | s
category        | Write-Ahead Log / Checkpoints
short_desc      | Sets the maximum time between automatic WAL checkpoints.
extra_desc      |
context         | sighup
vartype         | integer
source          | default
min_val         | 30
max_val         | 86400
enumvals        |
boot_val        | 300
reset_val       | 300
sourcefile      |
sourceline      |
pending_restart | f

SHOW log_checkpoints;
 log_checkpoints
-----------------
 on
(1 row)

-- Настроим выполнение контрольной точки раз в 30 секунд
ALTER SYSTEM SET checkpoint_timeout = 30;
ALTER SYSTEM

SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

--Проверим
SHOW checkpoint_timeout;
 checkpoint_timeout
--------------------
 30s
(1 row)

--Создадим БД для нагрузочного теста

CREATE DATABASE testwal;
CREATE DATABASE

--Посмотрим настройки по размерам WAL файлов
SHOW max_wal_size;
 max_wal_size
--------------
 1GB
(1 row)

SHOW min_wal_size;
 min_wal_size
--------------
 80MB
(1 row)

-- Посмотрим режим записи журнала WAL
SHOW synchronous_commit;
 synchronous_commit
--------------------
 on
(1 row)
```

Выполним инициализацию тестовых таблиц

```bash
sudo -u postgres pgbench -i -s 100 testwal
dropping old tables...
creating tables...
generating data (client-side)...
10000000 of 10000000 tuples (100%) done (elapsed 5.19 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 7.63 s (drop tables 0.21 s, create tables 0.00 s, client-side generate 5.34 s, vacuum 0.15 s, primary keys 1.93 s).
```

Выполним контрольную точку и запомним количество запланированных контрольных точек, которые уже были выполнены, текущую позицию добавления в журнале предзаписи LSN и имя текущего WAL файла.
 
```sql
CHECKPOINT;

postgres=# SELECT * FROM pg_stat_bgwriter\gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 649
checkpoints_req       | 48
checkpoint_write_time | 2399520
checkpoint_sync_time  | 114134
buffers_checkpoint    | 134938
buffers_clean         | 979158
maxwritten_clean      | 9695
buffers_backend       | 6030672
buffers_backend_fsync | 0
buffers_alloc         | 8763551
stats_reset           | 2025-02-02 16:25:09.462361+03

SELECT pg_current_wal_insert_lsn(), pg_walfile_name(pg_current_wal_insert_lsn());
 pg_current_wal_insert_lsn |     pg_walfile_name
---------------------------+--------------------------
 B/CADEACD0                | 000000010000000B000000CA
(1 row)
```

Подаем нагрузку в течении 10 минут c помощью утилиты pgbench в 50 клиентов и 2 потока.

```bash
sudo -u postgres  pgbench -c 50 -j 2 -P 60 -T 600 -U postgres testwal
pgbench (16.6 (Ubuntu 16.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 2413.8 tps, lat 20.684 ms stddev 20.403, 0 failed
progress: 120.0 s, 2458.0 tps, lat 20.341 ms stddev 17.164, 0 failed
progress: 180.0 s, 2468.5 tps, lat 20.251 ms stddev 16.627, 0 failed
progress: 240.0 s, 2493.8 tps, lat 20.047 ms stddev 16.411, 0 failed
progress: 300.0 s, 2563.7 tps, lat 19.500 ms stddev 15.517, 0 failed
progress: 360.0 s, 2237.0 tps, lat 22.340 ms stddev 19.257, 0 failed
progress: 420.0 s, 2142.4 tps, lat 23.344 ms stddev 19.004, 0 failed
progress: 480.0 s, 2020.9 tps, lat 24.737 ms stddev 20.229, 0 failed
progress: 540.0 s, 2198.3 tps, lat 22.744 ms stddev 18.071, 0 failed
progress: 600.0 s, 2334.1 tps, lat 21.419 ms stddev 19.274, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 1399880
number of failed transactions: 0 (0.000%)
latency average = 21.427 ms
latency stddev = 18.255 ms
initial connection time = 68.136 ms
tps = 2333.001215 (without initial connection time)
```
Рассчитаем, какой объем журнальных файлов был сгенерирован за это время. Оценим, какой объем приходится в среднем на одну контрольную точку.

```sql
-- После окончания теста
SELECT * FROM pg_stat_bgwriter\gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 662
checkpoints_req       | 56
checkpoint_write_time | 2906454
checkpoint_sync_time  | 135003
buffers_checkpoint    | 155252
buffers_clean         | 1233799
maxwritten_clean      | 12237
buffers_backend       | 7318408
buffers_backend_fsync | 0
buffers_alloc         | 11132743
stats_reset           | 2025-02-02 16:25:09.462361+03

SELECT pg_current_wal_insert_lsn(), pg_walfile_name(pg_current_wal_insert_lsn());
 pg_current_wal_insert_lsn |     pg_walfile_name
---------------------------+--------------------------
 E/5199CE00                | 000000010000000E00000051
(1 row)

--Размер журнальных записей за время теста (в байтах)
SELECT ('E/5199CE00'::pg_lsn - 'B/CADEACD0'::pg_lsn) AS wal_size;
  wal_size
-------------
 10850345264
(1 row)

--Или в более удобном формате
SELECT pg_size_pretty('E/5199CE00'::pg_lsn - 'B/CADEACD0'::pg_lsn) AS wal_size;
 wal_size
----------
 10 GB
(1 row)

-- Расчетное количество контрольных точек за 10 минут при checkpoint_timeout=30сек. равно 20

--по расписанию (по достижении интервала времени checkpoint_timeout=30сек):
 SELECT 662 - 649 AS count_cp;
 count_cp
----------
       13
(1 row)

-- По требованию (в том числе по достижении объема max_wal_size):
SELECT 56 - 48 AS count_cp_req;
 count_cp_req
--------------
            8
(1 row)

-- Средний объем данных на одну контрольную точку

SELECT ('E/5199CE00'::pg_lsn - 'B/CADEACD0'::pg_lsn) / (13+8) AS wal_per_cp;
     wal_per_cp
--------------------
 516683107.80952381
(1 row)

--Или в более удобном формате
SELECT pg_size_pretty(('E/5199CE00'::pg_lsn - 'B/CADEACD0'::pg_lsn) / (13 + 8)) AS wal_per_cp;
 wal_per_cp
------------
 493 MB
(1 row)
```

По статистике 13 контрольных точек выполнились по расписанию, а 8 по требованию. Посмотрим содержимое журнала.

```bash
sudo tail -n 100 /var/log/postgresql/postgresql-16-main.log
...
2025-02-03 11:40:49.353 MSK [1472] postgres@postgres STATEMENT:  SELECT ('Старт'::pg_lsn - 'Теста'::pg_lsn) AS wal_size;
2025-02-03 11:41:11.588 MSK [979] LOG:  checkpoints are occurring too frequently (28 seconds apart)
2025-02-03 11:41:11.588 MSK [979] HINT:  Consider increasing the configuration parameter "max_wal_size".
2025-02-03 11:41:11.588 MSK [979] LOG:  checkpoint starting: wal
2025-02-03 11:41:32.342 MSK [979] LOG:  checkpoint complete: wrote 1827 buffers (11.2%); 0 WAL file(s) added, 0 removed, 33 recycled; write=19.679 s, sync=0.961 s, total=20.755 s; sync files=28, longest=0.378 s, average=0.035 s; distance=526432 kB, estimate=526432 kB; lsn=C/9E64A98, redo lsn=B/EB002FD8
2025-02-03 11:41:33.938 MSK [979] LOG:  checkpoints are occurring too frequently (22 seconds apart)
2025-02-03 11:41:33.938 MSK [979] HINT:  Consider increasing the configuration parameter "max_wal_size".
2025-02-03 11:41:33.938 MSK [979] LOG:  checkpoint starting: wal
2025-02-03 11:41:57.127 MSK [979] LOG:  checkpoint complete: wrote 1583 buffers (9.7%); 0 WAL file(s) added, 2 removed, 31 recycled; write=20.689 s, sync=2.425 s, total=23.190 s; sync files=17, longest=1.252 s, average=0.143 s; distance=540693 kB, estimate=540693 kB; lsn=C/2C54C890, redo lsn=C/C008508
2025-02-03 11:41:57.617 MSK [979] LOG:  checkpoints are occurring too frequently (24 seconds apart)
2025-02-03 11:41:57.617 MSK [979] HINT:  Consider increasing the configuration parameter "max_wal_size".
2025-02-03 11:41:57.617 MSK [979] LOG:  checkpoint starting: wal
2025-02-03 11:42:20.772 MSK [979] LOG:  checkpoint complete: wrote 1290 buffers (7.9%); 0 WAL file(s) added, 0 removed, 33 recycled; write=22.078 s, sync=1.002 s, total=23.155 s; sync files=10, longest=0.703 s, average=0.101 s; distance=540675 kB, estimate=540691 kB; lsn=C/4B4CB890, redo lsn=C/2D009380
2025-02-03 11:42:23.100 MSK [979] LOG:  checkpoints are occurring too frequently (26 seconds apart)
2025-02-03 11:42:23.100 MSK [979] HINT:  Consider increasing the configuration parameter "max_wal_size".
2025-02-03 11:42:23.101 MSK [979] LOG:  checkpoint starting: wal
2025-02-03 11:42:46.842 MSK [979] LOG:  checkpoint complete: wrote 1123 buffers (6.9%); 0 WAL file(s) added, 0 removed, 33 recycled; write=22.954 s, sync=0.443 s, total=23.742 s; sync files=22, longest=0.269 s, average=0.021 s; distance=540697 kB, estimate=540697 kB; lsn=C/6BD89FA8, redo lsn=C/4E00F840
2025-02-03 11:42:49.421 MSK [979] LOG:  checkpoints are occurring too frequently (26 seconds apart)
2025-02-03 11:42:49.421 MSK [979] HINT:  Consider increasing the configuration parameter "max_wal_size".
2025-02-03 11:42:49.421 MSK [979] LOG:  checkpoint starting: wal
2025-02-03 11:43:14.467 MSK [979] LOG:  checkpoint complete: wrote 1013 buffers (6.2%); 0 WAL file(s) added, 0 removed, 33 recycled; write=23.907 s, sync=0.954 s, total=25.046 s; sync files=11, longest=0.617 s, average=0.087 s; distance=540642 kB, estimate=540691 kB; lsn=C/8D242098, redo lsn=C/6F0081E8
2025-02-03 11:43:17.175 MSK [979] LOG:  checkpoints are occurring too frequently (28 seconds apart)
2025-02-03 11:43:17.175 MSK [979] HINT:  Consider increasing the configuration parameter "max_wal_size".
2025-02-03 11:43:17.176 MSK [979] LOG:  checkpoint starting: wal
2025-02-03 11:43:43.848 MSK [979] LOG:  checkpoint complete: wrote 928 buffers (5.7%); 0 WAL file(s) added, 0 removed, 33 recycled; write=25.276 s, sync=1.312 s, total=26.673 s; sync files=19, longest=0.914 s, average=0.069 s; distance=540701 kB, estimate=540701 kB; lsn=C/AE70BE78, redo lsn=C/9000F730
2025-02-03 11:43:46.503 MSK [979] LOG:  checkpoints are occurring too frequently (29 seconds apart)
2025-02-03 11:43:46.503 MSK [979] HINT:  Consider increasing the configuration parameter "max_wal_size".
2025-02-03 11:43:46.503 MSK [979] LOG:  checkpoint starting: wal
2025-02-03 11:44:14.176 MSK [979] LOG:  checkpoint complete: wrote 920 buffers (5.6%); 0 WAL file(s) added, 0 removed, 33 recycled; write=26.504 s, sync=1.115 s, total=27.674 s; sync files=12, longest=0.746 s, average=0.093 s; distance=540649 kB, estimate=540696 kB; lsn=C/CF581FC0, redo lsn=C/B1009CE0
2025-02-03 11:44:16.176 MSK [979] LOG:  checkpoint starting: time
2025-02-03 11:44:43.915 MSK [979] LOG:  checkpoint complete: wrote 821 buffers (5.0%); 0 WAL file(s) added, 0 removed, 32 recycled; write=26.873 s, sync=0.727 s, total=27.739 s; sync files=18, longest=0.498 s, average=0.041 s; distance=530669 kB, estimate=539693 kB; lsn=C/EF253490, redo lsn=C/D1645368
2025-02-03 11:44:46.726 MSK [979] LOG:  checkpoint starting: time
2025-02-03 11:45:14.958 MSK [979] LOG:  checkpoint complete: wrote 906 buffers (5.5%); 0 WAL file(s) added, 0 removed, 32 recycled; write=26.332 s, sync=1.774 s, total=28.233 s; sync files=11, longest=1.336 s, average=0.162 s; distance=533949 kB, estimate=539119 kB; lsn=D/10ECE9A8, redo lsn=C/F1FB48B8
2025-02-03 11:45:16.000 MSK [979] LOG:  checkpoints are occurring too frequently (29 seconds apart)
2025-02-03 11:45:16.000 MSK [979] HINT:  Consider increasing the configuration parameter "max_wal_size".
2025-02-03 11:45:16.000 MSK [979] LOG:  checkpoint starting: wal
2025-02-03 11:45:43.551 MSK [979] LOG:  checkpoint complete: wrote 961 buffers (5.9%); 0 WAL file(s) added, 0 removed, 33 recycled; write=26.085 s, sync=1.411 s, total=27.551 s; sync files=16, longest=1.076 s, average=0.089 s; distance=524629 kB, estimate=537670 kB; lsn=D/2FE8A3F8, redo lsn=D/12009DC0
2025-02-03 11:45:45.552 MSK [979] LOG:  checkpoint starting: time
2025-02-03 11:46:12.479 MSK [979] LOG:  checkpoint complete: wrote 821 buffers (5.0%); 0 WAL file(s) added, 0 removed, 31 recycled; write=26.526 s, sync=0.339 s, total=26.928 s; sync files=11, longest=0.243 s, average=0.031 s; distance=522044 kB, estimate=536107 kB; lsn=D/4D9FFE80, redo lsn=D/31DD90C0
2025-02-03 11:46:15.480 MSK [979] LOG:  checkpoint starting: time
2025-02-03 11:46:42.366 MSK [979] LOG:  checkpoint complete: wrote 895 buffers (5.5%); 0 WAL file(s) added, 0 removed, 31 recycled; write=26.597 s, sync=0.222 s, total=26.886 s; sync files=16, longest=0.101 s, average=0.014 s; distance=493817 kB, estimate=531878 kB; lsn=D/6956C148, redo lsn=D/50017618
2025-02-03 11:46:45.368 MSK [979] LOG:  checkpoint starting: time
2025-02-03 11:47:12.324 MSK [979] LOG:  checkpoint complete: wrote 846 buffers (5.2%); 0 WAL file(s) added, 0 removed, 27 recycled; write=26.692 s, sync=0.068 s, total=26.957 s; sync files=11, longest=0.020 s, average=0.007 s; distance=456396 kB, estimate=524330 kB; lsn=D/8408D558, redo lsn=D/6BDCA848
2025-02-03 11:47:15.328 MSK [979] LOG:  checkpoint starting: time
2025-02-03 11:47:42.384 MSK [979] LOG:  checkpoint complete: wrote 769 buffers (4.7%); 0 WAL file(s) added, 0 removed, 27 recycled; write=26.748 s, sync=0.242 s, total=27.057 s; sync files=15, longest=0.177 s, average=0.017 s; distance=441146 kB, estimate=516011 kB; lsn=D/A0153DF8, redo lsn=D/86C991D8
2025-02-03 11:47:45.388 MSK [979] LOG:  checkpoint starting: time
2025-02-03 11:48:16.121 MSK [979] LOG:  checkpoint complete: wrote 839 buffers (5.1%); 0 WAL file(s) added, 0 removed, 28 recycled; write=26.685 s, sync=3.805 s, total=30.733 s; sync files=13, longest=3.036 s, average=0.293 s; distance=459471 kB, estimate=510357 kB; lsn=D/BEA78090, redo lsn=D/A2D4CEF8
2025-02-03 11:48:16.121 MSK [979] LOG:  checkpoint starting: time
2025-02-03 11:48:44.206 MSK [979] LOG:  checkpoint complete: wrote 718 buffers (4.4%); 0 WAL file(s) added, 0 removed, 28 recycled; write=26.913 s, sync=1.018 s, total=28.086 s; sync files=19, longest=0.721 s, average=0.054 s; distance=458239 kB, estimate=505146 kB; lsn=D/D7049468, redo lsn=D/BECCCEB8
2025-02-03 11:48:46.208 MSK [979] LOG:  checkpoint starting: time
2025-02-03 11:49:14.605 MSK [979] LOG:  checkpoint complete: wrote 930 buffers (5.7%); 0 WAL file(s) added, 0 removed, 26 recycled; write=26.841 s, sync=1.267 s, total=28.397 s; sync files=12, longest=0.766 s, average=0.106 s; distance=425160 kB, estimate=497147 kB; lsn=D/F0E8C828, redo lsn=D/D8BFF1F0
2025-02-03 11:49:16.607 MSK [979] LOG:  checkpoint starting: time
2025-02-03 11:49:43.620 MSK [979] LOG:  checkpoint complete: wrote 898 buffers (5.5%); 0 WAL file(s) added, 0 removed, 26 recycled; write=26.465 s, sync=0.417 s, total=27.014 s; sync files=15, longest=0.297 s, average=0.028 s; distance=429031 kB, estimate=490335 kB; lsn=E/D72C3E8, redo lsn=D/F2EF8E38
2025-02-03 11:49:46.624 MSK [979] LOG:  checkpoint starting: time
2025-02-03 11:50:14.404 MSK [979] LOG:  checkpoint complete: wrote 872 buffers (5.3%); 0 WAL file(s) added, 0 removed, 30 recycled; write=26.463 s, sync=1.102 s, total=27.780 s; sync files=10, longest=0.791 s, average=0.111 s; distance=480273 kB, estimate=489329 kB; lsn=E/2B79DBA8, redo lsn=E/103FD378
2025-02-03 11:50:16.404 MSK [979] LOG:  checkpoint starting: time
2025-02-03 11:50:43.445 MSK [979] LOG:  checkpoint complete: wrote 820 buffers (5.0%); 0 WAL file(s) added, 0 removed, 29 recycled; write=26.627 s, sync=0.265 s, total=27.042 s; sync files=17, longest=0.127 s, average=0.016 s; distance=475632 kB, estimate=487959 kB; lsn=E/47F47E60, redo lsn=E/2D479468
2025-02-03 11:50:46.448 MSK [979] LOG:  checkpoint starting: time
2025-02-03 11:51:14.404 MSK [979] LOG:  checkpoint complete: wrote 865 buffers (5.3%); 0 WAL file(s) added, 0 removed, 29 recycled; write=26.553 s, sync=1.367 s, total=27.956 s; sync files=8, longest=1.365 s, average=0.171 s; distance=483997 kB, estimate=487563 kB; lsn=E/5199CE38, redo lsn=E/4AD20BC8
```

Из журнала мы видим, что 8 контрольных точек происходили слишком часто, с интервалом меньше 30 секунд., т.е. они были внеплановыми, их вызов был связан с тем, что из-за нагрузки был сгенерирован слишком большой объем журнальных записей WAL, превышающий заданный в параметре max_wal_size=1GB, а при превышении этого предела сервер инициирует внеплановую контрольную точку. Так же в журнале мы видим рекомендации по рассмотрению возможности увеличения параметра конфигурации "max_wal_size".

Переключим режим записи журналов WAL в асинхронный режим.

```sql
SHOW synchronous_commit;
 synchronous_commit
--------------------
 on
(1 row)

ALTER SYSTEM SET synchronous_commit=off;

SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# SELECT * FROM pg_settings WHERE name='synchronous_commit' \gx
-[ RECORD 1 ]---+------------------------------------------------------
name            | synchronous_commit
setting         | off
unit            |
category        | Write-Ahead Log / Settings
short_desc      | Sets the current transactions synchronization level.
extra_desc      |
context         | user
vartype         | enum
source          | configuration file
min_val         |
max_val         |
enumvals        | {local,remote_write,remote_apply,on,off}
boot_val        | on
reset_val       | off
sourcefile      | /var/lib/postgresql/16/main/postgresql.auto.conf
sourceline      | 4
pending_restart | f
```

И еще раз запустим наш тест: в течении 10 минут c помощью утилиты pgbench в 50 клиентов и 2 потока.

```bash
sudo -u postgres  pgbench -c 50 -j 2 -P 60 -T 600 -U postgres testwal
[sudo] password for sa5:
pgbench (16.6 (Ubuntu 16.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 2785.6 tps, lat 17.922 ms stddev 18.761, 0 failed
progress: 120.0 s, 2879.4 tps, lat 17.353 ms stddev 21.895, 0 failed
progress: 180.0 s, 2804.1 tps, lat 17.839 ms stddev 21.127, 0 failed
progress: 240.0 s, 2787.7 tps, lat 17.934 ms stddev 21.160, 0 failed
progress: 300.0 s, 2823.8 tps, lat 17.704 ms stddev 20.205, 0 failed
progress: 360.0 s, 2820.2 tps, lat 17.729 ms stddev 21.935, 0 failed
progress: 420.0 s, 2773.5 tps, lat 18.022 ms stddev 19.569, 0 failed
progress: 480.0 s, 2619.7 tps, lat 19.088 ms stddev 20.180, 0 failed
progress: 540.0 s, 2618.5 tps, lat 19.088 ms stddev 20.044, 0 failed
progress: 600.0 s, 2737.1 tps, lat 18.269 ms stddev 21.024, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 1659019
number of failed transactions: 0 (0.000%)
latency average = 18.080 ms
latency stddev = 20.633 ms
initial connection time = 73.439 ms
tps = 2764.874295 (without initial connection time)
```

Значение tps выросло на 431 с 2333 до 2764, так как в асинхронном режиме транзакция завершается немедленно, а журнал записывается в фоновом режиме, т.е. транзакции не дожидаются подтверждения записи WAL на диск.

Асинхронная запись эффективнее синхронной — фиксация изменений не ждет физической записи на диск. Однако надежность уменьшается: в случае сбоя зафиксированные данные могут пропасть, если после фиксации прошло менее 3 × wal_writer_delay единиц времени (что при настройке по умолчанию составляет 0,6 секунды).

*****

Создадим новый кластер с включенной контрольной суммой страниц. 

```bash
sudo -u postgres pg_createcluster 16 crc -- --data-checksums
Creating new PostgreSQL cluster 16/crc ...
/usr/lib/postgresql/16/bin/initdb -D /var/lib/postgresql/16/crc --auth-local peer --auth-host scram-sha-256 --no-instructions --data-checksums
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/16/crc ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Europe/Moscow
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Warning: systemd does not know about the new cluster yet. Operations like "service postgresql start" will not handle it. To fix, run:
  sudo systemctl daemon-reload
Ver Cluster Port Status Owner    Data directory             Log file
16  crc     5433 down   postgres /var/lib/postgresql/16/crc /var/log/postgresql/postgresql-16-crc.log

sudo systemctl start postgresql@16-crc

pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  crc     5433 online postgres /var/lib/postgresql/16/crc  /var/log/postgresql/postgresql-16-crc.log
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log

sudo -u postgres psql -p 5433
psql (16.6 (Ubuntu 16.6-1.pgdg22.04+1))
Type "help" for help.
```

```sql
-- Проверим, включён ли в кластере контроль целостности данных
SHOW data_checksums;
 data_checksums
----------------
 on
(1 row)

-- Создадим тестовую БД
CREATE DATABASE testcrc;

-- Подключимся к тестовой БД
\c testcrc
You are now connected to database "testcrc" as user "postgres".

-- Создадим тестовую таблицу и наполним ее данными
CREATE TABLE accounts(
    id integer PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY,
    client text,
    amount numeric);
CREATE TABLE

INSERT INTO accounts(id, client, amount)
    VALUES (1,'alice',100.00),
    (2,'bob',200.00),
    (3,'charlie',300.00);
INSERT 0 3

-- Сделаем select для простановки бита xmin committed
 SELECT * FROM accounts;
 id | client  | amount
----+---------+--------
  1 | alice   | 100.00
  2 | bob     | 200.00
  3 | charlie | 300.00
(3 rows)

-- Полный путь к файлу таблицы accounts (относительно каталога данных PGDATA).
SELECT pg_relation_filepath('accounts');
 pg_relation_filepath
----------------------
 base/16388/16390
(1 row)

\q
```

Остановим кластер

```bash
sudo systemctl stop postgresql@16-crc
```

Отредактируем файл таблицы, найдем имя charlie и заменим его на Vasilii

```bash
sudo nano /var/lib/postgresql/16/crc/base/16388/16390
```

Запустим кластер

```bash
sudo systemctl start postgresql@16-crc

sudo -u postgres psql -p 5433
psql (16.6 (Ubuntu 16.6-1.pgdg22.04+1))
Type "help" for help.
```

Подключимся к тестовой БД и попытаемся прочитать данные из тестовой таблицы

```sql
\c testcrc
You are now connected to database "testcrc" as user "postgres".

SELECT * FROM accounts;
WARNING:  page verification failed, calculated checksum 26075 but expected 38999
ERROR:  invalid page in block 0 of relation base/16388/16390
```

Мы видим:
* Предупреждение: проверка страницы не удалась, рассчитанная контрольная сумма 26075, а ожидалась 38999;
* Ошибка: недопустимая страница в блоке 0 таблицы base/16388/16390.


Данная ошибка возникла из-за несоответствия контрольных сумм рассчитанной для страницы данных с сохраненной. При инициализации кластера, мы включили контроль целостности данных, в таком случае каждая страница данных будет содержать контрольную сумму, рассчитываемую при записи и проверяемую при каждом чтении страницы. Контрольными суммами защищены только страницы данных, но не внутренние структуры данных и временные файлы. Расчёт контрольных сумм может повлечь заметное снижение производительности. 
При попытке восстановить страницы после повреждения иногда нужно обойти защиту, защищенную контрольными суммами. Для этого можно временно установить параметр конфигурации gnore_checksum_failure. Если параметр ignore_checksum_failure включён, система игнорирует проблему (но всё же предупреждает о ней) и продолжает обработку. Это поведение может привести к краху, распространению или сокрытию повреждения данных и другим серьёзными проблемам. Однако включив его, вы можете обойти ошибку и получить неповреждённые данные, которые могут находиться в таблице, если цел заголовок блока. Если же повреждён заголовок, будет выдана ошибка, даже когда этот параметр включён.

```sql
-- Включим игнорирование проверки контрольных сумм
SET ignore_checksum_failure = on;
SET

SHOW ignore_checksum_failure;
 ignore_checksum_failure
-------------------------
 on
(1 row)

-- Попытаемся получить данные из тестовой таблицы
SELECT * FROM accounts;
WARNING:  page verification failed, calculated checksum 26075 but expected 38999
 id | client  | amount
----+---------+--------
  1 | alice   | 100.00
  2 | bob     | 200.00
  3 | Vasilii | 300.00
(3 rows)
```

Запрос успешно выполнился, и вместо имени charlie мы видим имя Vasilii. Так же PostreSQL предупреждает нас о том, что проверка страницы не удалась, рассчитанная контрольная сумма 26075, а ожидалась 38999;
