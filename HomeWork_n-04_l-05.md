# Домашнее задание
## Работа с базами данных, пользователями и правами
 
Разворачиваем новый кластер PostgreSQL 16 на Ubuntu 22.04 LTS в WMware

```bash
#Подключаемся к новому кластеру
sudo -u postgres psql
psql (16.6 (Ubuntu 16.6-1.pgdg22.04+1))
Type "help" for help.

# Создаем новую БД testdb
CREATE DATABASE testdb;
CREATE DATABASE

#Заходим в созданную базу данных под пользователем postgres
\c testdb
You are now connected to database "testdb" as user "postgres".

# Создадим схему testnm
CREATE SCHEMA testnm;
CREATE SCHEMA

# Создадим новую таблицу t1 с одной колонкой c1 типа integer и вставим строку со значением c1=1
CREATE TABLE t1(c1 integer);
CREATE TABLE
INSERT INTO t1 VALUES(1);
INSERT 0 1

# Создадим новую роль readonly
CREATE ROLE readonly;
CREATE ROLE

# Дадим новой роли право на подключение к базе данных testdb
GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT

# Дадим новой роли право на использование схемы testnm
GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT

# Дадим новой роли право на select для всех таблиц схемы testnm
GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT

# Создадим пользователя testread с паролем test123
CREATE ROLE testread WITH LOGIN PASSWORD 'test123';
CREATE ROLE

# Дадим роль readonly пользователю testread
GRANT readonly TO testread;
GRANT ROLE

# Зайдем под пользователем testread в базу данных testdb
psql -h 127.0.0.1 -U testread -d testdb -W

# Сделаем select * from t1;
select * from t1;
ERROR:  permission denied for table t1
```

Ошибка возникла из-за того, что t1 принадлежит схеме public а не testnm, на все таблицы которой мы давали права. При создании таблицы, мы не указывали схему, не изменяли текущую схему, и таблица создалась в текущей схеме public (public - текущая, т.к. схемы с именем пользователя, который создавал таблицу - postgres, у нас нет).

```bash
\dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | postgres

SELECT current_schema();
 current_schema
----------------
 public
(1 row)

SHOW search_path;
   search_path
-----------------
 "$user", public
(1 row)

# Сохраним id объекта таблицы t1
SELECT oid, relname FROM pg_class WHERE relname='t1';
  oid  | relname
-------+---------
 16399 | t1
(1 row)
```

Под пользователем postgres удалим и создадим заново эту же таблицу, но уже с указанием схемы

```bash
DROP TABLE t1;
DROP TABLE

CREATE TABLE testnm.t1(c1 integer);
CREATE TABLE

INSERT INTO testnm.t1 VALUES(1);
INSERT 0 1

# Посмотрим еще раз id объекта таблицы t1, он изменился
SELECT oid, relname FROM pg_class WHERE relname='t1';
  oid  | relname
-------+---------
 16402 | t1
(1 row)

# Зайдем под пользователем testread в базу данных testdb и выполним select
SELECT * FROM testnm.t1;
ERROR:  permission denied for table t1
```

Отказ в доступе произошел в следствии того, что действие команды GRANT распространяется только на существующие объекты. У нас был объект таблица со своим oid=16399 и именем t1 и именно для доступа к этому объекту предоставлялись привилегии. После пересоздания таблицы у нас уже новый объект со своим oid=16402, а на него мы никаких привилегий мы не давали.

Под пользователем postgres заново раздадим права, а под пользователем testread повторим запрос

```bash
# В консоли под пользователем postgres
SELECT session_user, current_user;
 session_user | current_user
--------------+--------------
 postgres     | postgres

GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT

# В консоли под пользователем testread
SELECT session_user, current_user;
 session_user | current_user
--------------+--------------
 testread     | testread

SELECT * FROM testnm.t1;
 c1
----
  1
(1 row)
```

Теперь пользователь testread смог прочитать данные из таблицы t1.

