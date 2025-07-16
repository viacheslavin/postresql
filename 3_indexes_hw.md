# Написать запросы поиска данных без индексов, посмотреть их план запросов.
1.
explain (costs true, verbose true, analyze true)
select *
from flights
where status = 'Delayed';

---------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on bookings.flights  (cost=0.00..1644.80 rows=35 width=63) (actual time=0.046..5.367 rows=41 loops=1)
   Output: flight_id, flight_no, scheduled_departure, scheduled_arrival, departure_airport, arrival_airport, status, aircraft_code, actual_departure, actual_arrival
   Filter: ((flights.status)::text = 'Delayed'::text)
   Rows Removed by Filter: 65623
 Planning Time: 0.067 ms
 Execution Time: 5.385 ms
(6 rows)


2.
explain (costs true, verbose true, analyze true)
select *
from flights
where departure_airport = 'DME'
and scheduled_departure between '2016-07-01' and '2016-08-31';

-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on bookings.flights  (cost=0.00..1973.12 rows=2489 width=63) (actual time=0.011..5.376 rows=2475 loops=1)
   Output: flight_id, flight_no, scheduled_departure, scheduled_arrival, departure_airport, arrival_airport, status, aircraft_code, actual_departure, actual_arrival
   Filter: ((flights.scheduled_departure >= '2016-07-01 00:00:00+00'::timestamp with time zone) AND (flights.scheduled_departure <= '2016-08-31 00:00:00+00'::timestamp with time zone) AND (flights.departure_airport = 'DME'::bpchar))
   Rows Removed by Filter: 63189
 Planning Time: 0.083 ms
 Execution Time: 5.459 ms
(6 rows)

# Добавить на таблицы индексы с целью оптимизации запросов поиска данных.
1.
Простой индекс
create index idx_flights_status on flights(status);

explain (costs true, verbose true, analyze true)
select *
from flights
where status = 'Delayed';

---------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using idx_flights_status on bookings.flights  (cost=0.29..93.26 rows=35 width=63) (actual time=0.028..0.102 rows=41 loops=1)
   Output: flight_id, flight_no, scheduled_departure, scheduled_arrival, departure_airport, arrival_airport, status, aircraft_code, actual_departure, actual_arrival
   Index Cond: ((flights.status)::text = 'Delayed'::text)
 Planning Time: 0.210 ms
 Execution Time: 0.122 ms
(5 rows)


2.
Составной индекс
create index idx_flights_airport_sched on flights(departure_airport, scheduled_departure);

explain (costs true, verbose true, analyze true)
select *
from flights
where departure_airport = 'DME'
and scheduled_departure between '2016-07-01' and '2016-08-31';

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on bookings.flights  (cost=72.15..939.71 rows=2489 width=63) (actual time=0.194..0.492 rows=2475 loops=1)
   Output: flight_id, flight_no, scheduled_departure, scheduled_arrival, departure_airport, arrival_airport, status, aircraft_code, actual_departure, actual_arrival
   Recheck Cond: ((flights.departure_airport = 'DME'::bpchar) AND (flights.scheduled_departure >= '2016-07-01 00:00:00+00'::timestamp with time zone) AND (flights.scheduled_departure <= '2016-08-31 00:00:00+00'::timestamp with time zone))
   Heap Blocks: exact=77
   ->  Bitmap Index Scan on idx_flights_airport_sched  (cost=0.00..71.53 rows=2489 width=0) (actual time=0.179..0.179 rows=2475 loops=1)
         Index Cond: ((flights.departure_airport = 'DME'::bpchar) AND (flights.scheduled_departure >= '2016-07-01 00:00:00+00'::timestamp with time zone) AND (flights.scheduled_departure <= '2016-08-31 00:00:00+00'::timestamp with time zone))
 Planning Time: 0.246 ms
 Execution Time: 0.574 ms
(8 rows)

# Сравнить новые планы запросов с предыдущими.
1.
Без индекса идет последовательное сканирование, запрос занимает около 5 мс
После создания используется Index Scan и запрос занимает меньше 1 мс

2.
Без индекса идет последовательное сканирование, запрос занимает около 5 мс
После создания используется Bitmap Index Scan с использованием составного индекса и запрос занимает меньше 1 мс

