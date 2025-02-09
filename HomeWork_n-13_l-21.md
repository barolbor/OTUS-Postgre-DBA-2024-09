# Домашнее задание
## Репликация
### Реализовать свой миникластер на 3 ВМ.

ДЗ выполняется на ВМ в WMware. У всех  ВМ была однотипная конфигурация: 1 процессор, 2 ядра, 4Гб ОЗУ и 20Гб SSD диска, кластер PostgreSQL 16 на Ubuntu 22.04 LTS.

Инфраструктура:
|||||||
|-|-|-|-|-|-|
| ВМ | ip | ver | port | user | pswd |
| 1 |192.168.145.101| 16 | 5432 | postgres | pg1 |
| 2 |192.168.145.102| 16 | 5432 | postgres | pg2 |
| 3 |192.168.145.103| 16 | 5432 | postgres | pg3 |
| 4 |192.168.145.104| 16 | 5432 | postgres | pg4 |

Первоначальные настройки для PostgreSQL одинаковы для всех ВМ: настройки по умолчанию плюс:

```bash
# Разрешаем прослушивать все адреса, Московское время
sudo sh -c 'echo "
listen_addresses = '"'*'"'
log_timezone = 'Europe/Moscow'
timezone = 'Europe/Moscow'
" >> /etc/postgresql/16/main/postgresql.conf'

# Разрешаем аутентификацию по паролю  при подключении с любого адреса
sudo sh -c 'echo "
host    all             all             0.0.0.0/0               scram-sha-256
host    all             all             ::/0                    scram-sha-256
" >> /etc/postgresql/16/main/pg_hba.conf'
```

На 1ой ВМ создаем таблицы test для записи, test2 для запросов на чтение. Создаем публикацию таблицы test.

```sql
-- Проверяем уровень журналирования:
SHOW wal_level ;
 wal_level
-----------
 replica
(1 row)

-- Репликация конкретной таблицы возможна только на уровне logical. Зададим ее
ALTER SYSTEM SET wal_level = logical;

-- Чтобы сделанные изменения вступили в силу, нужно перезагрузить кластер
sudo systemctl restart postgresql@16-main

-- Зададим пароль для пользователя postgres
\password 
pg1

-- Создадим тестовую БД и подключимся к ней
CREATE DATABASE testrep;
CREATE DATABASE

\c testrep
You are now connected to database "testrep" as user "postgres".

-- Создадим тестовую схему
CREATE SCHEMA testschema;
CREATE SCHEMA

ALTER DATABASE testrep SET search_path TO testschema;
ALTER DATABASE

 \c testrep
You are now connected to database "testrep" as user "postgres".

-- Создадим тестовые таблицы
CREATE TABLE test(id SERIAL PRIMARY KEY, fio char(20));
CREATE TABLE

CREATE TABLE test2(id SERIAL PRIMARY KEY, name char(20));
CREATE TABLE

-- Наполним таблицу test данными
INSERT INTO test (fio) VALUES ('01 fio'),('02 fio'),('03 fio'),('04 fio'),('05 fio');
INSERT 0 5

-- Создаем публикацию таблицы test
CREATE PUBLICATION test_pub FOR TABLE test;
CREATE PUBLICATION

-- Посмотрим созданные публикации
\dRp+
                            Publication test_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "testschema.test"
```

На 2ой ВМ создаем таблицы test2 для записи, test для запросов на чтение. Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1.

```sql
-- Проделываем все шаги, как и на 1ой ВМ вплоть до наполнения и публикации таблицы test
ALTER SYSTEM SET wal_level = logical;
ALTER SYSTEM

sudo systemctl restart postgresql@16-main

\password 
pg2

CREATE DATABASE testrep;
CREATE DATABASE

\c testrep
You are now connected to database "testrep" as user "postgres".

CREATE SCHEMA testschema;
CREATE SCHEMA

ALTER DATABASE testrep SET search_path TO testschema;
ALTER DATABASE

 \c testrep
You are now connected to database "testrep" as user "postgres".

CREATE TABLE test(id SERIAL PRIMARY KEY, fio char(20));
CREATE TABLE

CREATE TABLE test2(id SERIAL PRIMARY KEY, name char(20));
CREATE TABLE

-- Наполним таблицу test2 данными
INSERT INTO test2 (name) VALUES ('01 name'),('02 name'),('03 name'),('04 name'),('05 name');

-- Создаем публикацию таблицы test2
CREATE PUBLICATION test2_pub FOR TABLE test2;
CREATE PUBLICATION

-- Посмотрим созданные публикации
\dRp+
                           Publication test2_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "testschema.test2"

-- Создадим подписку на публикацию таблицы test на 1ой ВМ
CREATE SUBSCRIPTION test_sub
CONNECTION 'host=192.168.145.101 port=5432 user=postgres password=pg1 dbname=testrep'
PUBLICATION test_pub WITH (copy_data = true);
NOTICE:  created replication slot "test_sub" on publisher
CREATE SUBSCRIPTION

-- Посмотрим созданные подписки
\dRs
            List of subscriptions
   Name   |  Owner   | Enabled | Publication
----------+----------+---------+-------------
 test_sub | postgres | t       | {test_pub}
(1 row)

-- Посмотрим содержимое таблицы test, данные на этой 2ой ВМ мы в нее не заливали. Ожидаем, что данные зальются с 1ой ВМ
SELECT * FROM test;
 id |         fio
----+----------------------
  1 | 01 fio
  2 | 02 fio
  3 | 03 fio
  4 | 04 fio
  5 | 05 fio
(5 rows)

-- Отлично, подписка заработала
```

