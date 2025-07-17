# Настроить физическую репликацию между двумя нодами: мастер - слейв.

db1=# show wal_level;
 wal_level 
-----------
 replica
(1 row)

sudo pg_createcluster -d /var/lib/postgresql/17/main3 17 main3
sudo rm -rf /var/lib/postgresql/17/main3

sudo -u postgres pg_basebackup -p 5432 -R -D /var/lib/postgresql/17/main3

sudo ls -la /var/lib/postgresql/17
total 20
drwxr-xr-x  5 postgres postgres 4096 Jul 17 14:25 .
drwxr-xr-x  5 postgres postgres 4096 Jul 16 22:52 ..
drwxr-x--- 19 postgres postgres 4096 Jul 17 14:10 main
drwx------ 19 postgres postgres 4096 Jul 17 14:10 main2
drwxr-x--- 19 postgres postgres 4096 Jul 17 14:26 main3

sudo pg_ctlcluster 17 main3 start

sudo -u postgres psql	-d db1 -p 5434

# на реплике (main3)
insert into student values(11, 'test');
ERROR:  cannot execute INSERT in a read-only transaction

# на мастере (main)
insert into student values(11, 'test');
INSERT 0 1

select * from student;
Дает на мастере и на реплике одинаковый результат
 id |    fio     
----+------------
  1 | e4cd6e2427
  2 | 381843a85d
  3 | 1c0656c37b
  4 | 00fbfe9b7d
  5 | 5f4b2b6d09
  6 | fc8f8ecb95
  7 | 50a3e3edbc
  8 | 05d4f8985b
  9 | c19d66602f
 10 | 19dc089ed0
 11 | test      
(11 rows)

# на мастере
SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 2001
usesysid         | 10
usename          | postgres
application_name | 17/main3
client_addr      | 
client_hostname  | 
client_port      | -1
backend_start    | 2025-07-17 14:35:33.097595+00
backend_xmin     | 
state            | streaming
sent_lsn         | 0/FD000508
write_lsn        | 0/FD000508
flush_lsn        | 0/FD000508
replay_lsn       | 0/FD000508
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2025-07-17 15:29:19.434268+00

SELECT * FROM pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 0/FD000508
(1 row)

# На реплике
select * from pg_stat_wal_receiver \gx
pid                   | 2000
status                | streaming
receive_start_lsn     | 0/FD000000
receive_start_tli     | 2
written_lsn           | 0/FD000508
written_lsn           | 0/FD000508
flushed_lsn           | 0/FD000508
received_tli          | 2
last_msg_send_time    | 2025-07-17 15:35:19.560695+00
last_msg_receipt_time | 2025-07-17 15:35:19.560726+00
latest_end_lsn        | 0/FD000508
latest_end_time       | 2025-07-17 15:25:49.360212+00
slot_name             | 
sender_host           | /var/run/postgresql
sender_port           | 5432
conninfo              | user=postgres passfile=/var/lib/postgresql/.pgpass channel_binding=prefer dbname=replication port=5432 fallbac
k_application_name=17/main3 sslmode=prefer sslnegotiation=postgres sslcompression=0 sslcertmode=allow sslsni=1 ssl_min_protocol_versio
n=TLSv1.2 gssencmode=prefer krbsrvname=postgres gssdelegation=0 target_session_attrs=any load_balance_hosts=disable

select pg_last_wal_receive_lsn();
 pg_last_wal_receive_lsn 
-------------------------
 0/FD000508
(1 row)

select pg_last_wal_replay_lsn();
 pg_last_wal_replay_lsn 
------------------------
 0/FD000508
(1 row)


# Настроить каскадную репликацию со слейва на третью ноду.

sudo pg_createcluster -d /var/lib/postgresql/17/main3c 17 main3c
sudo rm -rf /var/lib/postgresql/17/main3c

sudo -u postgres pg_basebackup -p 5434 -R -D /var/lib/postgresql/17/main3c

sudo pg_ctlcluster 17 main3c start

sudo -u postgres psql	-d db1 -p 5436

# на мастере (main)
insert into student values(12, 'test2');
select * from student;
 id |    fio     
----+------------
  1 | e4cd6e2427
  2 | 381843a85d
  3 | 1c0656c37b
  4 | 00fbfe9b7d
  5 | 5f4b2b6d09
  6 | fc8f8ecb95
  7 | 50a3e3edbc
  8 | 05d4f8985b
  9 | c19d66602f
 10 | 19dc089ed0
 11 | test      
 12 | test2     
(12 rows)

# на третьей ноде (main3c)
select * from student;
 id |    fio     
----+------------
  1 | e4cd6e2427
  2 | 381843a85d
  3 | 1c0656c37b
  4 | 00fbfe9b7d
  5 | 5f4b2b6d09
  6 | fc8f8ecb95
  7 | 50a3e3edbc
  8 | 05d4f8985b
  9 | c19d66602f
 10 | 19dc089ed0
 11 | test      
 12 | test2     
(12 rows)


# Настроить логическую репликацию таблицы с мастер ноды на четвертую ноду.

sudo pg_createcluster -d /var/lib/postgresql/17/main4 17 main4

sudo pg_ctlcluster 17 main4 start

# на main
ALTER SYSTEM SET wal_level = logical;

sudo pg_ctlcluster 17 main restart

CREATE PUBLICATION test_pub FOR TABLE student;
WARNING:  "wal_level" is insufficient to publish logical changes
HINT:  Set "wal_level" to "logical" before creating subscriptions.
CREATE PUBLICATION

\dRp+
                            Publication test_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root 
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.student"

\password 
123

# на main4
CREATE SUBSCRIPTION db1_sub 
CONNECTION 'host=localhost port=5432 user=postgres password=123 dbname=db1' 
PUBLICATION test_pub WITH (copy_data = true);

\dRs
           List of subscriptions
  Name   |  Owner   | Enabled | Publication 
---------+----------+---------+-------------
 db1_sub | postgres | t       | {test_pub}
(1 row)

SELECT * FROM pg_stat_subscription \gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16394
subname               | db1_sub
worker_type           | apply
pid                   | 13750
leader_pid            | 
relid                 | 
received_lsn          | 0/FE0017A8
last_msg_send_time    | 2025-07-17 20:22:31.046905+00
last_msg_receipt_time | 2025-07-17 20:22:31.046958+00
latest_end_lsn        | 0/FE0017A8
latest_end_time       | 2025-07-17 20:22:31.046905+00

select * from student;
 id |    fio     
----+------------
  1 | e4cd6e2427
  2 | 381843a85d
  3 | 1c0656c37b
  4 | 00fbfe9b7d
  5 | 5f4b2b6d09
  6 | fc8f8ecb95
  7 | 50a3e3edbc
  8 | 05d4f8985b
  9 | c19d66602f
 10 | 19dc089ed0
 11 | test      
 12 | test2     
(12 rows)

# на main
insert into student values(13, 'test3');

# на main4
select * from student;
 id |    fio     
----+------------
  1 | e4cd6e2427
  2 | 381843a85d
  3 | 1c0656c37b
  4 | 00fbfe9b7d
  5 | 5f4b2b6d09
  6 | fc8f8ecb95
  7 | 50a3e3edbc
  8 | 05d4f8985b
  9 | c19d66602f
 10 | 19dc089ed0
 11 | test      
 12 | test2     
 13 | test3     
(13 rows)
