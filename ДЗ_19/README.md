1 вариант:
Создать индексы на БД, которые ускорят доступ к данным.
В данном задании тренируются навыки:
определения узких мест
написания запросов для создания индекса
оптимизации 
```sql

Необходимо:
`Создать индекс к какой-либо из таблиц вашей БД`

CREATE INDEX tickets_ticket_no_idx ON bookings.tickets (ticket_no);

`Прислать текстом результат команды explain, в которой используется данный индекс`

QUERY PLAN                                             
-------------------------------------------------------
Index Scan using tickets_ticket_no_idx on tickets t  (c
  Index Cond: (ticket_no = '0005432000297'::bpchar)    

`Реализовать индекс для полнотекстового поиска`

--добваляем колонку с tsvector
alter table bookings.airports_data add column some_text_lexeme tsvector;

--заполняем созданную колонку
update bookings.airports_data
set some_text_lexeme = to_tsvector(timezone);

--создаём полнотекстовой индекс GIN
CREATE INDEX search_index_ord ON bookings.airports_data USING GIN (some_text_lexeme);

QUERY PLAN                                                                   |
-----------------------------------------------------------------------------+
Bitmap Heap Scan on airports_data  (cost=8.26..12.52 rows=1 width=172)       |
  Recheck Cond: (some_text_lexeme @@ to_tsquery('asia/magadan'::text))       |
  ->  Bitmap Index Scan on search_index_ord  (cost=0.00..8.26 rows=1 width=0)|
        Index Cond: (some_text_lexeme @@ to_tsquery('asia/magadan'::text))   |

`Реализовать индекс на часть таблицы или индекс на поле с функцией`

--создаём функциональный индекс
CREATE INDEX flights_flight_no_idx ON bookings.flights(lower(flight_no));

explain
select * from bookings.flights t where lower(t.flight_no) = lower('PG0210')

QUERY PLAN                                                                            |
--------------------------------------------------------------------------------------+
Bitmap Heap Scan on flights t  (cost=12.62..2039.19 rows=1074 width=63)               |
  Recheck Cond: (lower((flight_no)::text) = 'pg0210'::text)                           |
  ->  Bitmap Index Scan on flights_flight_no_idx  (cost=0.00..12.35 rows=1074 width=0)|
        Index Cond: (lower((flight_no)::text) = 'pg0210'::text)                       |

--индес на часть таблицы
CREATE INDEX flights_flight_id_idx ON bookings.flights(flight_id) where flight_id < 100000;

--используется
explain
select * from bookings.flights t where flight_id = 71914   

QUERY PLAN                                                                            |
--------------------------------------------------------------------------------------+
Index Scan using flights_flight_id_idx on flights t  (cost=0.29..8.31 rows=1 width=63)|
  Index Cond: (flight_id = 71914)                                                     |

--не используется
explain
select * from bookings.flights t where flight_id = 100667

QUERY PLAN                                                                |
--------------------------------------------------------------------------+
Gather  (cost=1000.00..5204.00 rows=1 width=63)                           |
  Workers Planned: 1                                                      |
  ->  Parallel Seq Scan on flights t  (cost=0.00..4203.90 rows=1 width=63)|
        Filter: (flight_id = 100667)                                      |

`Создать индекс на несколько полей`

create index idx_dep_arr on bookings.flights(departure_airport, arrival_airport);

explain
select * from bookings.flights t where departure_airport = 'VOG' and arrival_airport = 'ROV'

QUERY PLAN                                                                                     |
-----------------------------------------------------------------------------------------------+
Bitmap Heap Scan on flights t  (cost=4.69..147.02 rows=39 width=63)                            |
  Recheck Cond: ((departure_airport = 'VOG'::bpchar) AND (arrival_airport = 'ROV'::bpchar))    |
  ->  Bitmap Index Scan on idx_dep_arr  (cost=0.00..4.68 rows=39 width=0)                      |
        Index Cond: ((departure_airport = 'VOG'::bpchar) AND (arrival_airport = 'ROV'::bpchar))|


Написать комментарии к каждому из индексов +
Описать что и как делали и с какими проблемами столкнулись -

```
2 вариант: В результате выполнения ДЗ вы научитесь пользоваться различными вариантами соединения таблиц. 
В данном задании тренируются навыки:
написания запросов с различными типами соединений 

Необходимо:
```sql
`Реализовать прямое соединение двух или более таблиц`

select count(*) --8391852
from bookings.ticket_flights tf
join bookings.flights f
    on tf.flight_id  = f.flight_id ;

`Реализовать левостороннее (или правостороннее) соединение двух или более таблиц`

--левосторонне
select count(*) --8391852
from bookings.ticket_flights tf
left join bookings.flights f
    on tf.flight_id  = f.flight_id ;

--правостороннее
select count(*) --8456131 
from bookings.ticket_flights tf
right join bookings.flights f
    on tf.flight_id  = f.flight_id ;   

`Реализовать кросс соединение двух или более таблиц`
--Запрос выполнялся очень долго, план приложил
select count(*)
from bookings.ticket_flights tf
cross join bookings.flights f

QUERY PLAN                                                                          
------------------------------------------------------------------------------------
Finalize Aggregate  (cost=23263100558.60..23263100558.61 rows=1 width=8)            
  ->  Gather  (cost=23263100558.39..23263100558.60 rows=2 width=8)                  
        Workers Planned: 2                                                          
        ->  Partial Aggregate  (cost=23263099558.39..23263099558.40 rows=1 width=8) 
              ->  Nested Loop  (cost=0.29..21384869222.10 rows=751292134515 width=0)
                    ->  Parallel Seq Scan on ticket_flights tf  (cost=0.00..104898.4
                    ->  Index Only Scan using flights_flight_no_idx on flights f    

`Реализовать полное соединение двух или более таблиц`

select count(*) --8456131
from bookings.ticket_flights tf
full join bookings.flights f
on tf.flight_id  = f.flight_id;

QUERY PLAN                                                                       
---------------------------------------------------------------------------------
Aggregate  (cost=364915.84..364915.85 rows=1 width=8)                            
  ->  Hash Full Join  (cost=8298.51..343936.57 rows=8391708 width=0)             
        Hash Cond: (tf.flight_id = f.flight_id)                                  
        ->  Seq Scan on ticket_flights tf  (cost=0.00..153850.08 rows=8391708 wid
        ->  Hash  (cost=4772.67..4772.67 rows=214867 width=4)                    
              ->  Seq Scan on flights f  (cost=0.00..4772.67 rows=214867 width=4)


`Реализовать запрос, в котором будут использованы разные типы соединений`

select tf.ticket_no, f.flight_id, t.ticket_no --count 8391852
from bookings.ticket_flights tf
join bookings.flights f on tf.flight_id  = f.flight_id
left join bookings.tickets t on t.ticket_no = tf.ticket_no;

Сделать комментарии на каждый запрос
К работе приложить структуру таблиц, для которых выполнялись соединения
--Была использвана DEMO база данных схемы Bookings, развёрнутая на локальном стенде

```