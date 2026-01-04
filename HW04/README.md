### ДЗ по теме "Логический уровень PostgreSQL".

<u>Работа с базами данных, пользователями и правами</u>

Цель:
создание новой базы данных, схемы и таблицы;
создание роли для чтения данных из созданной схемы созданной базы данных;
создание роли для чтения и записи из созданной схемы созданной базы данных;

- Описание/Пошаговая инструкция выполнения домашнего задания:
- создайте новый кластер PostgresSQL 18
- зайдите в созданный кластер под пользователем postgres
- создайте новую базу данных testdb
- зайдите в созданную базу данных под пользователем postgres
- создайте новую схему testnm
- создайте новую таблицу t1 с одной колонкой c1 типа integer
- вставьте строку со значением c1=1
- создайте новую роль readonly
- дайте новой роли право на подключение к базе данных testdb
- дайте новой роли право на использование схемы testnm
- дайте новой роли право на select для всех таблиц схемы testnm
- создайте пользователя testread с паролем test123
- дайте роль readonly пользователю testread
-зайдите под пользователем testread в базу данных testdb
- сделайте select * from t1;
- получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)
- напишите что именно произошло в тексте домашнего задания
- у вас есть идеи почему? ведь права то дали?
- посмотрите на список таблиц
- подсказка в шпаргалке под пунктом 20
- а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)
- вернитесь в базу данных testdb под пользователем postgres
- удалите таблицу t1
- создайте ее заново но уже с явным указанием имени схемы testnm
- вставьте строку со значением c1=1
- зайдите под пользователем testread в базу данных testdb
- сделайте select * from testnm.t1;
- получилось?
- есть идеи почему? если нет - смотрите шпаргалку
- как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку
- сделайте select * from testnm.t1;
- получилось?
- есть идеи почему? если нет - смотрите шпаргалку
- сделайте select * from testnm.t1;
- получилось?
- ура!
- теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
- а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?
- есть идеи как убрать эти права? если нет - смотрите шпаргалку
- если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды
- теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);
- расскажите что получилось и почему.

---

ВМ создал в VK Cloud.
Добавил репозиторий и установил Postgresql-18:
```
sudo apt install -y postgresql-common  
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
sudo apt install postgresql-18
```