На 1ой ВМ подписываемся на публикацию таблицы test2 со 2ой ВМ.

```sql
-- Содержимое таблицы test2
SELECT * FROM test2;
 id | name
----+------
(0 rows)

-- Создадим подписку на публикацию таблицы test2 на 2ой ВМ
CREATE SUBSCRIPTION test2_sub
CONNECTION 'host=192.168.145.102 port=5432 user=postgres password=pg2 dbname=testrep'
PUBLICATION test2_pub WITH (copy_data = true);
NOTICE:  created replication slot "test2_sub" on publisher
CREATE SUBSCRIPTION

-- Содержимое таблицы test2
SELECT * FROM test2;
 id |         name
----+----------------------
  1 | 01 name
  2 | 02 name
  3 | 03 name
  4 | 04 name
  5 | 05 name
(5 rows)

-- Отлично и эта, подписка заработала
```

На 3ей ВМ создаем подписки на таблицы из 1ой и 2ой ВМ, ее будем использовать как реплику для чтения и бэкапов. 

```sql
-- Проверяем уровень журналирования:
SHOW wal_level ;
 wal_level
-----------
 replica
(1 row)

-- Его не трогаем, он нам нужен для построения репликации на 4ю ВМ

-- Далее проделываем все шаги, как и на 1ой ВМ вплоть до наполнения и публикации таблиц

\password 
pg3

CREATE DATABASE testrep;
CREATE DATABASE

\c testrep
You are now connected to database "testrep" as user "postgres".

CREATE SCHEMA testschema;
CREATE SCHEMA

ALTER DATABASE testrep SET search_path TO testschema;
ALTER DATABASE

 \c testrep
You are now connected to database "testrep" as user "postgres".

CREATE TABLE test(id SERIAL PRIMARY KEY, fio char(20));
CREATE TABLE

CREATE TABLE test2(id SERIAL PRIMARY KEY, name char(20));
CREATE TABLE

-- Создадим подписку на публикацию таблицы test на 1ой ВМ
CREATE SUBSCRIPTION test_from_3_to_1_sub
CONNECTION 'host=192.168.145.101 port=5432 user=postgres password=pg1 dbname=testrep'
PUBLICATION test_pub WITH (copy_data = true);
NOTICE:  created replication slot "test_from_3_to_1_sub" on publisher
CREATE SUBSCRIPTION

-- Создадим подписку на публикацию таблицы test2 на 2ой ВМ
CREATE SUBSCRIPTION test2_from_3_to_2_sub
CONNECTION 'host=192.168.145.102 port=5432 user=postgres password=pg2 dbname=testrep'
PUBLICATION test2_pub WITH (copy_data = true);
NOTICE:  created replication slot "test2_from_3_to_2_sub" on publisher
CREATE SUBSCRIPTION

-- Посмотрим созданные подписки
\dRs
                  List of subscriptions
         Name          |  Owner   | Enabled | Publication
-----------------------+----------+---------+-------------
 test2_from_3_to_2_sub | postgres | t       | {test2_pub}
 test_from_3_to_1_sub  | postgres | t       | {test_pub}
(2 rows)

-- Посмотрим содержимое таблиц
SELECT * FROM test;
 id |         fio
----+----------------------
  1 | 01 fio
  2 | 02 fio
  3 | 03 fio
  4 | 04 fio
  5 | 05 fio
(5 rows)

SELECT * FROM test2;
 id |         name
----+----------------------
  1 | 01 name
  2 | 02 name
  3 | 03 name
  4 | 04 name
  5 | 05 name
(5 rows)

-- Отлично все подписки работают
```