# Сравнить применение различных типов индексов.

# B-tree - в PostgreSQL используется по умолчанию
Универсальный индекс, может использоваться для диапазонов, подждерживает операции стравнения =, <, >, BETWEEN, LIKE

Остальные типы индекса создаются с использованием команды USING

# Hash
Точный поиск по значению, поддерживает только =, но работает быстрее
explain (costs true, verbose true, analyze true)
select *
from bookings.flights
where flight_no = 'PG0351';

---------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on bookings.flights  (cost=5.50..372.34 rows=140 width=63) (actual time=0.038..0.053 rows=121 loops=1)
   Output: flight_id, flight_no, scheduled_departure, scheduled_arrival, departure_airport, arrival_airport, status, aircraft_code, actual_departure, actual_arrival
   Recheck Cond: (flights.flight_no = 'PG0351'::bpchar)
   Heap Blocks: exact=4
   ->  Bitmap Index Scan on flights_flight_no_scheduled_departure_key  (cost=0.00..5.47 rows=140 width=0) (actual time=0.029..0.029 rows=121 loops=1)
         Index Cond: (flights.flight_no = 'PG0351'::bpchar)
 Planning Time: 0.080 ms
 Execution Time: 0.083 ms
(8 rows)

create index idx_flights_flight_no_hash on bookings.flights using hash (flight_no);

explain (costs true, verbose true, analyze true)
select *
from bookings.flights
where flight_no = 'PG0351';

---------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on bookings.flights  (cost=5.08..371.93 rows=140 width=63) (actual time=0.023..0.045 rows=121 loops=1)
   Output: flight_id, flight_no, scheduled_departure, scheduled_arrival, departure_airport, arrival_airport, status, aircraft_code, actual_departure, actual_arrival
   Recheck Cond: (flights.flight_no = 'PG0351'::bpchar)
   Heap Blocks: exact=4
   ->  Bitmap Index Scan on idx_flights_flight_no_hash  (cost=0.00..5.05 rows=140 width=0) (actual time=0.014..0.015 rows=121 loops=1)
         Index Cond: (flights.flight_no = 'PG0351'::bpchar)
 Planning Time: 0.158 ms
 Execution Time: 0.064 ms
(8 rows)

# GIN
Используется для полнотекстовго поиска, массивов, jsonb

alter table tickets add column fio_lexeme tsvector;
update tickets
set fio_lexeme = to_tsvector(passenger_name)
where ticket_no < '0005432005201';

explain (costs true, analyze true)
select ticket_no, passenger_name from tickets where fio_lexeme @@ to_tsquery('olga | Elena' ); 
--------------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..106434.64 rows=8270 width=30) (actual time=1523.417..1530.807 rows=59 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on tickets  (cost=0.00..104607.64 rows=3446 width=30) (actual time=1488.207..1489.969 rows=20 loops=3)
         Filter: (fio_lexeme @@ to_tsquery('olga | Elena'::text))
         Rows Removed by Filter: 276337
 Planning Time: 0.087 ms
 JIT:
   Functions: 12
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 1.292 ms (Deform 0.613 ms), Inlining 0.000 ms, Optimization 0.892 ms, Emission 27.454 ms, Total 29.638 ms
 Execution Time: 1531.147 ms
(12 rows)


CREATE INDEX search_index_title ON tickets USING GIN (fio_lexeme);

explain (costs true, analyze true)
select ticket_no, passenger_name from tickets where fio_lexeme @@ to_tsquery('olga | Elena' ); 
------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on tickets  (cost=13.65..340.38 rows=81 width=30) (actual time=0.032..0.065 rows=59 loops=1)
   Recheck Cond: (fio_lexeme @@ to_tsquery('olga | Elena'::text))
   Heap Blocks: exact=22
   ->  Bitmap Index Scan on search_index_title  (cost=0.00..13.63 rows=81 width=0) (actual time=0.023..0.023 rows=59 loops=1)
         Index Cond: (fio_lexeme @@ to_tsquery('olga | Elena'::text))
 Planning Time: 0.240 ms
 Execution Time: 0.082 ms
(7 rows)
