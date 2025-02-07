# Домашнее задание
## Работа с join'ами, статистикой
 
ДЗ выполняется на ВМ в WMware (1 процессор, 2 ядра, 4Гб ОЗУ и 40Гб SSD диска), кластер PostgreSQL 16 на Ubuntu 22.04 LTS.

Создадим БД для учета посещаемости тренировок игроками хоккейной команды

```sql
CREATE DATABASE hockey;

-- Таблица игроков
CREATE TABLE players(
    player_id SERIAL PRIMARY KEY,
    fio char(20)
);

-- Стадионы
CREATE TABLE stadiums(
    stadium_id SERIAL PRIMARY KEY,
    name char(10)
);

-- Тренировки
CREATE TABLE trainings(
     training_id SERIAL PRIMARY KEY,
     stadium_id  integer,
     date_time timestamp ,
     FOREIGN KEY (stadium_id) REFERENCES stadiums(stadium_id),
     CONSTRAINT trainings_date_time_stadium_uniq UNIQUE (date_time, stadium_id)
);
-- Учет посещаемости тренировок
CREATE TABLE training_attend(
    training_id integer,
    player_id integer,
    FOREIGN KEY (training_id) REFERENCES trainings(training_id),
    FOREIGN KEY (player_id) REFERENCES players(player_id),
    CONSTRAINT training_attend_uniq UNIQUE (training_id, player_id)
);

-- Наполним данными
INSERT INTO players(fio) VALUES('Mihajlov'),('Petrov'),('Harlamov'),('Vasilev'),('Gusev');
INSERT INTO stadiums(name) VALUES('CSKA'),('Dinamo'),('MSA'),('Spartak'),('Torpedo');
INSERT INTO trainings(stadium_id,date_time) VALUES(1,'1967-09-02 08:00'),(2,'1967-09-04 16:00'),(3,'1967-09-06 19:00') ;
INSERT INTO training_attend(training_id,player_id) SELECT generate_series(1,3), c FROM (SELECT generate_series(1,5) AS c);
DELETE FROM training_attend WHERE (training_id=2 AND player_id=4)OR(training_id=3 AND player_id=5);
```

Реализуем прямое соединение двух или более таблиц. Посмотрим расписание тренировок

```sql
SELECT s.name, t.date_time FROM trainings AS t JOIN stadiums AS s ON s.stadium_id=t.stadium_id;
    name    |      date_time
------------+---------------------
 CSKA       | 1967-09-02 08:00:00
 Dinamo     | 1967-09-04 16:00:00
 MSA        | 1967-09-06 19:00:00
(3 rows)
```

Реализуем левостороннее (или правостороннее) соединение двух или более таблиц. Выберем те арены, на которых не было тренировок

```sql
-- Левостороннее
SELECT s.* FROM stadiums AS s LEFT JOIN trainings AS t ON t.stadium_id = s.stadium_id WHERE t.stadium_id IS NULL;

 stadium_id |    name
------------+------------
          4 | Spartak
          5 | Torpedo
(2 rows)

-- Правостороннее
SELECT s.name, t.* FROM trainings AS t RIGHT JOIN stadiums AS s ON s.stadium_id = t.stadium_id;

    name    | training_id | stadium_id |      date_time
------------+-------------+------------+---------------------
 CSKA       |           1 |          1 | 1967-09-02 08:00:00
 Dinamo     |           2 |          2 | 1967-09-04 16:00:00
 MSA        |           3 |          3 | 1967-09-06 19:00:00
 Torpedo    |             |            |
 Spartak    |             |            |
(5 rows)

-- Или более наглядно

-- Для левостороннего
SELECT * FROM stadiums AS s LEFT JOIN trainings AS t ON t.stadium_id = s.stadium_id;
 stadium_id |    name    | training_id | stadium_id |      date_time
------------+------------+-------------+------------+---------------------
          1 | CSKA       |           1 |          1 | 1967-09-02 08:00:00
          2 | Dinamo     |           2 |          2 | 1967-09-04 16:00:00
          3 | MSA        |           3 |          3 | 1967-09-06 19:00:00
          5 | Torpedo    |             |            |
          4 | Spartak    |             |            |
(5 rows)

-- Для правостороннего
SELECT * FROM trainings AS t RIGHT JOIN stadiums AS s ON s.stadium_id = t.stadium_id;
 training_id | stadium_id |      date_time      | stadium_id |    name
-------------+------------+---------------------+------------+------------
           1 |          1 | 1967-09-02 08:00:00 |          1 | CSKA
           2 |          2 | 1967-09-04 16:00:00 |          2 | Dinamo
           3 |          3 | 1967-09-06 19:00:00 |          3 | MSA
             |            |                     |          5 | Torpedo
             |            |                     |          4 | Spartak
(5 rows)
```

Реализуем кросс соединение двух или более таблиц. Получим набор данных с идеальной 100% посещаемостью, которую мы хотели бы видеть.

