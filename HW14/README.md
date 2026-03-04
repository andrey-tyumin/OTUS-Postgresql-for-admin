### Домашнее задание по теме "Репликация".

Цель:
реализовать свой миникластер на трех виртуальных машинах;  
Описание/Пошаговая инструкция выполнения домашнего задания:  
1. Настройте ВМ1:  
Создайте таблицу test, которая будет для операций записи  
Создайте таблицу test2, которая будет для чтения  
Настройте публикацию таблицы test  
2. Настройте ВМ2:  
Создайте таблицу test2, которая будет для операций записи  
Создайте таблицу test, которая будет для чтения  
Настройте публикацию таблицы test2  
Сделайте подписку таблицы test на публикацию таблицы test с ВМ1  
3. на ВМ1:  
Сделайте подписку таблицы test2 на публикацию таблицы test2 с ВМ2   
4. Настройте ВМ3:  
Создайте таблицы: test и test2  
Подпишите test на публикацию таблицы test с ВМ1  
Подпишите test2 на публикацию таблицы test2 с ВМ2  
Используйте этот узел для чтения объединённых данных и резервного копирования  
Проверьте работу системы:  
Выполните вставку в test на ВМ1 — убедитесь, что данные появились в test на ВМ2 и ВМ3  
Выполните вставку в test2 на ВМ2 — убедитесь, что данные появились в test2 на ВМ1 и ВМ3  
Задание повышенной сложности(*):  
Настройте физическую репликацию с ВМ4, используя ВМ3 в качестве источника.  

-----------------------------------------

Установил 4 LXC контейнера с AlmaLinux 9.4 в конфигурации: 1cpu, 4gb ram, 64Gb HDD.  
Установил Postgresql-17.  

Создал БД и таблицы на ВМ1 и ВМ2: 
```sql
CREATE DATABASE repl;
\c repl
CREATE TABLE test(id int, text varchar);
CREATE TABLE test2(id int, text varchar);
```
На ВМ1:   
настроил публикацию для таблицы test:
```sql
CREATE PUBLICATION test_pub FOR TABLE test;
```
Проверил что создалось:
```sql
SELECT * from pg_publication;
SELECT * from pg_publication_tables;
```
Создал пользователя для репликации:
```sql
CREATE ROLE repl_user WITH REPLICATION LOGIN PASSWORD 'str0ngp@ssw0rd';
```
Дал пользователю необходимые права:
```sql
GRANT USAGE ON SCHEMA public TO repl_user;
GRANT SELECT ON public.test TO repl_user;
```
В pg_hba.conf для пользователя репликации добавил запись:
```
host    replication     repl_user       192.168.0.0/24          scram-sha-256
```
В /var/lib/pgsql/17/data/postgresql.conf изменил уровень журналирования:
```
 wal_level = logical
```
Перестартовал сервер:
```bash
systemctl restart postgresql-17
```

