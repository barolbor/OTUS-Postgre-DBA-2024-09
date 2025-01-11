# Домашнее задание
##  Установка и настройка PostgreSQL 
 
 ### Ubuntu 22.04 LTS в WMware на хосте с ОС семейства Windows

Основная трудность, с которой пришлось столкнуться при выполнении ДЗ, это периодическое зависание при загрузке ОС Ubuntu (пробовал от 20 до 24 версии), установленной на виртуальной машине в VirtualBox. Гугл по данной теме рассказывал об известной проблеме VirtualBox на CPU AMD Ryzen 7 7840, которую исправили в версии 7.х.х, установил самую последнюю 7.1.4 - не помогло. К счастью, VMware Workstation Pro стал бесплатным: 

On Nov 11, 2024, Broadcom announced that VMware Desktop Hypervisor (VMware Fusion Pro & VMware Workstation Pro) is available free for Commercial, Educational, and Personal users.  For details on this and subscription options, refer to the following blog post: https://knowledge.broadcom.com/external/article?articleNumber=368667 

Скачиваем и устанавливаем Ubuntu 22.04 на виртуальную машину WMware и настраиваем его:

```bash
# Определяем ip адрес машины и имя сетевого интерфейса

netplan status
     Online state: online
    DNS Addresses: 127.0.0.53 (stub)
       DNS Search: localdomain

●  1: lo ethernet UNKNOWN/UP (unmanaged)
      MAC Address: 00:00:00:00:00:00
        Addresses: 127.0.0.1/8
                   ::1/128
           Routes: ::1 metric 256

●  2: ens33 ethernet UP (networkd: ens33)
      MAC Address: 00:0c:29:e6:6f:a2 (Intel Corporation)
        Addresses: 192.168.145.128/24 (dhcp)
    DNS Addresses: 192.168.145.2
       DNS Search: localdomain
           Routes: default via 192.168.145.2 from 192.168.145.128 metric 100 (dhcp)
                   192.168.145.0/24 from 192.168.145.128 metric 100 (link)
                   192.168.145.2 from 192.168.145.128 metric 100 (dhcp, link)

# Подключаемся к виртуальной машине из терминала Windows

C:\> ssh sa1@192.168.145.128
sa1@192.168.145.128`s password:
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-130-generic x86_64)
...

# Добавляем, используя клипборд, свой открытый ключ, сгенерированный в 1-ом ДЗ, в файл uthorized_keys, чтобы подключаться без пароля

nano ~/.ssh/authorized_keys

# Задаем статический ip адрес

sudo nano /etc/netplan/99-static-ip.yaml

network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      dhcp6: no
      accept-ra: false
      link-local: []
      addresses:
      - 192.168.145.101/24
      routes:
        - to: default
          via: 192.168.145.2
      nameservers:
        addresses:
        - 192.168.145.2
        search:
        - localdomain

# Изменим права на наш конфигурационный файл

sudo chmod 600 /etc/netplan/99-static-ip.yaml

 # Отключаем функцию сетевого конфигурирования cloud-init’s

sudo echo 'network: {config: disabled}' > /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg

# Генерируем файлы конфигурации для бэкэнда

sudo netplan generate

# применяем текущую конфигурацию netplan к работающей системе

sudo netplan apply

# Переподключаемся к VM, с учетом внесенных изменений

C:\> ssh -i  %USERPROFILE%/.ssh/yc_bob_key sa1@192.168.145.101
```

Устанавливаем PostgreSQL 16. Установку будем производить из репозитория с официального сайта.

```bash
# Импортируем ключ подписи репозитория

sudo apt install curl ca-certificates
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc

# Создаем конфигурационный файл репозитория

sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

# Обновляем списки пакетов

sudo apt update

# Устанавливаем 16-ю версию PostgreSQL

sudo apt -y install postgresql-16
```

Проверим что кластер запущен

```bash
sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```

Подключаемся к postgres и создадим тестовую таблицу

```bash
sudo -u postgres psql
psql (16.6 (Ubuntu 16.6-1.pgdg22.04+1))
Type "help" for help.

CREATE TABLE persons(id serial, first_name text, second_name text);
  CREATE TABLE

INSERT INTO persons(first_name, second_name) VALUES('ivan', 'ivanov'), ('petr', 'petrov'), ('sergey', 'sergeev');
  INSERT 0 3

SELECT * FROM persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

\q
```

Создадим новый диск объемом 10Гб в виртуальной машине,

```bash
# Список доступных дисков

sudo fdisk -l | grep "Disk /"
...
Disk /dev/sda: 20 GiB, 21474836480 bytes, 41943040 sectors
Disk /dev/sdb: 10 GiB, 10737418240 bytes, 20971520 sectors

sudo parted -l | grep Error
Error: /dev/sdb: unrecognised disk label
...

