
### Работа с уровнями изоляции транзакции в PostgreSQL

Цель:

-   научиться управлять уровнем изоляции транзакции в PostgreSQL и понимать особенность работы уровней read commited и repeatable read.

Для выполнения задания сделал LXC контейнер в Proxmox-е:
***
Создание контейнера:
список доступных для скачивания LXC контейнеров с postgresql:
```bash
pveam available | grep postgres
```
список локальных storage:
```bash
pvesm status --enabled
```
Скачать LXC образ в storage isos:
```bash
pveam download isos debian-12-turnkey-postgresql_18.1-1_amd64.tar.gz
```
Получить крайний id вм, для нового контейнера берем следующий id:
```bash
qm list | awk 'NR>1 {print $1}' | sort -n | tail -1
```
Создаем LXC контейнер:
```sql
pct create 111 isos:vztmpl/debian-12-turnkey-postgresql_18.1-1_amd64.tar.gz \
  --hostname postgresql18 \
  --memory 4096 \
  --cores 1 \
  --rootfs nvme:64 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --password 'postgresql' \
  --start
  ```
 ***
подключаемся к вм по ssh.
Переходим под пользователя postgres:
```bash
su - postgres
```
Создать таблицу, внести данные, сделать запрос, показать уровень изоляции:
```sql
CREATE TABLE test2 (i serial, amount int); 
INSERT INTO test2(amount) VALUES (100),(500); 
SELECT * FROM test2; 
show transaction isolation level; 
\set AUTOCOMMIT OFF 
```

#### уровень изоляции "READ COMMITTED"
на 1-й консоли:
```sql
begin; 
SELECT * FROM test2; 
```
на 2-й консоли:
```sql
begin; 
UPDATE test2 set amount = 555 WHERE i = 1; 
SELECT * FROM test2; 
commit; 
```
на 1-й консоли:
```sql
commit;
SELECT * FROM test2; 
```
Это демонстрирует "неповторяющееся чтение": до фиксации 2-й транзакции, измененные в ней данные не видны в др. транзакции.


#### Уровень изоляции "REPEATABLE READ"
В 1-й консоли открываю транзакцию, устанавливаю уровень изоляции "REPEATABLE READ" и проверяю:
```sql
begin; 
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ; 
SHOW TRANSACTION ISOLATION LEVEL; 
```
Во 2-й консоли также:
```sql
begin; 
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ; 
SHOW TRANSACTION ISOLATION LEVEL; 
```
В 1-й консоли запрос:
```sql
SELECT * FROM test2; 
```
Во 2-й консоли добавляем запись в таблицу и фиксирую транзакцию:
```sql
INSERT INTO test2(amount) VALUES (777); 
SELECT * FROM test2;
COMMIT;
```
В 1-й консоли запрос:
```sql
SELECT * FROM test2; 
```
смотрю рез-т в 1-й консоли, потом фиксирую:
```sql
COMMIT;
```
смотрю рез-т, вижу разницу.
Это демонстрирует запрет "фантомного чтения", т.е. до фиксации транзакции в ней не видны изменения, вносимые другими транзакциями. 


#### уровень изоляции "SERIALIZABLE"
Создал таблицу, внёс данные:
```sql
DROP TABLE IF EXISTS testS; 
CREATE TABLE testS (i int, amount int); 
INSERT INTO testS VALUES (1,10), (1,20), (2,100), (2,200); 
```
В 1-й консоли начал транзакцию, установил уровень изоляции "SERIALIZABLE", изменил данные:
```sql
BEGIN; 
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE; 
SELECT sum(amount) FROM testS WHERE i = 1; 
INSERT INTO testS VALUES (2,30); 
```
Во 2-й консоли также начал транзакцию, установил уровень изоляции "SERIALIZABLE", изменил данные:
```sql
BEGIN; 
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE; 
SELECT sum(amount) FROM testS WHERE i = 2; 
INSERT INTO testS VALUES (1,300); 
SELECT * FROM testS; 
```
Фиксирую в 1-й консоли, и потом во 2-й.
Во 2-й консоли ошибка коммита.
Это демонстрирует работу с уровнем изоляции "SERIALIZABLE", когда транзакции полностью изолируются друг от друга, и результат выполнения нескольких параллельных транзакций должен быть таким, как если бы они выполнялись последовательно.


