Цель:
 Создать триггер для поддержки витрины в актуальном состоянии.

Описание/Пошаговая инструкция выполнения домашнего задания:
Скрипт и развернутое описание задачи – в ЛК (файл hw_triggers.sql) или по ссылке: https://disk.yandex.ru/d/l70AvknAepIJXQ
В БД создана структура, описывающая товары (таблица goods) и продажи (таблица sales).
Есть запрос для генерации отчета – сумма продаж по каждому товару.
БД была денормализована, создана таблица (витрина), структура которой повторяет структуру отчета.
Создать триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии (вычисляющий при каждой продаже сумму и записывающий её в витрину)
Подсказка: не забыть, что кроме INSERT есть еще UPDATE и DELETE

*Развернул предлеженые структуры*
```sql
DROP SCHEMA IF EXISTS pract_functions CASCADE;
CREATE SCHEMA pract_functions;

SET search_path = pract_functions, publ

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

CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);        

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

--создадим PK на good_name
ALTER TABLE good_sum_mart ADD PRIMARY KEY (good_name);

--Добавил тригерную функцию
CREATE OR REPLACE FUNCTION good_sum_mart() RETURNS TRIGGER AS $$
 DECLARE
  v_qty     integer;
  v_good_id integer;
 begin
	 
   IF (TG_OP = 'INSERT') THEN
     v_qty = NEW.sales_qty;
     v_good_id = NEW.good_id; 
   ELSIF (TG_OP = 'UPDATE') THEN
     v_qty = NEW.sales_qty - OLD.sales_qty;
     v_good_id = OLD.good_id;
   ELSIF (TG_OP = 'DELETE') THEN
     v_qty = 0 - OLD.sales_qty;
     v_good_id = OLD.good_id;
   END IF;

  INSERT INTO good_sum_mart (good_name, sum_sale)
    SELECT good_name , good_price * v_qty
      FROM goods WHERE goods_id = v_good_id
  ON CONFLICT ON CONSTRAINT good_sum_mart_pkey
    DO UPDATE SET sum_sale = good_sum_mart.sum_sale + EXCLUDED.sum_sale
        WHERE good_sum_mart.good_name = EXCLUDED.good_name;

    RETURN NULL;
    END;
$$ LANGUAGE plpgsql;

--создадим триггер
CREATE TRIGGER good_sum_mart
AFTER INSERT OR UPDATE OR DELETE ON sales
FOR EACH ROW EXECUTE FUNCTION good_sum_mart();

--добавим продажи
INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);

    --проверим
    -- отчет:
    SELECT G.good_name, sum(G.good_price * S.sales_qty)
    FROM goods G
    INNER JOIN sales S ON S.good_id = G.goods_id
    GROUP BY G.good_name;

    good_name               |sum         |
    ------------------------+------------+
    Автомобиль Ferrari FXX K|185000000.01|
    Спички хозайственные    |       65.50|

    select * from good_sum_mart

    good_name               |sum_sale    |
    ------------------------+------------+
    Автомобиль Ferrari FXX K|185000000.01|
    Спички хозайственные    |       65.50|

--обновим
update sales set sales_qty = 1 where sales_id = 2;

    --отчёт
    good_name               |sum         |
    ------------------------+------------+
    Автомобиль Ferrari FXX K|185000000.01|
    Спички хозайственные    |      560.50|
    --good_sum_mart
    good_name               |sum_sale    |
    ------------------------+------------+
    Автомобиль Ferrari FXX K|185000000.01|
    Спички хозайственные    |      560.50|

--удалим продажу    
delete from sales where sales_id = 8; --Автомобиль Ferrari FXX K

    --отчёт
    good_name           |sum   |
    --------------------+------+
    Спички хозайственные|560.50|

    --good_sum_mart
    good_name               |sum_sale|
    ------------------------+--------+
    Спички хозайственные    |  560.50|
    Автомобиль Ferrari FXX K|    0.00|

--добавим новый товар
    INSERT INTO goods (goods_id, good_name, good_price)
    VALUES 	(3, 'Автомобиль Ford', 1000000)

    --добавим продажи по новому товару
    INSERT INTO sales (good_id, sales_qty) VALUES (3, 2);

    --изменим цену у нового товара
    update goods set good_price = 500000 where goods_id = 3;

    --отчёт !!!цена в отчёте стала не корректной
    good_name               |sum         |
    ------------------------+------------+
    Автомобиль Ferrari FXX K|185000000.01|
    Спички хозайственные    |      626.00|
    Автомобиль Ford         |  1000000.00|

    select * from good_sum_mart --в таблице всё верно

    good_name               |sum_sale    |
    ------------------------+------------+
    Спички хозайственные    |      626.00|
    Автомобиль Ferrari FXX K|185000000.01|
    Автомобиль Ford         |  2000000.00|

    --добвим ещё одну продажу
    INSERT INTO sales (good_id, sales_qty) VALUES (3, 2);
    --отчёт
    good_name               |sum         |
    ------------------------+------------+
    Автомобиль Ferrari FXX K|185000000.01|
    Спички хозайственные    |      626.00|
    Автомобиль Ford         |  2000000.00| 

    select * from good_sum_mart
    good_name               |sum_sale    |
    ------------------------+------------+
    Спички хозайственные    |      626.00|
    Автомобиль Ferrari FXX K|185000000.01|
    Автомобиль Ford         |  3000000.00|

    --расхождения в отчёте и в таблице
```

Задание со звездочкой*
Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?
Подсказка: В реальной жизни возможны изменения цен.

*При изменении цены товара отчёт выдаёт неверные данные, а в таблице good_sum_mart осталось верное отображение*