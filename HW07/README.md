Домашнее задание по теме "Журналы".

Работа с журналами

Цель:

уметь работать с журналами и контрольными точками;  
уметь настраивать параметры журналов;
 

Описание/Пошаговая инструкция выполнения домашнего задания:

1.  Настройте выполнение контрольной точки раз в 30 секунд.
2.  10 минут c помощью утилиты pgbench подавайте нагрузку.
3.  Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.
4.  Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?
5.  Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.
6.  Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?

---

ВМ создал в vk cloud.  Параметры: 2 CPU, 8Gb RAM 30Gb HDD.
Добавил репозиторий и установил postgresql 18:
```bash
sudo apt install -y postgresql-common  
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
sudo apt install postgresql-18
```
Для настройки выполнения контрольной точки, нужно определить 2 параметра:
-   <u>checkpoint_timeout</u>  — максимальное время между контрольными точками. Если значение задано без единиц измерения, оно считается заданным в секундах. По умолчанию — 5 минут.
-   **checkpoint_completion_target**  — задаёт целевое время завершения контрольной точки как долю общего времени между контрольными точками. По умолчанию — 0,9. Это значение распределяет нагрузку контрольной точки по всему интервалу времени её завершения и обеспечивает плавную нагрузку на диски. Если уменьшить значение этого параметра, то контрольные точки будут происходить более быстро, но и нагрузка на диски сильно возрастёт.
Посмотрел текущие установки на свежеустановленном сервере:
```sql
postgres=# show checkpoint_timeout;
 checkpoint_timeout 
--------------------
 5min
(1 row)

postgres=# show checkpoint_completion_target;
 checkpoint_completion_target 
------------------------------
 0.9
(1 row)
```
Посмотрел нужно ли перезапустить сервис после изменения этих параметров:
```sql
postgres=# select context from pg_settings where name='checkpoint_timeout';
 context 
---------
 sighup
(1 row)

postgres=# select context from pg_settings where name='checkpoint_completion_target';
 context 
---------
 sighup
(1 row)
```
Перезагрузка сервиса не нужна, только reload.
Меняю параметры и делаю релоад:
```sql
postgres=# alter system set checkpoint_timeout = '30s';
ALTER SYSTEM
postgres=# show checkpoint_timeout;
 checkpoint_timeout 
--------------------
 5min
(1 row)
postgres=# SELECT pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)

postgres=# show checkpoint_timeout;
 checkpoint_timeout 
--------------------
 30s
(1 row)

```
10 минут перерыв с pgbench :
```bash
postgres@ubuntu-std2-2-8-30gb:~$ pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
vacuuming...                                                                              
creating primary keys...
done in 0.31 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.19 s, vacuum 0.05 s, primary keys 0.05 s).
postgres@ubuntu-std2-2-8-30gb:~$ pgbench -P 200 -T 600 postgres
pgbench (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
starting vacuum...end.
progress: 200.0 s, 138.8 tps, lat 7.202 ms stddev 6.220, 0 failed
progress: 400.0 s, 135.9 tps, lat 7.356 ms stddev 7.275, 0 failed
progress: 600.0 s, 134.4 tps, lat 7.437 ms stddev 7.809, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 81833
number of failed transactions: 0 (0.000%)
latency average = 7.331 ms
latency stddev = 7.124 ms
initial connection time = 3.799 ms
`tps = 136.387648 (without initial connection time)`
```
Оценка кол-ва сгенерированных логов:
```bash
postgres@ubuntu-std2-2-8-30gb:~$ ls -alhS /var/lib/postgresql/18/main/pg_wal/
total 49M
-rw-------  1 postgres postgres  16M Jan  9 17:23 000000010000000000000014
-rw-------  1 postgres postgres  16M Jan  9 17:24 000000010000000000000015
-rw-------  1 postgres postgres  16M Jan  9 17:25 000000010000000000000016
```
было сгенерировано 16 wal фалов по 16Мб, файлы, созданные до выполнения checkpoint удалены.
По настройкам, за 10 мин должно было быть 20 checkpoint-ов. Проверил:
```bash
postgres@ubuntu-std2-2-8-30gb:~$ grep 'checkpoint starting' /var/log/postgresql/postgresql-18-main.log 
2026-01-09 16:36:52.921 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:14:52.520 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:15:52.060 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:16:22.215 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:16:52.281 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:17:22.202 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:17:52.161 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:18:22.215 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:18:52.222 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:19:22.262 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:19:52.148 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:20:22.175 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:20:52.181 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:21:22.241 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:21:52.208 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:22:22.260 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:22:52.200 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:23:22.254 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:23:52.191 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:24:22.257 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:24:52.163 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:25:22.218 UTC [4639] LOG:  checkpoint starting: time
2026-01-09 17:26:52.184 UTC [4639] LOG:  checkpoint starting: time
postgres@ubuntu-std2-2-8-30gb:~$ grep 'checkpoint starting' /var/log/postgresql/postgresql-18-main.log | wc -l
23
postgres@ubuntu-std2-2-8-30gb:~$ 
```
checkpoint-ов 23, скорее всего, создавались, когда менял настройки.

