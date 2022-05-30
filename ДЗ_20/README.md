Описание/Пошаговая инструкция выполнения домашнего задания:
Секционировать большую таблицу из демо базы flights

```sql

--для секционирования выбрана таблица bookings

--
CREATE TABLE bookings.bookings (
	book_ref bpchar(6) NOT NULL,
	book_date timestamptz NOT NULL,
	total_amount numeric(10, 2) NOT NULL
);
CREATE INDEX bookings_book_ref_idx ON bookings.bookings USING btree (book_ref);

--план запроса без секций
explain
select *
from table1
where create_date between date'2020-01-01' and date'2020-02-01'-1;

QUERY PLAN                                                                               |
-----------------------------------------------------------------------------------------+
Gather  (cost=1000.00..27641.54 rows=1 width=21)                                         |
  Workers Planned: 2                                                                     |
  ->  Parallel Seq Scan on bookings  (cost=0.00..26641.44 rows=1 width=21)               |
        Filter: ((book_date >= '2016-07-01'::date) AND (book_date <= '2016-06-30'::date))|


--секционируем таблицу по диапазону, поле секционирования book_date
--drop table bookings.bookings_part;
create table bookings.bookings_part (
	book_ref bpchar(6) NOT NULL,
	book_date timestamptz NOT NULL,
	total_amount numeric(10, 2) NOT NULL
)
partition by range (book_date);

--нарежем секции таблиц по годам, и создадим секцию с default
create table bookings.bookings_part_2016 partition of bookings.bookings_part for values from ('2016-01-01') to ('2016-12-31');
create table bookings.bookings_part_2017 partition of bookings.bookings_part for values from ('2017-01-01') to ('2017-12-31');
create table bookings.bookings_part_2018 partition of bookings.bookings_part for values from ('2018-01-01') to ('2018-12-31');
create table bookings.bookings_part_2019 partition of bookings.bookings_part for values from ('2019-01-01') to ('2019-12-31');
create table bookings.bookings_part_2020 partition of bookings.bookings_part for values from ('2020-01-01') to ('2020-12-31');
create table bookings.bookings_part_2021 partition of bookings.bookings_part for values from ('2021-01-01') to ('2021-12-31');
create table bookings.bookings_part_default partition of bookings.bookings_part default;

--произведём вставку в таблицу
insert into bookings.bookings_part select * from bookings.bookings

--проверим вставку записей в секции таблицы
insert into bookings.bookings_part select '00002D', timestamp '2018-07-05 05:12:00.000 +0500', 777; --bookings_part_2018
insert into bookings.bookings_part select '111111', timestamp '2019-07-05 05:12:00.000 +0500', 888; --bookings_part_2019
insert into bookings.bookings_part select '222', timestamp '2020-07-05 05:12:00.000 +0500', 888; --bookings_part_2020
insert into bookings.bookings_part select '3333', timestamp '2021-07-05 05:12:00.000 +0500', 888; --bookings_part_2021
insert into bookings.bookings_part select '4444', timestamp '2022-07-05 05:12:00.000 +0500', 22222; --default

explain
select *
from bookings.bookings_part
where book_date between '2016-01-01 05:12:00.000' and '2016-12-05 05:12:00.000'; --bookings_part_2016

QUERY PLAN                                                                                                                                         |
---------------------------------------------------------------------------------------------------------------------------------------------------+
Seq Scan on bookings_part_2016 bookings_part  (cost=0.00..18086.50 rows=704242 width=21)                                                           |
  Filter: ((book_date >= '2016-01-01 05:12:00+05'::timestamp with time zone) AND (book_date <= '2016-12-05 05:12:00+05'::timestamp with time zone))|

explain
select *
from bookings.bookings_part
where book_date between '2021-01-01 05:12:00.000' and '2021-12-05 05:12:00.000'; --bookings_part_2021

QUERY PLAN                                                                                                                                         |
---------------------------------------------------------------------------------------------------------------------------------------------------+
Seq Scan on bookings_part_2021 bookings_part  (cost=0.00..25.30 rows=5 width=52)                                                                   |
  Filter: ((book_date >= '2021-01-01 05:12:00+05'::timestamp with time zone) AND (book_date <= '2021-12-05 05:12:00+05'::timestamp with time zone))|

explain
select *
from bookings.bookings_part
where book_date between '2022-01-01 05:12:00.000' and '2022-12-05 05:12:00.000'; --bookings_part_default

QUERY PLAN                                                                                                                                         
---------------------------------------------------------------------------------------------------------------------------------------------------
Seq Scan on bookings_part_default bookings_part  (cost=0.00..116.02 rows=1 width=21)                                                               
  Filter: ((book_date >= '2022-01-01 05:12:00+05'::timestamp with time zone) AND (book_date <= '2022-12-05 05:12:00+05'::timestamp with time zone))

```
