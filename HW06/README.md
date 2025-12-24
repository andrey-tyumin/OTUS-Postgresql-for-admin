### Домашняя работа по теме:   "MVCC, vacuum и autovacuum".
#### Настройка autovacuum с учетом особеностей производительности
Цель:
-   запустить нагрузочный тест pgbench
-   настроить параметры autovacuum
-   проверить работу autovacuum

Описание/Пошаговая инструкция выполнения домашнего задания:

-   Создать инстанс ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB
-   Установить на него PostgreSQL 15 с дефолтными настройками
-   Создать БД для тестов: выполнить pgbench -i postgres
-   Запустить pgbench -c8 -P 6 -T 60 -U postgres postgres
-   Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла
-   Протестировать заново
-   Что изменилось и почему?
-   Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк
-   Посмотреть размер файла с таблицей
-   5 раз обновить все строчки и добавить к каждой строчке любой символ
-   Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
-   Подождать некоторое время, проверяя, пришел ли автовакуум
-   5 раз обновить все строчки и добавить к каждой строчке любой символ
-   Посмотреть размер файла с таблицей
-   Отключить Автовакуум на конкретной таблице
-   10 раз обновить все строчки и добавить к каждой строчке любой символ
-   Посмотреть размер файла с таблицей
-   Объясните полученный результат
-   Не забудьте включить автовакуум)

Задание со *:  
Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице.  
Не забыть вывести номер шага цикла.

-------
#### Подготовка вм:
поиск подходящего образа с almalinux:
```
pveam available |grep almalinux |grep system
system          almalinux-10-default_20250930_amd64.tar.xz
system          almalinux-9-default_20240911_amd64.tar.xz
```

Скачал образ:
```
pveam download isos almalinux-9-default_20240911_amd64.tar.xz
```
Создал контейнер:
```
pct create 105 isos:vztmpl/almalinux-9-default_20240911_amd64.tar.xz \
--hostname postgresql15 \
--memory 4096 \
--cores 2 \
--rootfs nvme:10 \
--net0 name=eth0,bridge=vmbr0,ip=dhcp \
--password 'postgresql' \
--start
```
ip address:
```
pct exec 105 -- ip a show
```
Поставил sshd и включил его:
```
pct exec 105 -- dnf install openssh-server -y && systemctl enable --now sshd
```
разрешил логин под root:
```
pct exec 105 --- echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config
```
залогинился на контейнер, поставил postgresql 15:
```
dnf update

sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf install -y postgresql15-server
sudo /usr/pgsql-15/bin/postgresql-15-setup initdb
sudo systemctl enable postgresql-15
sudo systemctl start postgresql-15
```
Проверил
```
systemctl status postgresql-15
```
Установил пакет с pgbench:
```
dnf install postgresql-contrib-13.22-1.el9_6.x86_64
su - postgres
```
Подготовил бд к тестам:
```
pgbench -i postgres
```
Запустил тест:
```
pgbench -c8 -P 6 -T 60 -U postgres postgres
```
здесь опции:\
-c clients — устанавливает количество клиентов, которые будут имитировать одновременные сеансы работы с базой данных. По умолчанию установлено значение 1.\
-P - как часто показывать статус\
-Т - время тестирования\

Результат теста:
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 60 s
number of transactions actually processed: 82565
latency average = 5.807 ms
latency stddev = 4.255 ms
tps = 1375.667649 (including connections establishing)
tps = 1375.734979 (excluding connections establishing)
```
внес параметры настройки из дз:
```
# DB Version: 11
# OS Type: linux
# DB Type: dw
# Total Memory (RAM): 4 GB
# CPUs num: 1
# Data Storage: hdd

max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB
```
перезагрузил postgresql:
```
systemctl restart postgresql-15
```

Заново запустил тест:
```
pgbench -c8 -P 6 -T 60 -U postgres postgres
```
Результат:
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 60 s
number of transactions actually processed: 82505
latency average = 5.811 ms
latency stddev = 4.227 ms
tps = 1374.805273 (including connections establishing)
tps = 1374.872097 (excluding connections establishing)
```
Отличий почти никаких нет, 