---

Сравнение tps в синхронном\асинхронном режимах.
режим синхронной\асинхронной записи настраивается с параметром synchronous_commit. проверил тек. настройку:
```sql
postgres=# SHOW synchronous_commit;
 synchronous_commit 
--------------------
 on
(1 row)
```

Изменил в postgresql.conf:
```bash
postgres@ubuntu-std2-2-8-30gb:~$ echo 'synchronous_commit = off' >> /etc/postgresql/18/main/postgresql.conf 
postgres@ubuntu-std2-2-8-30gb:~$ tail /etc/postgresql/18/main/postgresql.conf 
#include_if_exists = '...'              # include file only if it exists
#include = '...'                        # include file


#------------------------------------------------------------------------------
# CUSTOMIZED OPTIONS
#------------------------------------------------------------------------------

# Add settings for extensions here
synchronous_commit = off
postgres@ubuntu-std2-2-8-30gb:~$ pg_ctlcluster 18 main reload
postgres@ubuntu-std2-2-8-30gb:~$ 

```
Еще один тест на 10 минут с pgbench:
```bash
postgres@ubuntu-std2-2-8-30gb:~$ pgbench -P 200 -T 600 postgres
pgbench (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
starting vacuum...end.
progress: 200.0 s, 1796.5 tps, lat 0.556 ms stddev 0.275, 0 failed
progress: 400.0 s, 1785.1 tps, lat 0.560 ms stddev 0.336, 0 failed
progress: 600.0 s, 1794.4 tps, lat 0.557 ms stddev 0.274, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 1075213
number of failed transactions: 0 (0.000%)
latency average = 0.558 ms
latency stddev = 0.296 ms
initial connection time = 5.336 ms
`tps = 1792.035928 (without initial connection time)`

```
В синхронном режиме: 
```
tps = 136.387648
```
В асинхронном режиме:
```
tps = 1792.035928
```
разница существенна, т.к.:

- фиксация изменений не ждет подтверждения записи(т.е. медленного ввода\вывода на диск)
- В режиме асинхронного подтверждения сервер сообщает об успешном завершении сразу, как только транзакция будет завершена логически, не дожидаясь записи wal на диск.
-   Если с прошлого раза в буферах была целиком заполнена одна или несколько страниц, сбрасываются только такие, полностью заполненные.
- но если новых полных страниц нет, то записывает последнюю до конца.

