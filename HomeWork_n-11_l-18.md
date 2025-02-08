# Домашнее задание
## Триггеры, поддержка заполнения витрин
 
ДЗ выполняется на ВМ в WMware (1 процессор, 2 ядра, 4Гб ОЗУ и 40Гб SSD диска), кластер PostgreSQL 16 на Ubuntu 22.04 LTS.

Создаем структуру для выполнения ДЗ

```sql
CREATE DATABASE newdb;

\c newdb

DROP SCHEMA IF EXISTS pract_functions CASCADE;
CREATE SCHEMA pract_functions;

--SET search_path = pract_functions, public;
ALTER DATABASE newdb SET search_path = pract_functions, pg_catalog;

-- товары:
CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);
INSERT INTO goods (goods_id, good_name, good_price)
VALUES 	(1, 'Спички хозайственные', .50),
		(2, 'Автомобиль Ferrari FXX K', 185000000.01);

-- Продажи
CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);

INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);

-- отчет:
SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;

-- с увеличением объёма данных отчет стал создаваться медленно
-- Принято решение денормализовать БД, создать таблицу
CREATE TABLE good_sum_mart
(
	good_name   varchar(63) NOT NULL,
	sum_sale	numeric(16, 2)NOT NULL
);
```

Создадим триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии (вычисляющий при каждой продаже сумму и записывающий её в витрину)

```sql
-- Создадим триггерную функцию

CREATE OR REPLACE FUNCTION sales_after_change()
RETURNS TRIGGER AS $$
BEGIN
/*
 TG_OP  | OLD  | NEW  |
--------+------+-------
 DELETE | OLD  | NULL |
 UPDATE | OLD  | NEW  |
 INSERT | NULL | NEW  | 

sales: sales_id, good_id, sales_qty, sales_time
god_sum_mart: good_name, sum_sale
*/
	IF TG_OP IN ('DELETE', 'UPDATE') THEN
		UPDATE
			good_sum_mart
		SET 
			-- А тут может быть и ноль и отрицательные значения при повышении цены
			sum_sale = 
			sum_sale -
			(
				SELECT 
					OLD.sales_qty * g.good_price
				FROM 
					goods g
				WHERE 
					g.goods_id = OLD.good_id
			)
		WHERE 
			good_name = (SELECT g.good_name FROM goods AS g WHERE g.goods_id = OLD.good_id);		
	END IF;

	IF TG_OP IN ('UPDATE', 'INSERT') THEN
		MERGE INTO 
			good_sum_mart AS gsm
		USING (
			SELECT 
				 good_name
				,good_price
			FROM
				goods
			WHERE
				goods_id = NEW.good_id
		) AS g
		ON 
			gsm.good_name = g.good_name
		WHEN MATCHED THEN
			UPDATE SET sum_sale = sum_sale + g.good_price * NEW.sales_qty
		WHEN NOT MATCHED THEN 
			INSERT (good_name, sum_sale) VALUES (g.good_name, g.good_price * NEW.sales_qty);
	END IF;

--	RAISE NOTICE 'Триггер сработал на: %, sales_id old=% new=%,  ',  TG_OP, OLD.sales_id, NEW.sales_id;
	
	RETURN null;
END;
$$ LANGUAGE plpgsql;


-- Создадим триггер
CREATE TRIGGER sales_after_change_trigger AFTER INSERT OR UPDATE OR DELETE ON sales
FOR EACH ROW
EXECUTE PROCEDURE sales_after_change();
```

В реальной жизни теперь мы должны были бы наполнить нашу витрину данными, загнать отчет в нее

```sql
INSERT INTO good_sum_mart
    -- отчет:
    SELECT G.good_name, sum(G.good_price * S.sales_qty)
    FROM goods G
    INNER JOIN sales S ON S.good_id = G.goods_id
    GROUP BY G.good_name;
```

Но в ДЗ поступим по другому, перезальем продажи заново, заодно и триггер проверим на вставку.

