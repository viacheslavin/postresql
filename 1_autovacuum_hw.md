#  Выполнить pgbench -c8 -P 6 -T 60 -U postgres postgres.
latency average = 35.072 ms
latency stddev = 27.293 ms
initial connection time = 20.230 ms
tps = 228.018100 (without initial connection time)

# Настроить параметры vacuum/autovacuum.
alter system set autovacuum_max_workers = '10';
alter system set autovacuum_naptime = '15';
alter system set autovacuum_vacuum_threshold = '25';
alter system set autovacuum_vacuum_scale_factor = '0.05';
alter system set autovacuum_vacuum_cost_limit = '1000';

# Заново протестировать: pgbench -c8 -P 6 -T 60 -U postgres postgres и сравнить результаты.
latency average = 33.621 ms
latency stddev = 25.676 ms
initial connection time = 20.275 ms
tps = 237.841469 (without initial connection time)

Пояснение результата: 
при прогоне после настройки autovacuum увеличился TPS, так как autovacuum стал запускаться чаще и быстрее убирать мертвые записи, что позволило ускорить доступ к таблицам.

# Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк.
INSERT INTO student(fio) SELECT 'noname' FROM generate_series(1,1000000);

# Посмотреть размер файла с таблицей.
db1=# SELECT pg_size_pretty(pg_total_relation_size('student'));

 135 MB

# 5 раз обновить все строчки и добавить к каждой строчке любой символ.
db1=# update student set fio = 'noname1';
UPDATE 1000000
db1=# update student set fio = 'noname12';
UPDATE 1000000
db1=# update student set fio = 'noname123';
UPDATE 1000000
db1=# update student set fio = 'noname1234';
UPDATE 1000000
db1=# update student set fio = 'noname12345';
UPDATE 1000000

# Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум.
db1=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum 
FROM pg_stat_user_TABLEs WHERE relname = 'student';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 student |    1333854 |          0 |      0 | 2025-06-10 19:21:07.844683+00
(1 row)

SELECT pg_size_pretty(pg_total_relation_size('student'));

 404 MB

# Отключить Автовакуум на таблице и опять 5 раз обновить все строки.
db1=# ALTER TABLE student SET (autovacuum_enabled = off);

db1=# update student set fio = 'noname123456';
UPDATE 1000000
db1=# update student set fio = 'noname1234567';
UPDATE 1000000
db1=# update student set fio = 'noname12345678';
UPDATE 1000000
db1=# update student set fio = 'noname123456789';
UPDATE 1000000
db1=# update student set fio = 'noname1234567890';
UPDATE 1000000

db1=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum 
FROM pg_stat_user_TABLEs WHERE relname = 'student';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 student |    1333854 |    4999514 |    374 | 2025-06-10 19:21:07.844683+00
(1 row)

db1=# SELECT pg_size_pretty(pg_total_relation_size('student'));
 808 MB

# Объяснить полученные результаты.
При изменении записей со включенным автовакуум объем вырос со 135 Мб до 404, так как автовакуум сохраняет очищенное пространство закрепленным за таблицей, при этом оно может быть повторно использоваться при добавлении новых записей, поэтом рост не в 6 раз, а меньше.
При обновлении записей без автовакуум, размер вырос с 404 Мб до 808, так как было переиспользовано часть очищенного в результатае предыдущих (до отключения) срабатываний автовакуум.
n_dead_tup после отключения автовакуума стало накапливаться, так очистка перестала запускаться автоматически.