На ВМ2:   
настроил публикацию таблицы test2:
```sql
CREATE PUBLICATION test2_pub FOR TABLE test2;
```
Проверил что создалось:
```sql
SELECT * from pg_publication;
SELECT * from pg_publication_tables;
```
Создал пользователя для репликации:
```sql
CREATE ROLE repl_user WITH REPLICATION LOGIN PASSWORD 'str0ngp@ssw0rd';
```
Дал права:
```sql
GRANT USAGE ON SCHEMA public TO repl_user;
GRANT SELECT ON public.test2 TO repl_user;
```
В pg_hba.conf для пользователя репликации добавил запись:
```
host    replication     repl_user       192.168.0.0/24          scram-sha-256
```
В /var/lib/pgsql/17/data/postgresql.conf изменил уровень журналирования:
```
 wal_level = logical
```
Перестартовал сервер:
```bash
systemctl restart postgresql-17
```
На ВМ1 сделал подписку на test2 с ВМ2:
```sql
CREATE SUBSCRIPTION sub_test2 
CONNECTION 'host=192.168.0.41 port=5432 user=repl_user password=str0ngp@ssw0rd dbname=repl' 
PUBLICATION test2_pub;
```
На ВМ2 сделал подписку на test с ВМ1:
```sql
CREATE SUBSCRIPTION sub_test 
CONNECTION 'host=192.168.0.40 port=5432 user=repl_user password=str0ngp@ssw0rd dbname=repl' 
PUBLICATION test_pub;
```
Проверил состояние слота репликации:
```sql
select * from pg_replication_slots;
```
и подписки:
```sql
select * from pg_subscription;
```
lsn сверил с издателем:
```sql
SELECT subname, received_lsn FROM pg_stat_subscription;
```
На ВМ3:   
создал бд repl и таблицы test и tes2:
```sql
CREATE DATABASE repl;
\c repl
CREATE TABLE test(id int, text varchar);
CREATE TABLE test2(id int, text varchar);
```
Создал подписки:
```sql
CREATE SUBSCRIPTION sub3_test 
CONNECTION 'host=192.168.0.40 port=5432 user=repl_user password=str0ngp@ssw0rd dbname=repl' 
PUBLICATION test_pub;

CREATE SUBSCRIPTION sub3_test2 
CONNECTION 'host=192.168.0.41 port=5432 user=repl_user password=str0ngp@ssw0rd dbname=repl' 
PUBLICATION test2_pub;
```
Внес данные в test на ВМ1: 
```sql
INSERT INTO test VALUES(1,'from vm1');
```
Внес данные в test2 на ВМ2: 
```sql
INSERT INTO test VALUES(2,'from vm2');
```
Проверил на ВМ3:
```sql
SELECT * FROM test;
SELECT * FROM test2;
```
результат:
```sql
postgres=# \c repl
You are now connected to database "repl" as user "postgres".
repl=# SELECT * FROM test;
 id |   text   
----+----------
  1 | from vm1
(1 row)

repl=# SELECT * FROM test2;
 id |   text   
----+----------
  2 | from vm2
(1 row)

repl=# 
```
Проверил состояние подписки на ВМ1:
```sql
[postgres@HW14-1 ~]$ psql
psql (13.23, server 17.9)
WARNING: psql major version 13, server major version 17.
         Some psql features might not work.
Type "help" for help.

postgres=# \x
Expanded display is on.
postgres=# SELECT * FROM pg_stat_subscription;
-[ RECORD 1 ]---------+------------------------------
subid                 | 16411
subname               | sub_test2
worker_type           | apply
pid                   | 46814
leader_pid            | 
relid                 | 
received_lsn          | 0/2A693A0
last_msg_send_time    | 2026-03-04 19:49:15.749657+00
last_msg_receipt_time | 2026-03-04 19:49:15.749711+00
latest_end_lsn        | 0/2A693A0
latest_end_time       | 2026-03-04 19:49:15.749657+00

postgres=# 
```
Проверил состояние подписки на ВМ2:
```sql
[postgres@HW14-2 ~]$ psql
psql (17.9)
Type "help" for help.

postgres=# \x
Expanded display is on.
postgres=# SELECT * FROM pg_stat_subscription;
-[ RECORD 1 ]---------+------------------------------
subid                 | 16422
subname               | sub_test
worker_type           | apply
pid                   | 44494
leader_pid            | 
relid                 | 
received_lsn          | 0/1981B18
last_msg_send_time    | 2026-03-04 19:48:11.007254+00
last_msg_receipt_time | 2026-03-04 19:48:11.007308+00
latest_end_lsn        | 0/1981B18
latest_end_time       | 2026-03-04 19:48:11.007254+00

postgres=# 
```
Репликация настроена.

-----------------

Задание со \*.
На ВМ3:
проверил уровень журналирования:
```sql
[postgres@HW14-3 ~]$ psql
psql (17.9)
Type "help" for help.

postgres=# show wal_level;
 wal_level 
-----------
 replica
(1 row)
```
replica - установлено по умолчанию.
Проверил на каких адресах "висит" Postgresql:
```sql
postgres=# show listen_addresses;
 listen_addresses 
------------------
 *
(1 row)
```
'*' устанавливал раннее.

