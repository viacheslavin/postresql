# Создать бекапы с помощью pg_dump, pg_dumpall и pg_basebackup сравнить скорость создания и возможности.
1. pg_dump создает логическую копию, не создает табличные пространства и пользователей.
скопировать базу:
sudo -u postgres pg_dump -d db1 -C -p 5432 > /mnt/files/arh_pgdump.sql
скопировать одну таблицу:
sudo -u postgres pg_dump -d db1 -p 5432 -t student > /mnt/files/table.sql
скопировать только схему таблицы:
sudo -u postgres pg_dump -d db1 -p 5432 -t student --schema-only > /mnt/files/table_schema.sql
скопировать только данные таблицы:
sudo -u postgres pg_dump -d db1 -p 5432 -t student --data-only > /mnt/files/table_schema.sql


2. pg_dumpall создает логичекскую копию кластера, включая роли и табличные пространства.
sudo -u postgres pg_dumpall -p 5432 > /mnt/files/bkup_all.sql

3. pg_basebackup создает физическую копию, работает быстрее, так как копирует файлы напрямую с диска.
show wal_level;
 wal_level 
-----------
 replica
(1 row)

sudo -u postgres pg_createcluster -d /var/lib/postgresql/17/main3 17 main3

/mnt/files$ sudo ls -la /var/lib/postgresql/17
total 20
drwxr-xr-x  5 postgres postgres 4096 Jul 16 19:29 .
drwxr-xr-x  5 postgres postgres 4096 Jul 13 06:42 ..
drwx------ 19 postgres postgres 4096 Jul 16 17:11 main
drwx------ 19 postgres postgres 4096 Jul 16 17:11 main2
drwx------ 19 postgres postgres 4096 Jul 16 19:29 main3

rm -rf /var/lib/postgresql/17/main3
sudo -u postgres pg_basebackup -p 5432 -D /var/lib/postgresql/17/main3

sudo pg_ctlcluster 17 main3 start
sudo -u postgres psql -p 5434

\l
   Name    |  Owner   | Encoding | Locale Provider | Collate |  Ctype  | Locale | ICU Rules |   Access privileges   
-----------+----------+----------+-----------------+---------+---------+--------+-----------+-----------------------
 db1       | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |        |           | 
 demo      | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |        |           | 
 postgres  | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |        |           | 
 template0 | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |        |           | =c/postgres          +
           |          |          |                 |         |         |        |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |        |           | =c/postgres          +
           |          |          |                 |         |         |        |           | postgres=CTc/postg

sudo pg_ctlcluster 17 main3 stop
sudo pg_dropcluster 17 main3

# Настроить копирование WAL файлов.
SELECT name, setting FROM pg_settings WHERE name IN ('archive_mode','archive_command','archive_timeout');
      name       |  setting   
-----------------+------------
 archive_command | (disabled)
 archive_mode    | off
 archive_timeout | 0
(3 rows)

sudo mkdir /archive_wal
sudo chown -R postgres:postgres /archive_wal

ALTER SYSTEM SET archive_mode = 'on';
ALTER SYSTEM SET archive_command = 'test ! -f /archive_wal/%f && cp %p /archive_wal/%f';

sudo systemctl restart postgresql

sudo mkdir /full_backup
sudo chown -R postgres:postgres /full_backup

sudo -u postgres pg_basebackup -p 5432 -v -D /full_backup

create table test (c1 text);

insert into test (c1) values
  ('str1'),
  ('str1'),
  ('str1'),
  ('str1'),
  ('str1');

select * from test;
  c1  
------
 str1
 str1
 str1
 str1
 str1
(5 rows)

select now();
              now              
-------------------------------
 2025-07-16 20:47:14.245984+00
(1 row)

update student set fio = 'Иванова';
UPDATE 20

select * from student;
 id |    fio     
----+------------
  1 | Иванова   
  2 | Иванова   
  3 | Иванова   
  4 | Иванова   
  5 | Иванова   
  6 | Иванова   
  7 | Иванова   
  8 | Иванова   
  9 | Иванова   
 10 | Иванова   
  1 | Иванова   
  2 | Иванова   
  3 | Иванова   
  4 | Иванова   
  5 | Иванова   
  6 | Иванова   
  7 | Иванова   
  8 | Иванова   
  9 | Иванова   
 10 | Иванова   
(20 rows)

# Восстановить базу на другой машине PostgreSQL на заданное время, используя ранее созданные бекапы и WAL файлы.
sudo systemctl stop postgresql

sudo mkdir /old_data
sudo mv /var/lib/postgresql/17/main /old_data
sudo mkdir /var/lib/postgresql/17/main
sudo cp -a /full_backup/. /var/lib/postgresql/17/main

в postgresql.conf:
restore_command = 'cp /archive_wal/%f "%p"'
recovery_target_time = '2025-07-16 20:47:14.245984+00'

sudo touch /var/lib/postgresql/17/main/recovery.signal
sudo chown -R postgres:postgres /var/lib/postgresql/17/main/
sudo chmod -R 750 /var/lib/postgresql/17/main/

sudo systemctl start postgresql

select * from test;
  c1  
------
 str1
 str1
 str1
 str1
 str1
(5 rows)

select * from student;
 id |    fio     
----+------------
  1 | 181165b122
  2 | fa3b36521f
  3 | 5c46d58c02
  4 | cd3e5c2781
  5 | 8f259342de
  6 | 45f4fd675f
  7 | 766d407275
  8 | 2a81d31a1b
  9 | 15d13c2919
 10 | 67e08c90ab
  1 | 181165b122
  2 | fa3b36521f
  3 | 5c46d58c02
  4 | cd3e5c2781
  5 | 8f259342de
  6 | 45f4fd675f
  7 | 766d407275
  8 | 2a81d31a1b
  9 | 15d13c2919
 10 | 67e08c90ab
(20 rows)

select pg_wal_replay_resume();
 pg_wal_replay_resume 
----------------------
 
(1 row)