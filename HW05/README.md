### Нагрузочное тестирование и тюнинг PostgreSQL

Цель:

-   сделать нагрузочное тестирование PostgreSQL
-   настроить параметры PostgreSQL для достижения максимальной производительности

Нагрузочное тестирование:
-   развернуть виртуальную машину любым удобным способом
-   поставить на неё PostgreSQL 18 любым способом
-   настроить кластер PostgreSQL 18 на максимальную производительность не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины
-   нагрузить кластер через утилиту через утилиту pgbench ([https://postgrespro.ru/docs/postgrespro/18/pgbench](https://postgrespro.ru/docs/postgrespro/18/pgbench "https://postgrespro.ru/docs/postgrespro/18/pgbench"))
-   написать какого значения tps удалось достичь, показать какие параметры в какие значения устанавливали и почему
  
Задание со *: аналогично протестировать через утилиту  [https://github.com/Percona-Lab/sysbench-tpcc](https://github.com/Percona-Lab/sysbench-tpcc "https://github.com/Percona-Lab/sysbench-tpcc")  (требует установки [https://github.com/akopytov/sysbench](https://github.com/akopytov/sysbench "https://github.com/akopytov/sysbench"))

------------
 Создаю LXC контейнер:
```bash
pct create 105 isos:vztmpl/debian-12-turnkey-postgresql_18.1-1_amd64.tar.gz \
--hostname postgresql \
--memory 4096 \
--cores 1 \
--rootfs nvme:64 \
--net0 name=eth0,bridge=vmbr0,ip=dhcp \
--password 'postgresql' \
--start
```
Проверяю список вм, ищу новую:
```bash
pct list
```
Узнать ip адрес:
```bash
pct exec 105 -- ip addr show
```
захожу под postgres пользователя:
```sql
su - postgres
```
создать БД для теста:
```bash
psql -c "create database test"
```
инициализация БД test для тестирования:
```bash
postgres@postgresql:~$ pgbench -i test
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.11 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.35 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.13 s, vacuum 0.10 s, primary keys 0.10 s).
```
1-й тест с pgbench:
```bash
pgbench -c 50 -j 2 -P 10 -T 60 test
```
Здесь используемые аргументы:\
-c - количество соединений\
-j - количество потоков\
-P - как часто показывать статус\
-Т время тестирования\

Вывод(немного обрезал, оставил только нужное):
```bash
pgbench (15.7 (Debian 15.7-0+deb12u1))

number of transactions actually processed: 39048
number of failed transactions: 0 (0.000%)
latency average = 76.750 ms
latency stddev = 56.968 ms
initial connection time = 130.800 ms
tps = 650.398043 (without initial connection time)
```
Добавил след. рекомендации по CPU:\
*max_worker_processes* – максимальное число фоновых процессов, которое может поддерживаться в кластере (<= числу ядер ЦПУ)\
*max_parallel_workers* – максимальное число рабочих процессов, котороекластер сможет поддерживать для параллельных операций (<= числу ядер ЦПУ)\
*max_parallel_workers_per_gather* – максимальное число рабочих процессов, которые могут параллельно использоваться при построении\
плана запроса (max_parallel_workers / 2)\
*max_parallel_maintenance_workers* – максимальное число рабочих процессов, которые могут запускаться одной служебной командой (max_parallel_workers / 2)\
Запустил такой же тест:
```bash
postgres@postgresql:~$ pgbench -c 50 -j 2 -P 10 -T 60 test
pgbench (15.7 (Debian 15.7-0+deb12u1))

number of transactions actually processed: 59720
number of failed transactions: 0 (0.000%)
latency average = 50.217 ms
latency stddev = 57.724 ms
initial connection time = 66.891 ms
tps = 994.820520 (without initial connection time)
```
Добавил рекомендации по RAM:\
shared_buffers (25% – 40% ОЗУ) - задаёт объём памяти, который будет использовать сервер баз данных для буферов в разделяемой памяти.\
effective_cache_size (2/3 ОЗУ) -определяет для планировщика ориентировочную оценку эффективного размера дискового кеша, доступного для одного запроса. shared_buffers + effective_cache_size <= ОЗУ RAM\
temp_buffers (8 Mb) - используется для хранения временных таблиц.\
work_mem (4 Mb) - используется на этапе выполнения запроса для сортировок строк.\
maintenance_work_mem (64 Mb) - используется служебными операциями типа VACUUM и REINDEX. параметр задаёт максимальный объём памяти для операций обслуживания базы данных.рекомендуют установить maintenance_work_mem на значение выше, чем work_mem, так как операции обслуживания обычно требуют больше памяти и менее часты, чем регулярные операции запросов work_mem <= maintenance_work_mem.
```bash
postgres@postgresql:~$ pgbench -c 50 -j 2 -P 10 -T 60 test
pgbench (15.7 (Debian 15.7-0+deb12u1))

number of transactions actually processed: 58847
number of failed transactions: 0 (0.000%)
latency average = 50.956 ms
latency stddev = 57.326 ms
initial connection time = 70.126 ms
tps = 980.416674 (without initial connection time)
```
С опциями по памяти я "поиграл подольше", но эффекта это не дало, предполагаю, что из-за общего малого объема памяти на вм.

Добавил рекомендации по диску:\
checkpoint_timeout - максимальное время между автоматическими контрольными точками(между сбросом "грязных" страниц на диск) диапазон от 30сек до 24часов. рекомендуется от 10 минут до часа.\
checkpoint_completion_target – чем ближе к единице тем менее резкими будут скачки I/O при операциях checkpoint (определяет, какую часть времени между двумя соседними контрольными точками следует использовать для записи страниц) рекомендуется < 0.9\
effective_io_concurrency – число параллельных операций ввода/вывода (по количеству дисков, для SSD – несколько сотен)\
random_page_cost – отношение рандомного чтения к последовательному.Рекомендуется для SSD дисков 1.1 – 1.25, для SAS дисков 1.25 - 2. \
wal_recycle при переработке PostgreSQL переименовывает файлы WAL. В файловых системах с копированием \
при записи (COW) создание новых файлов WAL быстрее. рекомендуется отключить параметр.\

wal_init_zero - По умолчанию пространство WAL выделяется перед вставкой записей WAL - замедляет операции WAL в файловых \
системах COW. рекомендуется отключить параметр.\

wal_buffers - объём пространства, доступного серверным частям для записи данных WAL в память, чтобы затем WALWriter мог записать их в журнал WAL на диске в фоновом режиме. Рекомендуется 128Мб.\

Отключить метку времени доступа atime для файлов данных для диска, где расположены данные - в опциях строки монтирования указать noatime и перемонтировать раздел.
Провел тест:\
```bash
postgres@postgresql:~$ pgbench -c 50 -j 2 -P 10 -T 60 test
pgbench (15.7 (Debian 15.7-0+deb12u1))
number of transactions actually processed: 58454
number of failed transactions: 0 (0.000%)
latency average = 51.297 ms
latency stddev = 57.245 ms
initial connection time = 77.234 ms
tps = 973.770890 (without initial connection time)
```
Для эффекта, самые действенные опции:\
fsync – данные журнала принудительно сбрасываются на диск с кэша ОС.\
synchronous_commit – транзакция завершается только когда данные фактически сброшены на диск\
```
postgres@postgresql:~$ pgbench -c 50 -j 2 -P 10 -T 60 test
pgbench (15.7 (Debian 15.7-0+deb12u1))
number of transactions actually processed: 99506
number of failed transactions: 0 (0.000%)
latency average = 30.112 ms
latency stddev = 35.337 ms
initial connection time = 106.058 ms
tps = 1658.646442 (without initial connection time)
```
Итоговый набор опций :
```
max_worker_processes = 2
max_parallel_workers = 2
max_parallel_workers_per_gather = 1
shared_buffers = 1GB
effective_cache_size = 2GB
temp_buffers = 16MB
work_mem = 32MB 
maintenance_work_mem = 64MB
checkpoint_timeout = 10min
checkpoint_completion_target = 0.9
effective_io_concurrency = 200
random_page_cost = 1.2
wal_recycle = off
wal_init_zero = off 
wal_buffers = 128MB 
fsync = off
synchronous_commit = off
```
Рез-т 1-го теста:
```
tps = 650.398043 (without initial connection time)
```
Рез-т финального теста:
```
tps = 1658.646442 (without initial connection time)
```

-----
#### Sysbench.
Установка:
```bash
curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | bash
apt -y install sysbench
```
Создать БД и пользователя:
```sql
CREATE USER sbtest WITH PASSWORD 'password';
CREATE DATABASE sbtest;
GRANT ALL PRIVILEGES ON DATABASE sbtest TO sbtest;
GRANT CREATE ON SCHEMA public TO sbtest;
```
Добавил в pg_hba.conf:
```
host    sbtest          sbtest          192.168.0.0/24         md5
```
Перезапустил postgresql.
Проверил вход под пользователем sbtest:
```bash
postgres@postgresql:~$ psql -U sbtest -h 192.168.0.40 -p 5432 -d sbtest -W
Password: 
psql (15.7 (Debian 15.7-0+deb12u1))
Type "help" for help.

sbtest=> 
```
Подготовка таблиц:
```bash
sysbench \
    --db-driver=pgsql \
--table_size=10000 \
--tables=1 \
--threads=10 \
--pgsql-host=192.168.0.40 \
--pgsql-port=5432 \
--pgsql-user=sbtest \
--pgsql-db=sbtest \
--pgsql-password=password \
oltp_common \
prepare
```
Запуск теста:
```bash
sysbench \
    --db-driver=pgsql \
--threads=100 \
--pgsql-host=192.168.0.40 \
--pgsql-port=5432 \
--pgsql-user=sbtest \
--pgsql-db=sbtest \
--pgsql-password=password \
$1 \
run
```
Вывод теста:
```
postgres@postgresql:/usr/share/sysbench$ sysbench     --db-driver=pgsql --table_size=10000 --tables=1 --threads=10 --pgsql-host=192.168.0.40 --pgsql-port=5432 --pgsql-user=sbtest --pgsql-db=sbtest --pgsql-password=password oltp_insert run
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 10
Initializing random number generator from current time


Initializing worker threads...

Threads started!

SQL statistics:
    queries performed:
        read:                            0
        write:                           127517
        other:                           0
        total:                           127517
    transactions:                        127517 (12717.23 per sec.)
    queries:                             127517 (12717.23 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          10.0247s
    total number of events:              127517

Latency (ms):
         min:                                    0.09
         avg:                                    0.78
         max:                                   31.87
         95th percentile:                        1.32
         sum:                                99822.62

Threads fairness:
    events (avg/stddev):           12751.7000/602.73
    execution time (avg/stddev):   9.9823/0.01
```
Еще один тест:
```
postgres@postgresql:/usr/share/sysbench$ sysbench     --db-driver=pgsql --table_size=10000 --tables=1 --threads=10 --pgsql-host=192.168.0.40 --pgsql-port=5432 --pgsql-user=sbtest --pgsql-db=sbtest --pgsql-password=password oltp_read_only run
```
```
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 10
Initializing random number generator from current time


Initializing worker threads...

Threads started!

SQL statistics:
    queries performed:
        read:                            169330
        write:                           0
        other:                           24190
        total:                           193520
    transactions:                        12095  (1207.37 per sec.)
    queries:                             193520 (19317.90 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          10.0157s
    total number of events:              12095

Latency (ms):
         min:                                    6.42
         avg:                                    8.27
         max:                                   25.79
         95th percentile:                        9.22
         sum:                               100061.20

Threads fairness:
    events (avg/stddev):           1209.5000/7.03
    execution time (avg/stddev):   10.0061/0.00
```
И еще один тест:
```
postgres@postgresql:/usr/share/sysbench$ sysbench     --db-driver=pgsql --table_size=10000 --tables=1 --threads=10 --pgsql-host=192.168.0.40 --pgsql-port=5432 --pgsql-user=sbtest --pgsql-db=sbtest --pgsql-password=password oltp_delete run
```
```
sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

Running the test with following options:
Number of threads: 10
Initializing random number generator from current time


Initializing worker threads...

Threads started!

SQL statistics:
    queries performed:
        read:                            0
        write:                           4596
        other:                           266719
        total:                           271315
    transactions:                        271315 (27091.53 per sec.)
    queries:                             271315 (27091.53 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          10.0127s
    total number of events:              271315

Latency (ms):
         min:                                    0.05
         avg:                                    0.37
         max:                                    6.67
         95th percentile:                        0.57
         sum:                                99901.51

Threads fairness:
    events (avg/stddev):           27131.5000/248.51
    execution time (avg/stddev):   9.9902/0.00
```