```sql
postgres=# show wal_log_hints;
 wal_log_hints 
---------------
 on
(1 row)
```
- у меня был в off, установил в "on" в /var/lib/pgsql/data/postgresql.conf, 
Создал пользователя для репликации:
```sql
CREATE ROLE repl_user WITH REPLICATION LOGIN PASSWORD 'str0ngp@ssw0rd';
```
В /var/lib/pgsql/data/pg_hba.conf добавил
```
host    replication     repl_user       0.0.0.0/0               scram-sha-256
```
перезапустил сервер:
```bash
systemctl restart postgresql-17
systemctl status postgresql-17
```
Создал слот репликации:
```sql
SELECT * FROM pg_create_physical_replication_slot('repl_slot');
```
Проверил:
```sql
SELECT * FROM pg_create_physical_replication_slot('repl_slot');
```
На ВМ4:
остановил postgresql:
```bash
su root -c 'systemctl stop postgresql-17'
```
удалил все в /var/lib/pgsql/17/data/:
```bash
rm -rf /var/lib/pgsql/17/data/
```
Запустил создание реплики на ВМ4:
```bash
pg_basebackup -D /var/lib/pgsql/17/data -R -X stream -C -S repl_slot -v -h 192.168.0.42 -U repl_user -W
```
получил ошибку, что слот репликации уже присутствует на сервере:
```
pg_basebackup: error: could not send replication command "CREATE_REPLICATION_SLOT "repl_slot" PHYSICAL ( RESERVE_WAL)": ERROR:  replication slot "repl_slot" already exists
```
почитал, выяснил что pg_basebackup по умолчанию пытается создать временный слот репликации  
для обеспечения целостности бэкапа, т.е. можно не создавать предварительно слот репликации на мастере.  
попробовал с другим именем слота репликации:
```bsh
pg_basebackup -D /var/lib/pgsql/17/data -R -X stream -C -S repl_slot_new -v -h 192.168.0.42 -U repl_user -W
```
- так получилось.
Проверил слот на ВМ3:
```sql
[postgres@HW14-3 ~]$ psql
psql (17.9)
Type "help" for help.

postgres=# SELECT * FROM pg_replication_slots \gx
-[ RECORD 1 ]-------+----------------------------
slot_name           | repl_slot
plugin              | 
slot_type           | physical
datoid              | 
database            | 
temporary           | f
active              | f
active_pid          | 
xmin                | 
catalog_xmin        | 
restart_lsn         | 
confirmed_flush_lsn | 
wal_status          | 
safe_wal_size       | 
two_phase           | f
inactive_since      | 2026-03-04 18:59:28.1497+00
conflicting         | 
invalidation_reason | 
failover            | f
synced              | f
-[ RECORD 2 ]-------+----------------------------
slot_name           | repl_slot_new
plugin              | 
slot_type           | physical
datoid              | 
database            | 
temporary           | f
active              | t
active_pid          | 65606
xmin                | 
catalog_xmin        | 
restart_lsn         | 0/4000168
confirmed_flush_lsn | 
wal_status          | reserved
safe_wal_size       | 
two_phase           | f
inactive_since      | 
conflicting         | 
invalidation_reason | 
failover            | f
synced              | f

postgres=#
```
- присутствует.  
запустил postgresql.
```bash
systemctl start postgresql-17
systemctl status postgresql-17
```
Проверил данные в Postgresql - все на месте:
```sql
[postgres@HW14-4 ~]$ psql
psql (17.9)
Type "help" for help.

postgres=# \l
                                                    List of databases
   Name    |  Owner   | Encoding | Locale Provider |  Collate   |   Ctype    | Locale | ICU Rules |   Access privileges   
-----------+----------+----------+-----------------+------------+------------+--------+-----------+-----------------------
 postgres  | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |        |           | 
 repl      | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |        |           | 
 template0 | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |        |           | =c/postgres          +
           |          |          |                 |            |            |        |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |        |           | =c/postgres          +
           |          |          |                 |            |            |        |           | postgres=CTc/postgres
(4 rows)

postgres=# \c repl
You are now connected to database "repl" as user "postgres".
repl=# \dt
         List of relations
 Schema | Name  | Type  |  Owner   
--------+-------+-------+----------
 public | test  | table | postgres
 public | test2 | table | postgres
(2 rows)

repl=# select * from test;
 id |   text   
----+----------
  1 | from vm1
(1 row)

repl=# select * from test2;
 id |   text   
----+----------
  2 | from vm2
(1 row)

repl=# 
```