Реализуем горячее реплицирование для высокой доступности на 4ую ВМ. Источником должна выступать 3ия ВМ. Для реализации нам потребуется физическая репликация.

Настраиваем мастера 3ию ВМ

```bash
# Разрешаем аутентификацию по паролю при подключении с любого адреса для репликации
sudo sh -c 'echo "
host    replication     all             0.0.0.0/0               scram-sha-256
" >> /etc/postgresql/16/main/pg_hba.conf'
```

```sql
-- Проверим настройки на 3ей ВМ
SELECT name, setting FROM pg_settings where name in ('wal_level','max_wal_senders','hot_standby','synchronous_standby_names','synchronous_commit') ORDER BY name;
           name            | setting
---------------------------+---------
 hot_standby               | on
 max_wal_senders           | 10
 synchronous_commit        | on
 synchronous_standby_names |
 wal_level                 | replica
(5 rows)
```

Настраиваем реплику 4ую ВМ

```bash
# Останавливаем кластер

sudo systemctl stop postgresql@16-main
pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 down   postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log

# Удаляем каталог с данными
sudo -u postgres rm -r /var/lib/postgresql/16/main
ls -la /var/lib/postgresql/16
total 8
drwxr-xr-x 2 postgres postgres 4096 Feb  9 18:49 .
drwxr-xr-x 3 postgres postgres 4096 Feb  9 18:07 ..

# Сделаем бэкап БД testrep мастера - 3ей ВМ. Ключ 
# --write-recovery-conf Создать файл standby.signal и добавить параметры конфигурации в файл postgresql.auto.conf в целевом каталоге
# --format=plain Записывает выводимые данные в обычные файлы, сохраняя структуру каталогов данных и табличных пространств как на исходном сервере.
# --wal-method=stream Передавать журнал предзаписи в процессе создания резервной копии.

sudo -u postgres pg_basebackup --host=192.168.145.103 --port=5432 --write-recovery-conf --pgdata=/var/lib/postgresql/16/main --format=plain --wal-method=stream
Password: pg3
```

Проверим репликацию

```sql
-- На мастере 3ейпосмотрим подключенные реплики
SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 2345
usesysid         | 10
usename          | postgres
application_name | 16/main
client_addr      | 192.168.145.104
client_hostname  |
client_port      | 36872
backend_start    | 2025-02-09 19:14:26.221362+03
backend_xmin     |
state            | streaming
sent_lsn         | 0/4000148
write_lsn        | 0/4000148
flush_lsn        | 0/4000148
replay_lsn       | 0/4000148
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2025-02-09 19:23:46.613024+03

-- На реплике 4ой ВМ проверяем статистику приемника WAL файлов 
SELECT * FROM pg_stat_wal_receiver \gx
-[ RECORD 1 ]---------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
pid                   | 5012
status                | streaming
receive_start_lsn     | 0/4000000
receive_start_tli     | 1
written_lsn           | 0/4000148
flushed_lsn           | 0/4000148
received_tli          | 1
last_msg_send_time    | 2025-02-09 19:24:26.631865+03
last_msg_receipt_time | 2025-02-09 19:24:26.622392+03
latest_end_lsn        | 0/4000148
latest_end_time       | 2025-02-09 19:18:26.500174+03
slot_name             |
sender_host           | 192.168.145.103
sender_port           | 5432
conninfo              | user=postgres password=******** channel_binding=prefer dbname=replication host=192.168.145.103 port=5432 fallback_application_name=16/main sslmode=prefer sslnegotiation=postgres sslcompression=0 sslcertmode=allow sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres gssdelegation=0 target_session_attrs=any load_balance_hosts=disable
```

Протестируем всю схему целиком

```sql
-- 1ая ВМ
INSERT INTO test(fio) VALUES ('06 fio replica');
UPDATE test SET fio = fio || ' replica' WHERE id = 5;
DELETE FROM test WHERE id =4;

-- 2ая ВМ
INSERT INTO test2(name) VALUES ('06 name replica');
UPDATE test2 SET name = name || ' replica' WHERE id = 5;
DELETE FROM test2 WHERE id =4;

-- 3ия ВМ
-- 4ая ВМ
SELECT * FROM test;
 id |         fio
----+----------------------
  1 | 01 fio
  2 | 02 fio
  3 | 03 fio
  6 | 06 fio replica
  5 | 05 fio replica
(5 rows)

SELECT * FROM test2;
 id |         name
----+----------------------
  1 | 01 name
  2 | 02 name
  3 | 03 name
  6 | 06 name replica
  5 | 05 name replica
(5 rows)
```

Репликация работает
