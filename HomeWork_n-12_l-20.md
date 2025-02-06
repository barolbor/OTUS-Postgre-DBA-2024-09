# Домашнее задание
## Бэкапы
 
ДЗ выполняется на ВМ в WMware (1 процессор, 2 ядра, 4Гб ОЗУ и 40Гб SSD диска), кластер PostgreSQL 16 на Ubuntu 22.04 LTS.

Создадим БД, схему, в ней таблицу, заполненную автосгенерированными 100 записями.

```sql
CREATE DATABASE testbackup;
CREATE DATABASE

\c testbackup
You are now connected to database "testbackup" as user "postgres".

CREATE SCHEMA testshema;
CREATE SCHEMA

ALTER DATABASE testbackup SET search_path TO testshema, pg_catalog;
ALTER DATABASE

CREATE TABLE testshema.maintbl AS 
SELECT 
    generate_series(1,100) AS id,
    gen_random_uuid()::char(100) AS name;

```

Под линукс пользователем Postgres создадим каталог для бэкапов и ограничим к нему доступ

```bash
sudo -u postgres mkdir /var/lib/postgresql/16/main/pb_backup
sudo -u postgres chmod 700 /var/lib/postgresql/16/main/pb_backup
```

Сделаем логический бэкап используя утилиту COPY

```sql
COPY testshema.maintbl TO '/var/lib/postgresql/16/main/pb_backup/maintbl_2025-02-06.copy' with delimiter ';';
COPY 100
```

Восстановим в 2ю таблицу данные из бэкапа. Но, прежде чем в нее восстанавливать, ее нужно создать

```sql
CREATE TABLE copytbl AS SELECT * FROM maintbl WHERE false;
SELECT 0

testbackup=# \dt
           List of relations
  Schema   |  Name   | Type  |  Owner
-----------+---------+-------+----------
 testshema | copytbl | table | postgres
 testshema | maintbl | table | postgres
(2 rows)

 COPY copytbl FROM '/var/lib/postgresql/16/main/pb_backup/maintbl_2025-02-06.copy' with delimiter ';';
COPY 100

SELECT * FROM copytbl LIMIT 5;
 id |                                                 name
----+------------------------------------------------------------------------------------------------------
  1 | 9fc53b66-4e2f-4f2f-b932-ed1ec008471c
  2 | 6cf5b079-56a4-4281-8e95-3747379c5648
  3 | 77d2977b-9b89-4a66-b643-881e49578f3d
  4 | 675fe90d-7c59-4504-8e40-012840f60d3a
  5 | 2936cc67-2a60-4b6b-8bc4-d0753a51933c
(5 rows)
```


Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц

```bash
sudo -u postgres sh -c 'pg_dump --username=postgres --dbname=testbackup --schema=testshema --format=custom --table=testshema.maintbl --table=testshema.copytbl > /var/lib/postgresql/16/main/pb_backup/maintbl_2025-02-06.dump'

# Убедимся, чтобэкап создался
sudo -u postgres ls -ls /var/lib/postgresql/16/main/pb_backup
total 24
12 -rw-r--r-- 1 postgres postgres 10392 Feb  6 15:09 maintbl_2025-02-06.copy
12 -rw-rw-r-- 1 postgres postgres  8842 Feb  6 15:57 maintbl_2025-02-06.dump
```

Используя утилиту pg_restore восстановим в новую БД только вторую таблицу!

```sql
-- Создадим новую БД и схему, без схемы не восстановим таблицу
CREATE DATABASE newdb;
CREATE DATABASE

\c newdb
You are now connected to database "newdb" as user "postgres".

newdb=# CREATE SCHEMA testshema;
CREATE SCHEMA
```

Восстановим таблицу maintbl
```bash
sudo -u postgres sh -c 'pg_restore --username=postgres --dbname=newdb --format=custom --table=maintbl --verbose /var/lib/postgresql/16/main/pb_backup/maintbl_2025-02-06.dump'
```

Проверим что восстановили
```sql
SET search_path TO testshema, pg_catalog;
SET
newdb=# \dt
           List of relations
  Schema   |  Name   | Type  |  Owner
-----------+---------+-------+----------
 testshema | maintbl | table | postgres
(1 row)

SELECT * FROM maintbl LIMIT 7;
 id |                                                 name
----+------------------------------------------------------------------------------------------------------
  1 | 9fc53b66-4e2f-4f2f-b932-ed1ec008471c
  2 | 6cf5b079-56a4-4281-8e95-3747379c5648
  3 | 77d2977b-9b89-4a66-b643-881e49578f3d
  4 | 675fe90d-7c59-4504-8e40-012840f60d3a
  5 | 2936cc67-2a60-4b6b-8bc4-d0753a51933c
  6 | a32db250-06a6-4bf1-b4f2-84dd32cf1fef
  7 | fb92cbdc-6782-4934-bc7a-5246ea39727c
(7 rows)
```

>[!NOTE]
> * Перед запуском pg_restore нужно предварительно создать схему;
> * В pg_restore имя таблицы нужно указывать без схемы;
> * Для отлавливания ошибок pg_restore нужно запускать с параметром --verbose;