# Создаем на диске таблицу разделов gpt 

sudo parted /dev/sdb mklabel gpt
Information: You may need to update /etc/fstab.

# Создаем новый раздел на весь диск

sudo parted -a opt /dev/sdb mkpart pg-db-2 ext4 0% 100%
Information: You may need to update /etc/fstab.

# Отформатируем раздел в файловой системе ext4

sudo mkfs.ext4 -L datapartition /dev/sdb1
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 2620928 4k blocks and 655360 inodes
Filesystem UUID: 5474b505-6faa-4e3e-a48e-d8d1395f10e7
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

# Добавим автомонтирование раздела в /mnt/data

sudo nano /etc/fstab
/dev/disk/by-uuid/5474b505-6faa-4e3e-a48e-d8d1395f10e7 /mnt/data ext4 defaults 0 2

# Перезагрузимся и убедимся, что раздел монтируется при загрузке автоматически

ls -sl /mnt/data/
total 16
16 drwx------ 2 root root 16384 Jan 10 11:15 lost+found

# Сделаем пользователя postrges владельцем /mnt/data

sudo chown -R postgres:postgres /mnt/data/

ls -sl /mnt/data/
total 16
16 drwx------ 2 postgres postgres 16384 Jan 10 11:15 lost+found
```

Остановим PostgreSQL и перенесем его содержимое в /mnt/data/

```bash
sudo -u postgres pg_ctlcluster 16 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@16-main

sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 down   postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log

sudo mv /var/lib/postgresql/16 /mnt/data
```

Попытаемся запустить кластер

```bash
sudo -u postgres pg_ctlcluster 16 main start
Error: /var/lib/postgresql/16/main is not accessible or does not exist
```

Попытка запуска кластера завершилась ошибкой, т.к. мы перенесли каталог с данными в другое место, а PostgreSQL об этом ничего не "сказали".

1. Попробуем решить проблему на уровне ОС, создав символьную ссылку на новое место расположения файлов

```bash
sudo -u postgres ln -s /mnt/data/16 /var/lib/postgresql/

ls -sl /var/lib/postgresql
total 0
0 lrwxrwxrwx 1 postgres postgres 12 Jan 11 10:34 16 -> /mnt/data/16

# Попытаемся запустить кластер

sudo -u postgres pg_ctlcluster 16 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@16-main

# Кластер стартовал, подключимся к нему

sudo -u postgres psql
psql (16.6 (Ubuntu 16.6-1.pgdg22.04+1))
Type "help" for help.

# Убедимся, что наша таблица на месте и содержит данные

postgres=# \dt
          List of relations
 Schema |  Name   | Type  |  Owner
--------+---------+-------+----------
 public | persons | table | postgres
(1 row)

postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

postgres=# \q
```

2. Попробуем решить проблему, задав новое расположение в конфигурационном файле

```bash
# Останавливаем кластер
sudo -u postgres pg_ctlcluster 16 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@16-main

# Удаляем символьную ссылку
sudo rm /var/lib/postgresql/16
ls -sl /var/lib/postgresql
total 0

# Убеждаемся, что кластер не стартует
sudo -u postgres pg_ctlcluster 16 main start
Error: /var/lib/postgresql/16/main is not accessible or does not exist

# Вносим изменения в конфигурационный файл, изменяем параметр data_directory, который задаёт каталог, где хранятся данные
sudo nano /etc/postgresql/16/main/postgresql.conf

data_directory = '/mnt/data/16/main'

# Попытаемся запустить кластер
sudo -u postgres pg_ctlcluster 16 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@16-main

# Убеждаемся, что кластер запущен
pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
16  main    5432 online postgres /mnt/data/16/main /var/log/postgresql/postgresql-16-main.log

# Подключимся
sudo -u postgres psql
psql (16.6 (Ubuntu 16.6-1.pgdg22.04+1))
Type "help" for help.

# Убеждаемся, что наша таблица на месте и содержит данные
postgres=# \dt
          List of relations
 Schema |  Name   | Type  |  Owner
--------+---------+-------+----------
 public | persons | table | postgres
(1 row)

postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)

# Список настроек, содержащих пути в файловой системе
postgres=# SELECT name, setting FROM pg_settings WHERE name LIKE '%file' OR name LIKE '%directory';
        name        |                 setting
--------------------+-----------------------------------------
 config_file        | /etc/postgresql/16/main/postgresql.conf
 data_directory     | /mnt/data/16/main
 external_pid_file  | /var/run/postgresql/16-main.pid
 hba_file           | /etc/postgresql/16/main/pg_hba.conf
 ident_file         | /etc/postgresql/16/main/pg_ident.conf
 krb_server_keyfile | FILE:/etc/postgresql-common/krb5.keytab
 log_directory      | log
 ssl_ca_file        |
 ssl_cert_file      | /etc/ssl/certs/ssl-cert-snakeoil.pem
 ssl_crl_file       |
 ssl_dh_params_file |
 ssl_key_file       | /etc/ssl/private/ssl-cert-snakeoil.key