Создал таблицу:
```sql
CREATE TABLE test_table ( id SERIAL PRIMARY KEY, text_data TEXT NOT NULL );
```
заполнил случайными 1000000 строками 
```sql
INSERT INTO test_table (text_data)
SELECT md5(random()::text)::text
FROM generate_series(1, 1000000);
```
Проверил:
```sql
SELECT count(*) FROM test_table;
  count  
---------
 1000000
(1 row)
```
Посмотрел, где расположен файл с таблицей:
```
postgres=# SELECT PG_relation_filepath('test_table');
 pg_relation_filepath 
----------------------
 base/5/16429
(1 row)
```
размер файла таблицы:
```sql
postgres=# SELECT pg_size_pretty(pg_total_relation_size('test_table'));
 pg_size_pretty 
----------------
 87 MB
(1 row)
```
Обновил все строки в таблице 5 раз этим запросом(5 запусков):
```sql
postgres=# UPDATE test_table SET text_data = text_data || chr(97 + floor(random() * 26)::int);
UPDATE 1000000
```
Количество "мертвых" строк:
```sql
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum 
FROM pg_stat_user_TABLEs WHERE relname = 'test_table';
  relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
------------+------------+------------+--------+-------------------------------
 test_table |    1000000 |          0 |      0 | 2025-12-24 19:27:45.695117+00
(1 row)
```
Автовакуум последний раз был(хотя он виден из пред. запроса):
```sql
postgres=# SELECT last_autovacuum, last_autoanalyze FROM pg_stat_user_tables WHERE relname = 'test_table';
        last_autovacuum        |       last_autoanalyze        
-------------------------------+-------------------------------
 2025-12-24 19:27:45.695117+00 | 2025-12-24 19:27:46.815394+00
(1 row)
```
Еще 5 раз обновил строки в таблице:
```sql
UPDATE test_table SET text_data = text_data || chr(97 + floor(random() * 26)::int);
```
Размер файла с таблицей теперь:
```sql
postgres=# SELECT pg_size_pretty(pg_total_relation_size('test_table'));
 pg_size_pretty 
----------------
 497 MB
(1 row)
```
Отключил автовакуум на таблице test_table:
```sql
ALTER TABLE test_table SET (autovacuum_enabled = false);
```
10 раз обновить все строчки и добавить к каждой строчке любой символ:
```sql
UPDATE test_table SET text_data = text_data || chr(97 + floor(random() * 26)::int);
```
Размер файла с таблицей теперь:
```sql
postgres=# select pg_size_pretty(pg_total_relation_size('test_table'));
 pg_size_pretty 
----------------
 943 MB
(1 row)
```
Размер файла с таблицей сильно увеличился, за счет большого кол-ва "мертвых" записей, которые не удалены, т.к. автовакуум отключен:
```sqs
SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum 
FROM pg_stat_user_TABLEs WHERE relname = 'test_table';
  relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
------------+------------+------------+--------+-------------------------------
 test_table |    1000000 |    9997489 |    999 | 2025-12-24 19:38:46.105975+00
(1 row)
````
из рез-та запроса видно, что "живых" записей 1млн, а "мертвых" записей 10 млн.\
Включил автовакуум обратно, немного подождал и посмотрел:
```sql
postgres=# ALTER TABLE test_table SET (autovacuum_enabled = true);
ALTER TABLE
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum 
FROM pg_stat_user_TABLEs WHERE relname = 'test_table';
  relname   | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
------------+------------+------------+--------+-------------------------------
 test_table |    1000000 |          0 |      0 | 2025-12-24 19:54:13.617275+00
(1 row)

postgres=# select pg_size_pretty(pg_total_relation_size('test_table'));
 pg_size_pretty 
----------------
 943 MB
(1 row)
```
количество "мертвых записей =0, но размер файла таблицы тот же. Запустил vacuum full; 
```sql
postgres=# vacuum full test_table;
VACUUM
postgres=# select pg_size_pretty(pg_total_relation_size('test_table'));
 pg_size_pretty 
----------------
 110 MB
(1 row)
```
Размер таблицы сильно уменьшился, за счет удаления из неё "дыр" оставшихся после удаления "мертвых" записей. Освободившееся пространство возвращено ОС.