---
Сделал новый кластер с включенными контрольными суммами:
```bash
postgres@ubuntu-std2-2-8-30gb:~$ pg_createcluster --port 5433 --start 18 second -- --data-checksums
Creating new PostgreSQL cluster 18/second ...
/usr/lib/postgresql/18/bin/initdb -D /var/lib/postgresql/18/second --auth-local peer --auth-host scram-sha-256 --no-instructions --data-checksums
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "C.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/18/second ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default "max_connections" ... 100
selecting default "shared_buffers" ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Warning: systemd does not know about the new cluster yet. Operations like "service postgresql start" will not handle it. To fix, run:
  sudo systemctl daemon-reload
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@18-second
Ver Cluster Port Status Owner    Data directory                Log file
18  second  5433 online postgres /var/lib/postgresql/18/second /var/log/postgresql/postgresql-18-second.log
postgres@ubuntu-std2-2-8-30gb:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory                Log file
18  main    5432 online postgres /var/lib/postgresql/18/main   /var/log/postgresql/postgresql-18-main.log
18  second  5433 online postgres /var/lib/postgresql/18/second /var/log/postgresql/postgresql-18-second.log
postgres@ubuntu-std2-2-8-30gb:~$ psql -p 5433 -c 'show data_checksums'
 data_checksums 
----------------
 on
(1 row)

postgres@ubuntu-std2-2-8-30gb:~$ 

```
Создал таблицу и внес данные:
```sql
postgres=# create table test_table(id int);
CREATE TABLE
postgres=# insert into test_table(id) values (1),(2),(3);
INSERT 0 3
postgres=# select * from test_table;
 id 
----
  1
  2
  3
(3 rows)

```
Посмотрел, где расположена таблица в файловой системе:
```sql
postgres=# SELECT pg_relation_filepath('test_table');
 pg_relation_filepath 
----------------------
 base/5/16388
(1 row)

```
Остановил кластер second:
```bash
postgres@ubuntu-std2-2-8-30gb:~$ pg_ctlcluster 18 second stop
postgres@ubuntu-std2-2-8-30gb:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory                Log file
18  main    5432 online postgres /var/lib/postgresql/18/main   /var/log/postgresql/postgresql-18-main.log
18  second  5433 down   postgres /var/lib/postgresql/18/second /var/log/postgresql/postgresql-18-second.log

```
Меняем файл таблицы:
```bash
hexedit /var/lib/postgresql/18/second/base/5/16388
```
Запустил кластер second:
```bash
postgres@ubuntu-std2-2-8-30gb:~$ pg_ctlcluster 18 second start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@18-second
postgres@ubuntu-std2-2-8-30gb:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory                Log file
18  main    5432 online postgres /var/lib/postgresql/18/main   /var/log/postgresql/postgresql-18-main.log
18  second  5433 online postgres /var/lib/postgresql/18/second /var/log/postgresql/postgresql-18-second.log

```
подключился к базе, сделал выборку:
```sql
postgres@ubuntu-std2-2-8-30gb:~$ psql -p 5433
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.

postgres=# \dt
             List of tables
 Schema |    Name    | Type  |  Owner   
--------+------------+-------+----------
 public | test_table | table | postgres
(1 row)

postgres=# select * from test_table;
ERROR:  invalid page in block 0 of relation "base/5/16388"
postgres=# 

(4 rows)

```
Для игнора ошибки, есть 2 варианта:
- можно включить параметр :
```
zero_damaged_pages = on
```
\- его включение заменит повреждённую страницу на блок нулевых байтов («занулит» её) и продолжит выполнение запроса:
```
 postgres=# SET zero_damaged_pages = on;
SET
postgres=# select * from test_table;
WARNING:  invalid page in block 0 of relation "base/5/16388"; zeroing out page
 id 
----
(0 rows)

postgres=# SET zero_damaged_pages = off;
SET
postgres=# select * from test_table;
 id 
----
(0 rows)
```
- или можно включить параметр:
```
ignore_checksum_failure = on
```
этот параметр позволяет попробовать прочитать таблицу, но с возможными искажениями!
```sql
postgres@ubuntu-std2-2-8-30gb:~$ psql -p 5433
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.

postgres=# select * from test_table;
ERROR:  invalid page in block 0 of relation "base/5/16388"
postgres=# SET ignore_checksum_failure = on;
SET
postgres=# select * from test_table;
WARNING:  ignoring checksum failure in block 0 of relation "base/5/16388"
 id 
----
  1
  2
  3
(3 rows)
```
