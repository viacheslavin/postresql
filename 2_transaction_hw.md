# Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд.

ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM SET deadlock_timeout = 200;
SELECT pg_reload_conf();
SHOW deadlock_timeout;

# Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

### 1 console
BEGIN;
UPDATE test2 SET amount = amount + 10 WHERE i = 1;
### 2 console
BEGIN;
UPDATE test2 SET amount = amount + 1 WHERE i = 1;

### 1 console
SELECT pg_sleep(10);
COMMIT;
### 2 console
COMMIT;

tail -n 10 /var/log/postgresql/postgresql-17-main.log

2025-06-23 17:10:03.549 UTC [3317] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "test2"
2025-06-23 17:10:03.549 UTC [3317] postgres@postgres STATEMENT:  UPDATE test2 SET amount = amount + 1 WHERE i = 1;
2025-06-23 17:10:43.204 UTC [3317] postgres@postgres LOG:  process 3317 acquired ShareLock on transaction 55027 after 39855.058 ms
2025-06-23 17:10:43.204 UTC [3317] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "test2"
2025-06-23 17:10:43.204 UTC [3317] postgres@postgres STATEMENT:  UPDATE test2 SET amount = amount + 1 WHERE i = 1;

По логу видно, что вторая транзакция попыталась изменить запись в 17:10:03.549, но начала выполняться (заблокировала запись) только в 17:10:43.204, после завершения первой транзакции.

# Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах.

### 1 console
BEGIN;
UPDATE test2 SET amount = amount + 10 WHERE i = 2;
### 2 console
BEGIN;
UPDATE test2 SET amount = amount - 1 WHERE i = 2;
### 3 console
BEGIN;
UPDATE test2 SET amount = amount - 2 WHERE i = 2;

# Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны.

postgres=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 13662;
   locktype    | relation | virtxid |  xid  |       mode       | granted 
---------------+----------+---------+-------+------------------+---------
 relation      | pg_locks |         |       | AccessShareLock  | t
 relation      | test2    |         |       | RowExclusiveLock | t
 virtualxid    |          | 22/16   |       | ExclusiveLock    | t
 transactionid |          |         | 55038 | ExclusiveLock    | t
(4 rows)

postgres=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 14398;
   locktype    | relation | virtxid |  xid  |       mode       | granted 
---------------+----------+---------+-------+------------------+---------
 relation      | test2    |         |       | RowExclusiveLock | t
 virtualxid    |          | 26/4    |       | ExclusiveLock    | t
 tuple         | test2    |         |       | ExclusiveLock    | t
 transactionid |          |         | 55038 | ShareLock        | f
 transactionid |          |         | 55039 | ExclusiveLock    | t
(5 rows)

postgres=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 14461;
   locktype    | relation | virtxid |  xid  |       mode       | granted 
---------------+----------+---------+-------+------------------+---------
 relation      | test2    |         |       | RowExclusiveLock | t
 virtualxid    |          | 27/5    |       | ExclusiveLock    | t
 transactionid |          |         | 55040 | ExclusiveLock    | t
 tuple         | test2    |         |       | ExclusiveLock    | f
(4 rows)

Первая транзакция (55038) выполнила UPDATE и удерживает запись.
Вторая транзакция (55039) ждёт завершения первой транзакции (ShareLock с granted = f).
Третья транзакция (55040) не может получить блокировку строки, так как она уже заблокирована второй транзакцией, которая сам ждет завершения первой транзации (tuple с granted = f).

# Воспроизведите взаимоблокировку трех транзакций.

### 1 console
BEGIN;
UPDATE accounts SET amount = amount + 1 WHERE acc_no = 1;
### 2 console
BEGIN;
UPDATE accounts SET amount = amount + 2 WHERE acc_no = 2;
### 3 console
BEGIN;
UPDATE accounts SET amount = amount + 3 WHERE acc_no = 3;

### 1 console
UPDATE accounts SET amount = amount + 4 WHERE acc_no = 2;
### 2 console
UPDATE accounts SET amount = amount + 5 WHERE acc_no = 3;
### 3 console
UPDATE accounts SET amount = amount + 4 WHERE acc_no = 1;

В итоге PostgreSQL обнаруживает deadlock и завершает одну из транзакций с ошибкой deadlock detected

# Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

Да, так как в журнале помимо записи об ошибке есть и предшествующие ей действия.