Для того чтобы такого не повторилось в дальнейшем, есть команда ALTER DEFAULT PRIVILEGES, позволяющая задавать права, которые будут применяться к объектам, созданным в будущем. ***Она не влияет на права, назначенные уже существующим объектам.*** (Поэтому на шаге выше, мы воспользовались командой GRANT, которая действует только на существующие объекты.) Права могут быть заданы глобально (т.е. для всех объектов, создаваемых в текущей базе данных) или только для объектов, создаваемых в указанных схемах.

```bash
# В консоли под пользователем postgres
ALTER DEFAULT PRIVILEGES IN SCHEMA testnm GRANT SELECT ON TABLES TO readonly;
ALTER DEFAULT PRIVILEGES

# Создадим таблицу t11 (идентичную t1)
CREATE TABLE testnm.t11(c1 integer);
CREATE TABLE

INSERT INTO testnm.t11 VALUES(1);
INSERT 0 1

SELECT oid, relname FROM pg_class WHERE relname='t11';
  oid  | relname
-------+---------
 16406 | t11
(1 row)

# В консоли под пользователем testread
SELECT session_user, current_user;
 session_user | current_user
--------------+--------------
 testread     | testread
(1 row)

SELECT * FROM testnm.t11;
 c1
----
  1
(1 row)
```

Все получилось, теперь для всех новых таблиц в схеме testnm роль readonly будет иметь права на SELECT.

Под пользователем testread попробуем создать новую таблицу t2, аналогичную t1 (без и с указанием схемы)

```bash
# В консоли под пользователем testread
CREATE TABLE t2(c1 integer);
ERROR:  permission denied for schema public
LINE 1: CREATE TABLE t2(c1 integer);

CREATE TABLE testnm.t2(c1 integer);
ERROR:  permission denied for schema testnm
LINE 1: CREATE TABLE testnm.t2(c1 integer);

# Посмотрим путь поиска схемы
SHOW search_path;
   search_path
-----------------
 "$user", public
(1 row)

# Список схем с правами доступа
\dn+
                                       List of schemas
  Name  |       Owner       |           Access privileges            |      Description
--------+-------------------+----------------------------------------+------------------------
 public | pg_database_owner | pg_database_owner=UC/pg_database_owner+| standard public schema
        |                   | =U/pg_database_owner                   |
 testnm | postgres          | postgres=UC/postgres                  +|
        |                   | readonly=U/postgres                    |
(2 rows)
```

Т.к. схемы с именем пользователя testread у нас нет, то первая команда CREATE TABLE t2(c1 integer), будет пытаться создать таблицу в схеме public. Права для роли public (все роли всегда имеют членство в PUBLIC по умолчанию и наследуют все привилегии, присвоенные этой роли.) в схеме public только на использование (=U/pg_database_owner), прав на создание объектов в этой схеме нет, поэтому и ошибка.

Так же, ошибкой завершается попытка создать таблицу в схеме testnm, т.к. у роли readonly, членом которой является роль testread, нет прав на создание объектов в этой схеме. 

```bash
\drg
                List of role grants
 Role name  | Member of  |   Options    | Grantor
------------+------------+--------------+----------
 testread   | readonly   | INHERIT, SET | postgres
```

### Отмена всех привилегий у PUBLIC
Первое, что следует сделать для любой новой базы данных - это отменить все привилегии, которые предоставляются по умолчанию роли PUBLIC. 



```bash
# Предотвратим подключения к базе данных тех, кому не предоставлена специальная привилегия CONNECT
REVOKE ALL ON DATABASE testdb FROM PUBLIC;
REVOKE

# Запретит создавать объекты в схеме public без предоставленной привилегии. По умолчанию задана в PG15 и выше (у нас PostgreSQL 16 уже задано из коробки)
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
REVOKE
```
Отмена 'ALL' на уровне базы данных препятствует пользователям подключаться к базе данных без конкретных разрешений.

Мы удалили все ALL привилегии из 'PUBLIC' на уровне базы данных, а не на уровне схемы базы данных. Если мы также удалим все привилегии у схемы, то обычные команды PostgreSQL не будут работать для многих пользователей без повторной установки дополнительных привилегий.