```sql
SELECT 
	p.*, t.*, s.name
FROM
	players AS p 
	CROSS JOIN trainings AS t 
	JOIN stadiums AS s ON s.stadium_id =t.stadium_id
ORDER BY 
	t.date_time, p.fio ;

 player_id |         fio          | training_id | stadium_id |      date_time      |    name
-----------+----------------------+-------------+------------+---------------------+------------
         5 | Gusev                |           1 |          1 | 1967-09-02 08:00:00 | CSKA
         3 | Harlamov             |           1 |          1 | 1967-09-02 08:00:00 | CSKA
         1 | Mihajlov             |           1 |          1 | 1967-09-02 08:00:00 | CSKA
         2 | Petrov               |           1 |          1 | 1967-09-02 08:00:00 | CSKA
         4 | Vasilev              |           1 |          1 | 1967-09-02 08:00:00 | CSKA
         5 | Gusev                |           2 |          2 | 1967-09-04 16:00:00 | Dinamo
         3 | Harlamov             |           2 |          2 | 1967-09-04 16:00:00 | Dinamo
         1 | Mihajlov             |           2 |          2 | 1967-09-04 16:00:00 | Dinamo
         2 | Petrov               |           2 |          2 | 1967-09-04 16:00:00 | Dinamo
         4 | Vasilev              |           2 |          2 | 1967-09-04 16:00:00 | Dinamo
         5 | Gusev                |           3 |          3 | 1967-09-06 19:00:00 | MSA
         3 | Harlamov             |           3 |          3 | 1967-09-06 19:00:00 | MSA
         1 | Mihajlov             |           3 |          3 | 1967-09-06 19:00:00 | MSA
         2 | Petrov               |           3 |          3 | 1967-09-06 19:00:00 | MSA
         4 | Vasilev              |           3 |          3 | 1967-09-06 19:00:00 | MSA
(15 rows)    

```

Реализуем полное соединение двух или более таблиц. Выведем полное расписание, включая стадионы на которых нет тренировок и включая тренировки на еще "отсутствующих" стадионах

```sql
-- Для демонстрации полного соединения нужно удалить ограничение и добавить ссылки на не существующие стадионы
ALTER TABLE trainings DROP CONSTRAINT trainings_stadium_id_fkey;
INSERT INTO trainings(stadium_id, date_time) VALUES (6, '2024-12-31 23:59'), (6, '2024-01-01 06:00');
INSERT 0 2

-- Полное соединение
SELECT * FROM stadiums s FULL JOIN trainings t ON s.stadium_id =t.stadium_id ORDER BY s.name;
 stadium_id |    name    | training_id | stadium_id |      date_time
------------+------------+-------------+------------+---------------------
          1 | CSKA       |           1 |          1 | 1967-09-02 08:00:00
          2 | Dinamo     |           2 |          2 | 1967-09-04 16:00:00
          3 | MSA        |           3 |          3 | 1967-09-06 19:00:00
          4 | Spartak    |             |            |
          5 | Torpedo    |             |            |
            |            |           5 |          6 | 2024-12-31 23:59:00
            |            |           6 |          6 | 2024-01-01 06:00:00
(7 rows)

-- удалим тренировки на не существующих стадионах
DELETE FROM trainings AS t WHERE t.stadium_id NOT IN (SELECT s.stadium_id FROM stadiums AS s);
DELETE 2

-- восстановим ограничение
ALTER TABLE trainings ADD CONSTRAINT trainings_stadium_id_fkey FOREIGN KEY (stadium_id) REFERENCES stadiums(stadium_id);
ALTER TABLE
```

Реализуем запрос, в котором будут использованы разные типы соединений, получим полную картину посещаемости.

```sql
SELECT
	p.fio, t.date_time, s.name,
	CASE  
		WHEN ta.player_id IS NULL THEN 'No'
		ELSE 'YES'
	END AS attended
FROM
	trainings AS t
	JOIN stadiums AS s ON s.stadium_id =t.stadium_id 
	CROSS JOIN players AS p  
	LEFT JOIN training_attend AS ta ON ta.player_id = p.player_id AND ta.training_id = t.training_id 
ORDER BY 
	t.date_time, p.fio;

         fio          |      date_time      |    name    | attended
----------------------+---------------------+------------+----------
 Gusev                | 1967-09-02 08:00:00 | CSKA       | YES
 Harlamov             | 1967-09-02 08:00:00 | CSKA       | YES
 Mihajlov             | 1967-09-02 08:00:00 | CSKA       | YES
 Petrov               | 1967-09-02 08:00:00 | CSKA       | YES
 Vasilev              | 1967-09-02 08:00:00 | CSKA       | YES
 Gusev                | 1967-09-04 16:00:00 | Dinamo     | YES
 Harlamov             | 1967-09-04 16:00:00 | Dinamo     | YES
 Mihajlov             | 1967-09-04 16:00:00 | Dinamo     | YES
 Petrov               | 1967-09-04 16:00:00 | Dinamo     | YES
 Vasilev              | 1967-09-04 16:00:00 | Dinamo     | No
 Gusev                | 1967-09-06 19:00:00 | MSA        | No
 Harlamov             | 1967-09-06 19:00:00 | MSA        | YES
 Mihajlov             | 1967-09-06 19:00:00 | MSA        | YES
 Petrov               | 1967-09-06 19:00:00 | MSA        | YES
 Vasilev              | 1967-09-06 19:00:00 | MSA        | YES
(15 rows)    
```

*****

Задание со звездочкой*

Придумайте 3 своих метрики на основе показанных представлений, отправьте их через ЛК, а так же поделитесь с коллегами в слаке. ???

1. я метрика: общий % посещаемости тренировок:
```sql
SELECT (((SELECT count(*) FROM training_attend ta)::float  / (SELECT count(*) FROM players p CROSS JOIN trainings)::float)*100)::NUMERIC(4,2)||'%';
 ?column?
----------
 86.67%
(1 row)
```

2. я метрика: общий % пропусков тренировок 
```sql
SELECT ((1-(SELECT count(*) FROM training_attend ta)::float  / (SELECT count(*) FROM players p CROSS JOIN trainings)::float)*100)::NUMERIC(4,2)||'%';
 ?column?
----------
 13.33%
(1 row)
```

3. я метрика: Среднее количество игроков на тренировке
```sql
SELECT ((SELECT count(*) FROM training_attend ta)::float / (SELECT count(*) FROM trainings t)::float)::NUMERIC(4,2);
 numeric
---------
    4.33
(1 row)

-- 4 хоккеиста и чьи-то ноги :)
```