Проверил статус репликации на реплике:
```sql
repl=# select * from pg_stat_wal_receiver \gx
-[ RECORD 1 ]---------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
pid                   | 63752
status                | streaming
receive_start_lsn     | 0/4000000
receive_start_tli     | 1
written_lsn           | 0/4000168
flushed_lsn           | 0/4000168
received_tli          | 1
last_msg_send_time    | 2026-03-04 19:27:37.103563+00
last_msg_receipt_time | 2026-03-04 19:27:37.103616+00
latest_end_lsn        | 0/4000168
latest_end_time       | 2026-03-04 19:24:07.037945+00
slot_name             | repl_slot_new
sender_host           | 192.168.0.42
sender_port           | 5432
conninfo              | user=repl_user password=******** channel_binding=prefer dbname=replication host=192.168.0.42 port=5432 fallback_application_name=walreceiver sslmode=prefer sslnegotiation=postgres sslcompression=0 sslcertmode=allow sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres gssdelegation=0 target_session_attrs=any load_balance_hosts=disable

repl=# select pg_last_wal_receive_lsn();
 pg_last_wal_receive_lsn 
-------------------------
 0/4000168
(1 row)

repl=# select pg_last_wal_replay_lsn();
 pg_last_wal_replay_lsn 
------------------------
 0/4000168
(1 row)

repl=# 
```

На мастере:
```sql
postgres=# SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 65606
usesysid         | 16413
usename          | repl_user
application_name | walreceiver
client_addr      | 192.168.0.43
client_hostname  | 
client_port      | 33510
backend_start    | 2026-03-04 19:24:03.309632+00
backend_xmin     | 
state            | streaming
sent_lsn         | 0/4000168
write_lsn        | 0/4000168
flush_lsn        | 0/4000168
replay_lsn       | 0/4000168
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2026-03-04 19:29:47.156597+00

postgres=# SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 65606
usesysid         | 16413
usename          | repl_user
application_name | walreceiver
client_addr      | 192.168.0.43
client_hostname  | 
client_port      | 33510
backend_start    | 2026-03-04 19:24:03.309632+00
backend_xmin     | 
state            | streaming
sent_lsn         | 0/4000168
write_lsn        | 0/4000168
flush_lsn        | 0/4000168
replay_lsn       | 0/4000168
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2026-03-04 19:29:57.156796+00

postgres=# SELECT * FROM pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 0/4000168
(1 row)

postgres=# select * from pg_replication_slots;
   slot_name   | plugin | slot_type | datoid | database | temporary | active | active_pid | xmin | catalog_xmin | restart_lsn | confirmed_flush_lsn | wal_status | safe_
wal_size | two_phase |       inactive_since        | conflicting | invalidation_reason | failover | synced 
---------------+--------+-----------+--------+----------+-----------+--------+------------+------+--------------+-------------+---------------------+------------+------
---------+-----------+-----------------------------+-------------+---------------------+----------+--------
 repl_slot     |        | physical  |        |          | f         | f      |            |      |              |             |                     |            |      
         | f         | 2026-03-04 18:59:28.1497+00 |             |                     | f        | f
 repl_slot_new |        | physical  |        |          | f         | t      |      65606 |      |              | 0/4000168   |                     | reserved   |      
         | f         |                             |             |                     | f        | f
(2 rows)

postgres=# 
```
Физическая репликация настроена.
