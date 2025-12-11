Установка и настройка PostgteSQL в контейнере Docker
Цель:
-   установить PostgreSQL в Docker контейнере
-   настроить контейнер для внешнего подключения
 
Описание/Пошаговая инструкция выполнения домашнего задания:
-   создать ВМ с Ubuntu 20.04/22.04 или развернуть докер любым удобным способом
-   поставить на нем Docker Engine
-   сделать каталог /var/lib/postgres
-   развернуть контейнер с PostgreSQL 15 смонтировав в него /var/lib/postgresql
-   развернуть контейнер с клиентом postgres
-   подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк
-   подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов ЯО/места установки докера
-   удалить контейнер с сервером
-   создать его заново
-   подключится снова из контейнера с клиентом к контейнеру с сервером
-   проверить, что данные остались на месте
-   оставляйте в ЛК ДЗ комментарии что и как вы делали и как боролись с проблемами.
***
#### Создал вм в proxmox:
Крайний занятый id:
```
qm list | awk 'NR>1 {print $1}' | sort -n | tail -1
```
создал вм:
```bash
qm create 112 --name wattos --memory 4096 --cores 1 --net0 virtio,bridge=vmbr0
```
создал диск для вм:
```bash
qm set 112 --scsi0 nvme:64
```
Добавил iso образ:
```bash
qm set 112 --cdrom isos:iso/wattOS-R13.iso
```
установил на неё wattOsR13(debian-based OS).

Установил docker через apt репозиторий по инструкции c офф. сайта: 
https://docs.docker.com/engine/install/debian/#install-using-the-repository
Создал каталог /var/lib/postgresql.
Поиск подходящего образа для контейнера с сервером:
```bash
docker search --filter is-official=true postgres
```
Скачал образ с 15-й версией:
```bash
docker pull postgres:15
```
Запустил сервер: 
```bash
docker run -e POSTGRES_PASSWORD=password -e PGDATA=/var/lib/postgres/15/docker -d --name postgres-server --mount type=bind,src=/var/lib/postgres,dst=/var/lib/postgres postgres:15
```
Проверил что контейнер запустился:
```
docker ps
```
![deploy postgres container](images/postgres-server-deploy.png)
Скачал образ клиента postgres: 
```bash
docker pull codingpuss/postgres-client
```
Запустил контейнер клиента postgres:
```bash
docker run -dit --name=pgclient codingpuss/postgres-client
```
Проверил что контейнер запустился:
```bash
docker ps
```

Подключился к БД postgres в контейнере postgres-server клиентом psql из контейнера pgclient:
```bash
docker exec -it pgclient psql postgresql://postgres:password@172.17.0.2:5432/postgres
```
![deploy psql](images/psql-deploy.png)
Создал таблицу, внес записи:
```sql
CREATE TABLE spp (i serial, amount int); 
INSERT INTO spp(amount) VALUES (1),(2),(3),(4),(5),(6),(7); 
```
Проверил:
```sql
SELECT * FROM spp; 
```
Вышел из контейнера.

Подключился с хоста:
```bash
psql -h 172.17.0.2 -U postgres
```
Проверил данные:
```sql
SELECT * FROM spp; 
```
отключился.

![psql add and check data](images/psql-add-check-data.png)

Удалил контейнер postgres-server:
```bash
docker stop postgres-server
```
```bash
docker rm postgres-server
```
Проверил:
```bash
docker ps -a
```
Снова запустил контейнер postgres-server: 
```bash
docker run -e POSTGRES_PASSWORD=password -e PGDATA=/var/lib/postgres/15/docker -d --name postgres-server --mount type=bind,src=/var/lib/postgres,dst=/var/lib/postgres postgres:15
```
![delete-add-postgres server container](images/delete-run-postgres_server.png)
Подключился к Postgres из контейнера с клиентом:
```bash
docker exec -it pgclient psql postgresql://postgres:password@172.17.0.2:5432/postgres
```
Список таблиц:
```
\dt
```
Проверил внесенные данные в таблице:
```sql
SELECT * FROM spp;
```
Вышел из контейнера.

![check data from client in docker](images/check-data-in-db.png)
Подключился с хоста:
```bash
psql -h 172.17.0.2 -U postgres
```
Проверил данные:
```sql
SELECT * FROM spp; 
```
Данные на месте.

![check data from host](images/check-data-in-db-from-host.png)