Создал базу, подключился к ней:
```bash
root@ubuntu-std2-2-8-30gb:/home/ubuntu# su - postgres
```
```sql
postgres@ubuntu-std2-2-8-30gb:~$ psql
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.

postgres=# CREATE DATABASE testdb;
CREATE DATABASE
postgres=# \l
                                                 List of databases
   Name    |  Owner   | Encoding | Locale Provider | Collate |  Ctype  | Locale | ICU Rules |   Access privileges   
-----------+----------+----------+-----------------+---------+---------+--------+-----------+-----------------------
 postgres  | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |        |           | 
 template0 | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |        |           | =c/postgres          +
           |          |          |                 |         |         |        |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |        |           | =c/postgres          +
           |          |          |                 |         |         |        |           | postgres=CTc/postgres
 testdb    | postgres | UTF8     | libc            | C.UTF-8 | C.UTF-8 |        |           | 
(4 rows)

postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# \conninfo
           Connection Information
      Parameter       |        Value        
----------------------+---------------------
 Database             | testdb
 Client User          | postgres
 Socket Directory     | /var/run/postgresql
 Server Port          | 5432
 Options              | 
 Protocol Version     | 3.0
 Password Used        | false
 GSSAPI Authenticated | false
 Backend PID          | 4746
 SSL Connection       | false
 Superuser            | on
 Hot Standby          | off
```
Создал схему, таблицу, роль, дал роли право на использование и select таблиц схемы:
```sql
testdb=# create schema testnm;
CREATE SCHEMA
testdb=# create table t1 (c1 int);
CREATE TABLE
testdb=# insert into t1(c1) VALUES(1);
INSERT 0 1
testdb=# CREATE ROLE readonly;
CREATE ROLE
testdb=# GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT
testdb=# \dn
      List of schemas
  Name  |       Owner       
--------+-------------------
 public | pg_database_owner
 testnm | postgres
(2 rows)

testdb=# GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
```
Для того, чтобы подключаться к серверу под пользователем testread добавил роли testread право на login:
```sql
postgres=# ALTER ROLE testread LOGIN;
ALTER ROLE
postgres=# \du
                             List of roles
 Role name |                         Attributes                         
-----------+------------------------------------------------------------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS
 readonly  | Cannot login
 testread  |
 ```
 в файл pg_hba.conf добавил:
 ```
 local   all             all                                     md5
```
 Подключился к БД:
 ```sql
 root@ubuntu-std2-2-8-30gb:/home/ubuntu# psql -U testread -d testdb;
Password for user testread: 
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.

testdb=> \conninfo
           Connection Information
      Parameter       |        Value        
----------------------+---------------------
 Database             | testdb
 Client User          | testread
 Socket Directory     | /var/run/postgresql
 Server Port          | 5432
 Options              | 
 Protocol Version     | 3.0
 Password Used        | true
 GSSAPI Authenticated | false
 Backend PID          | 14741
 SSL Connection       | false
 Superuser            | off
 Hot Standby          | off
(12 rows)
```
Сделал SELECT из t1:
```sql
testdb=> SELECT * FROM t1;
ERROR:  permission denied for table t1
```
Нет доступа, Посмотрел доп. инфо:
```sql
testdb=> \dn+
                                       List of schemas
  Name  |       Owner       |           Access privileges            |      Description       
--------+-------------------+----------------------------------------+------------------------
 public | pg_database_owner | pg_database_owner=UC/pg_database_owner+| standard public schema
        |                   | =U/pg_database_owner                   | 
 testnm | postgres          | postgres=UC/postgres                  +| 
        |                   | readonly=U/postgres                    | 
(2 rows)

testdb=> \d
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | t1   | table | postgres
```
Получается, что нет прав на select из t1, т.к. роли readonly, по заданию, выданы права на:
- connect к testdb
- usege и select из schema testnm.
- 
но, на select из t1 (она в schema public) прав не выдавалось.
Удалил t1, создал заново с указанием схемы, внес строку:
```sql
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# DROP TABLE t1;
DROP TABLE
testdb=# \dt
Did not find any tables.
testdb=# 
testdb=# CREATE TABLE testnm.t1(c1 int);
CREATE TABLE
testdb=# INSERT INTO testnm.t1(c1) VALUES(1);
INSERT 0 1
```
Зашел под пользователем testread, сделал select:
```sql
root@ubuntu-std2-2-8-30gb:/home/ubuntu# psql -U testread -d testdb;
Password for user testread: 
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.

testdb=> select * FROM testnm.t1;
ERROR:  permission denied for table t1
```
Ошибка, т.к. права на новые объекты не выдавались. Исправим:
```sql
testdb=# GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
```
Проверка:
```sql
root@ubuntu-std2-2-8-30gb:/home/ubuntu# psql -U testread -d testdb;
Password for user testread: 
psql (18.1 (Ubuntu 18.1-1.pgdg24.04+2))
Type "help" for help.

testdb=> select * FROM testnm.t1;
 c1 
 ----
  1
(1 row)
```
Успешно.
Чтобы в дальнейшем не приходилось выдавать права на создаваемые в схеме объекты:
```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA testnm GRANT SELECT ON TABLES TO testread;
```
сделал select:
```sql
testdb=> select * FROM testnm.t1;
 c1 
----
  1
(1 row)
```
Попробовал создать таблицу t2:
```sql
testdb=> create table t2(c1 integer); 
ERROR:  permission denied for schema public
testdb=> create table testnm.t2(c1 integer); 
ERROR:  permission denied for schema testnm
```
  т.к. права на создание таблиц я не выдавал, только на select.
Исправил:
```sql
testdb=# GRANT CREATE ON SCHEMA testnm TO testread;
GRANT
testdb=# ALTER DATABASE testdb SET SEARCH_PATH TO testnm;
ALTER DATABASE
```
Проверка:
```sql
testdb=> create table t2(c1 integer);
CREATE TABLE
testdb=> insert into t2 values (2);
INSERT 0 1
testdb=> create table t3(c1 integer);
CREATE TABLE
testdb=> insert into t2 values (2);
INSERT 0 1
testdb=> \dt
          List of tables
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 testnm | t1   | table | postgres
 testnm | t2   | table | testread
 testnm | t3   | table | testread
(3 rows)

testdb=> select * FROM t2;
 c1 
----
  2
  2
(2 rows)

testdb=> select * FROM t3;
 c1 
----
(0 rows)
```
Успешно.