(12 rows)

postgres=# \q
```

Создадим новую виртуальную машину ubn-srv-2, настрои и установим на нее PostgreSQL, как было описано выше, выключим первую ВМ ubn-srv-1 и подключим ко второй ubn-srv-2 ВМ диск с данными от первой ubn-srv-1

```bash
# Убедимся что диск подключен: интересующий нас раздел мы можем найти по UUID = 5474b505-6faa-4e3e-a48e-d8d1395f10e7 или PARTLABEL = pg-db-2
sa2@ubn-srv-2:~$ sudo lsblk -o PATH,SIZE,RO,TYPE,MOUNTPOINT,PARTLABEL,UUID
PATH                               SIZE RO TYPE MOUNTPOINT        PARTLABEL UUID
/dev/loop0                        63.9M  1 loop /snap/core20/2318
/dev/loop1                          87M  1 loop /snap/lxd/29351
/dev/loop2                        38.8M  1 loop /snap/snapd/21759
/dev/sda                            20G  0 disk
/dev/sda1                            1M  0 part
/dev/sda2                          1.8G  0 part /boot                       2325fd8a-e7c0-42c2-b87f-c67274400626
/dev/sda3                         18.2G  0 part                             91Gq5B-h4Ma-myvV-IlS5-fPZE-fqfY-iILpge
/dev/sdb                            10G  0 disk
/dev/sdb1                           10G  0 part                   pg-db-2   5474b505-6faa-4e3e-a48e-d8d1395f10e7
/dev/sr0                             2G  0 rom                              2024-09-11-18-46-48-00
/dev/mapper/ubuntu--vg-ubuntu--lv   10G  0 lvm  /                           044ce2a3-4ec8-46ad-a2b9-bae0f049ae24

# Добавим автомонтирование раздела в /mnt/data
sa2@ubn-srv-2:~$ sudo nano /etc/fstab

/dev/disk/by-uuid/5474b505-6faa-4e3e-a48e-d8d1395f10e7 /mnt/data ext4 defaults 0 2

# Перезагрузимся и убедимся, что раздел монтируется при загрузке автоматически
sa2@ubn-srv-2:~$ sudo lsblk -o PATH,SIZE,RO,TYPE,MOUNTPOINT,PARTLABEL,UUID /dev/sdb1
PATH      SIZE RO TYPE MOUNTPOINT PARTLABEL UUID
/dev/sdb1  10G  0 part /mnt/data  pg-db-2   5474b505-6faa-4e3e-a48e-d8d1395f10e7

# Останавливаем кластер
sa2@ubn-srv-2:~$ sudo systemctl stop postgresql@16-main
sa2@ubn-srv-2:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 down   postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log

# Удалим файлы с данными из /var/lib/postgres
sudo rm -r /var/lib/postgresql/*

# Вносим изменения в конфигурационный файл, изменяем параметр data_directory, который задаёт каталог, где хранятся данные
sa2@ubn-srv-2:~$ sudo nano /etc/postgresql/16/main/postgresql.conf

data_directory = '/mnt/data/16/main'

# Попытаемся запустить кластер
sa2@ubn-srv-2:~$ sudo systemctl start postgresql@16-main

# Убеждаемся, что кластер запущен
sa2@ubn-srv-2:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
16  main    5432 online postgres /mnt/data/16/main /var/log/postgresql/postgresql-16-main.log

# Подключимся
sa2@ubn-srv-2:~$ sudo -u postgres psql
psql (16.6 (Ubuntu 16.6-1.pgdg22.04+1))
Type "help" for help.

# Убеждаемся, что наша таблица на месте и содержит данные
postgres=# \dt
          List of relations
 Schema |  Name   | Type  |  Owner
--------+---------+-------+----------
 public | persons | table | postgres
(1 row)

postgres=# postgres=# select * from persons;
ERROR:  syntax error at or near "postgres"
LINE 1: postgres=# select * from persons;
        ^
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

Как мы видим, для того чтобы перенести данные от одного инстанса PostgreSQL к другому на другой машине, достаточно проделать те же самые манипуляции, как и при переносе данных в другой каталог. Единственное что при этом мы не делали, так это не давали права пользователю и группе postgres, т.к. у них не изменились UID и GID.

```bash
id postgres
uid=114(postgres) gid=120(postgres) groups=120(postgres),119(ssl-cert)

# group_name:password:GID:user_list
getent group postgres
postgres:x:120:
```
В большинстве дистрибутивов Linux UID 1-500 обычно зарезервирован для системных пользователей. 

Наверное, UID и GUI для пользователя и группы postgres фиксированы в пределах версии и ОС.
