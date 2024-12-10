# Домашнее задание
## Работа с уровнями изоляции транзакции в PostgreSQL
 
 ### 1. Создание виртуальной машины в yandex.cloud на хосте с ОС семейства Windows

Создаем новую пару ключей:

```bash
cd %USERPROFILE%/.ssh
ssh-keygen -t ed25519 -C ""
```

Создаем облачную сеть bob-yc-net:

```bash
 yc vpc network create \
    --name bob-yc-net \
    --labels bob-label=bob-value \
    --description "bob first network via yc"
```

Создаем подсеть bob-yc-subnet в  облачной сети bob-yc-net:

```bash
 yc vpc subnet create \
    --name bob-yc-subnet \
    --zone ru-central1-d \
    --range 192.168.222.0/24 \
    --network-name bob-yc-net \
    --description "bob first subnet via yc"
```

Создаем виртуальную машину bob-yc-vm на Linux Ubuntu 20.04 LTS

```bash
 yc compute instance create \
    --name bob-yc-vm \
    --hostname bob-yc-vm \
    --cores 2 \
    --memory 4 \
    --create-boot-disk size=20G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2004-lts \
    --network-interface subnet-name=bob-yc-subnet,nat-ip-version=ipv4 \
    --zone ru-central1-d \
    --ssh-key %USERPROFILE%/.ssh/yc_bob_key.pub
```

Получаем IP адрес виртуальной машины:

```bash
yc compute instance get bob-yc-vm
...
network_interfaces:
  - index: "0"
    mac_address: d0:0d:15:96:c3:8b
    subnet_id: fl8ufan1lnn7jctsn1b6
    primary_v4_address:
      address: 192.168.222.34
      one_to_one_nat:
        address: 158.160.146.175
        ip_version: IPV4
...
```

Подключаемся к виртуальной машине

```bash
ssh -i  %USERPROFILE%/.ssh/yc_bob_key sa@158.160.146.175
```

Устанавливаем PostgreSQL 16:

```bash
 sudo apt update
 sudo apt upgrade -y -q
 sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
 wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
 sudo apt-get update
 sudo apt -y install postgresql-16
```

Задаем пароль для роли postgres:

```bash
sudo -u postgres psql
psql (16.6 (Ubuntu 16.6-1.pgdg20.04+1))
Type "help" for help.

\password
Enter new password for user "postgres":
Enter it again:

\q
```

Открываем доступ к кластеру из интернета:

```bash
  # Разрешаем прослушивать все адреса
sudo nano /etc/postgresql/16/main/postgresql.conf
  
  listen_addresses = '*'

  # Разрешаем аутентификацию по паролю  при подключении с любого адреса
sudo nano /etc/postgresql/16/main/pg_hba.conf

  # TYPE  DATABASE        USER            ADDRESS                 METHOD
  host    all             all             0.0.0.0/0               md5
  host    all             all             ::/0                    md5

  # Чтобы изменения вступили в силу, перезагружаем кластер
sudo pg_ctlcluster 16 main restart
```

Подключаемся к кластеру из интернета

```bash
$ psql -p 5432 -U postgres -h 130.193.58.53 -d postgres -W
Password:
psql (16.6 (Ubuntu 16.6-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.

postgres=#
```

 ### 2. Работа с уровнями изоляции транзакции в PostgreSQL

Подключаемся к кластеру 2мя сессиями и в каждой отключает AUTOCONNIT:

```bash
# Сессия № 1 и № 2

\echo :AUTOCOMMIT
on

\set AUTOCOMMIT off

\echo :AUTOCOMMIT
off
```

Создадим тестовую таблицу и наполним ее данными

```bash
# Сессия № 1

CREATE DATABASE bob;
\c bob;
START TRANSACTION;
CREATE TABLE persons(id serial, first_name text, second_name text);
INSERT INTO persons(first_name, second_name) VALUES('ivan', 'ivanov'); 
INSERT INTO persons(first_name, second_name) VALUES('petr', 'petrov'); 
COMMIT;

SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```

Текущий уровень изоляции транзакций:

```bash
# Сессия № 1

SHOW transaction_isolation;

 transaction_isolation
-----------------------
 read committed
(1 row)
```

Начнем новую транзакцию в 1-ой сессии и добавим запись в таблицу:

```bash
# Сессия № 1

START TRANSACTION;
INSERT INTO persons(first_name, second_name) VALUES('sergey', 'sergeev');
SELECT * FROM persons;

 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
```

Во второй сессии начнем еще одну транзакцию:

```bash
# Сессия № 2

START TRANSACTION;
SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```

Во 2-ой транзакции не видно незафиксированных изменений, сделанных в 1-ой транзакции, т.к. "__Грязное чтение__" на уровне изоляции "__Read committed__" не допускается.

Завершим транзакцию в 1-ой сессии:

```bash
# Сессия № 1

COMMIT;
SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

Выполним SELECT во 2-ой сессии с еще не завершенной транзакцией:

```bash
# Сессия № 2

SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

# Завершим транзакцию

COMMIT;
```

Несмотря на то, что транзакция во 2-ом сеансе не завершена, в ней видны изменения, сделанные завершившейся транзакцией в 1-ом сеансе. Мы получили 2 разных набора строк по одному и тому же условию в одной транзакции. Это аномалия "__Фантомное чтение__", которое допускается на уровне изоляции транзакций "__Read committed__".

В 1-ой сессии начнем новую транзакцию с уровнем изоляции "__Repeatable Read__" и добавим запись в таблицу:
```bash
# Сессия № 1

START TRANSACTION ISOLATION LEVEL REPEATABLE READ;
INSERT INTO persons(first_name, second_name) VALUES('sveta', 'svetova');
SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```

Во 2-ой сессии так же начнем новую транзакцию с уровнем изоляции "__Repeatable Read__" и выполним SELECT:

```bash
# Сессия № 2

START TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

2-ая сессия не видит изменений, сделанных не зафиксированной транзакцией в 1-ой сессии, т.к. "__Грязное чтение__" на уровне изоляции "__Repeatable Read__" не допускается.

Завершим транзакцию в 1-ой сессии:
```bash
# Сессия № 1

COMMIT;
SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```

Выполним SELECT во 2-ой сессии, где транзакция еще не завершена:
```bash
# Сессия № 2

SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

2-ая сессия продолжает видеть все те же данные, что и раньше, она не видит изменений (добавления строки), сделанных уже зафиксированной транзакцией в 1-ой сессии, т.к. "__Фантомное чтение__" на уровне изоляции "__Repeatable Read__" не допускается.

Завершим транзакцию во 2-ой сессии и выполним SELECT:
```bash
COMMIT;
SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```

Теперь во 2-ом сеансе видны изменения, сделанные в 1-ом, т.к. обе транзакции и в 1-ом и во 2-ом сеансе успешно завершены.
