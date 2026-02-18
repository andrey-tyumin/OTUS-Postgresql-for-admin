Домашнее задание по теме "Резервное копирование и восстановление "

Бэкапы

Цель:
применить логический бэкап;
восстановиться из бэкапа;

Описание/Пошаговая инструкция выполнения домашнего задания:
Развернуть PostgreSQL (ВМ/Docker).
В БД test_db создать схему my_schema и две одинаковые таблицы (table1, table2).
Заполнить table1 100 строками с помощью generate_series.
Создать каталог /var/lib/postgresql/backups/ под пользователем postgres.
Бэкап через COPY: Выгрузить table1 в CSV командой \copy.
Восстановление из COPY: Загрузить данные из CSV в table2.
Бэкап через pg_dump: создать кастомный сжатый дамп (-Fc) только схемы my_schema.
Восстановление через pg_restore: В новую БД restored_db восстановить только table2 из дампа.

---

1.В БД test_db создать схему my_schema и две одинаковые таблицы (table1, table2).
Создал БД:
```sql
CREATE DATABASE test_db;
```
Подключился:
```sql
\c test_db
```
Создал схему
```sql
CREATE SCHEMA my_schema;
```
Проверил:
```sql
\dn
```
Создал таблицы:
```sql
CREATE TABLE my_schema.table1(id int);
CREATE TABLE public.table2(id int);
```
2.Заполнить table1 100 строками с помощью generate_series.
```sql
INSERT INTO my_schema.table1 SELECT generate_series(1,10000);
```
Проверил:
```sql
SELECT count(*) FROM my_schema.table1;
```
![createdb](images/HW13-1.png)

3.Создать каталог /var/lib/postgresql/backups/ под пользователем postgres.
```sql
\! mkdir /var/lib/pgsql/backups
```
4.Бэкап через COPY: Выгрузить table1 в CSV командой \copy.
```sql
COPY my_schema.table1 TO '/var/lib/pgsql/backups/table1.backup';
```
5.Восстановление из COPY: Загрузить данные из CSV в table2.
```sql
COPY public.table2 FROM '/var/lib/pgsql/backups/table1.backup';
```
Проверка:
```sql
SELECT count(*) FROM public.table2;
```
![COPY](images/HW13-3.png)

6.Бэкап через pg_dump: создать кастомный сжатый дамп (-Fc) только схемы my_schema.
```sql
\! pg_dump -n my_schema -Fc -d test_db -f /var/lib/pgsql/backups/test_db_schema.backup
```
Проверил:
```sql
\! pg_restore --list /var/lib/pgsql/backups/test_db_schema.backup
```
7.Восстановление через pg_restore: В новую БД restored_db восстановить только table2 из дампа.
```sql
DROP DATABASE test_db;
\! pg_restore -C -d postgres /var/lib/pgsql/backups/test_db_schema.backup
\l
ALTER DATABASE test_db RENAME TO restored_db;
```
Проверка:
```sql\c restored_db
\dn
```
![pg_dump](images/HW13-2.png)