```sql
TRUNCATE sales;
INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);

-- Проверим что выдает отчет и витрина
SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name
ORDER BY G.good_name;
        good_name         |     sum
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)

SELECT * FROM good_sum_mart gsm ORDER BY gsm.good_name ;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)

-- и еще пару манипуляций с продажами, чтобы проверить обновление и удаление

INSERT INTO goods (goods_id, good_name, good_price) VALUES 	(3, 'Ручка', 10) ;

-- Продадим 2+3+5=10 ручек по 10р. итого 100р.

INSERT INTO sales (good_id, sales_qty) VALUES (3, 2), (3, 3), (3, 5);
SELECT * FROM sales;
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
       18 |       1 | 2025-02-08 14:06:05.042352+03 |        10
       19 |       1 | 2025-02-08 14:06:05.042352+03 |         1
       20 |       1 | 2025-02-08 14:06:05.042352+03 |       120
       21 |       2 | 2025-02-08 14:06:05.042352+03 |         1
       22 |       3 | 2025-02-08 14:15:05.686392+03 |         2
       23 |       3 | 2025-02-08 14:15:05.686392+03 |         3
       24 |       3 | 2025-02-08 14:15:05.686392+03 |         5
(7 rows)

-- Отменим продажу 2х ручек (возврат 2х ручек), и изменим продажу 3х на 15: 0+15+5=20 по 10р. итого 200 
DELETE FROM sales WHERE sales_id = 23;
UPDATE sales SET sales_qty = 15 WHERE sales_id = 22;

-- Проверим витрину
SELECT * FROM good_sum_mart gsm ORDER BY gsm.good_name ;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Ручка                    |       200.00
 Спички хозайственные     |        65.50
(3 rows)

-- один из очевидных недостатков таблицы sales - отсутствие цены продажи, как следствие, уход в минус при повышении цены. повысим стоимость ручек до 20р. и произведем возврат 14 ручек из 15, (0+15+5)*10 - 15*20 + 1*20 = 200 - 300 + 20 = -80р.
UPDATE goods SET good_price=20 WHERE goods_id = 3;
UPDATE sales SET sales_qty = 1 WHERE sales_id = 22;
SELECT * FROM good_sum_mart gsm ORDER BY gsm.good_name ;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Ручка                    |       -80.00
 Спички хозайственные     |        65.50
(3 rows)

-- баг нашего триггера, а точнее не баг, а особенность работы программы :) , нулевые продажи, из-за ограничение на sales_qty > 0 данная особенность возникает только при возврате всех проданных товаров, причем цена продажи и возврата должна совпадать, в противном случае мы получим отрицательные продажи в витрине, при отсутствии каких либо продаж в таблице sales
DELETE FROM sales WHERE good_id = 3;
SELECT * FROM good_sum_mart gsm ORDER BY gsm.good_name ;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Ручка                    |      -200.00
 Спички хозайственные     |        65.50
(3 rows)

SELECT s.*, g.good_name, g.good_price FROM sales AS s JOIN goods AS g ON g.goods_id = s.good_id;
 sales_id | good_id |          sales_time           | sales_qty |        good_name         |  good_price
----------+---------+-------------------------------+-----------+--------------------------+--------------
       18 |       1 | 2025-02-08 14:06:05.042352+03 |        10 | Спички хозайственные     |         0.50
       19 |       1 | 2025-02-08 14:06:05.042352+03 |         1 | Спички хозайственные     |         0.50
       20 |       1 | 2025-02-08 14:06:05.042352+03 |       120 | Спички хозайственные     |         0.50
       21 |       2 | 2025-02-08 14:06:05.042352+03 |         1 | Автомобиль Ferrari FXX K | 185000000.01
(4 rows)

-- Теперь придется перезаливать витрину. Как вариант, чтобы избежать данной ситуации, нужно каждый раз пересчитывать витрину по конкретному товару, а в случае с обновлением, если меняется в продаже товар, то пересчет нужен по двум товарам - старому и новому. А теперь вспомним, зачем мы делали витрину: "отчет стал создаваться медленно", так вот, эти тормоза мы можем перенести в триггер, хоть и не в полном объеме, при постоянном пересчете.  Итого: не правильно спроектированная база приводит к большим проблемам.
```

Сама по себе идея создания витрины, а здесь это не что иное как регистр накопления в терминологии 1С, хороша. Но нужно 
правильно проектировать БД. В таблице sales, как минимум, отсутствует поле good_price, проблемы, связанные с его отсутствием, были продемонстрированы выше. Продажа - факт свершившийся, причем мы продали конкретный товар, по конкретной цене, о чем есть документы строгой отчетности, как-то фискальный чек, счет фактура, поэтому эти позиции надо четко фиксировать. Карточка товара должна содержать историю изменения наименования товара, замены артикулов и т.д. и т.п. плюс связана с прайсами закупками и....  

И еще, в витрине бы нужно добавить/(заменить good_name на) goods_id. А то в sales фигурирует goods_id (который обозван как good_id без S), а в витрине good_name, изменим имя в таблице goods и опять будет весело! И поиск по PKEY будет в разы быстрее, чем по строке да еще без индекса, опять же лишний тормоз в триггер.

Итого: goods_price в sales и goods_id в витрину и будет маленькое счастье.

*****

ps: в файле с ДЗ замечены следующие опечатки:
* Строка 6: "SET search_path = pract_functions, publ", наверное имелось ввиду public
* Строка 15: "VALUES 	(1, 'Спички хозайственные', .50)," а вместо я в слове "хозяйственные" 
* По тексту: "good_" пропущена буква S в goods_
