Создание инфраструктуры.  
Сети:  
1. Посмотреть существующие сети:
```
    openstack network list
```
2. Создать сеть:
```
 openstack network create diplom-net
```
3. Посмотреть список подсетей в сети:
```
    openstack subnet list --network diplom-net
```
4. Создать подсеть:
```
    openstack subnet create --network diplom-net --subnet-range 192.168.77.0/24 diplom-subnet
```
5. Просмотр текущих роутеров:
```
    openstack router list
```
6. Создать маршрутизатор:
```
    openstack router create diplom-router
```
7. Привязать подсеть к роутеру:
```
    openstack router add subnet diplom-router diplom-subnet
```
8. Список внешних сетей
```
    openstack network list --external -f value -c Name
```
9. Привязать внешнюю сеть к новому роутеру
```
    openstack router set --external-gateway internet diplom-router
```
![openstack-network](images/openstack-net.png)

#### Создание виртуалок.  
доступные шаблоны конфигурации вм: 
```
openstack flavor list
```
подробное инфо про выбранный шаблон: 
```
openstack flavor show тут-имя-шаблона
```
доступные образы ОС: 
```
openstack image list
```
подробнее про выбранный образ(я выбрал образ с almalinux 9.7): 
```
openstack image show тут-имя-образа
```
группы безопасности: 
```
openstack security group list
```
сделал security group:
```
openstack security group create haproxy
```
добавил правила в security group:
```
openstack security group rule create \
    --protocol tcp --dst-port 22 \
    --remote-ip 0.0.0.0/0 \
    haproxy

openstack security group rule create \
    --protocol tcp --dst-port 80 \
    --remote-ip 0.0.0.0/0 \
    haproxy

openstack security group rule create \
    --protocol tcp --dst-port 443 \
    --remote-ip 0.0.0.0/0 \
    haproxy
```
позднее к security group для haproxy добавлял правила по портам 5432 5433  

Посмотрел список сетей: 
```
openstack network list
```
Доступные ключевые пары: 
```
openstack keypair list
```
Создал диск для вм с haproxy:
```
openstack volume create haproxy \
--size 20 \
--image almalinux-9.7-202602051629.gite7a38aaf \
--availability-zone MS1 \
--bootable
```
Создал сервер с haproxy:
```
openstack server create haproxy \
--volume haproxy \
--network diplom-net \
--flavor STD3-1-2 \
--key-name pg1-vWKyphUl \
--security-group haproxy \
--availability-zone MS1
```
HaProxy нужен будет внешний адрес.  
Просмотрел доступные внешние адреса: 
```
openstack floating ip list
```
Получил новый "внешний" адрес: 
```
openstack floating ip create internet
```
Просмотрел полученный адрес: 
```
openstack floating ip list
```
"Привязал" полученный адрес к вм haproxy: 
```
openstack server add floating ip haproxy 217.16.30.131
```
Проверил, что адрес привязан к вм: 
```
openstack server show haproxy |grep addresses
```
ВМ с балансировщиком haproxy готова.


### Подготовка хостов для etcd

сделал security group:
```
openstack security group create etcd
```
добавил правила в security group:
```
openstack security group rule create \
    --protocol tcp --dst-port 22 \
    --remote-ip 0.0.0.0/0 \
    etcd

openstack security group rule create \
    --protocol tcp --dst-port 2379 \
    --remote-ip 0.0.0.0/0 \
    etcd

openstack security group rule create \
    --protocol tcp --dst-port 2380 \
    --remote-ip 0.0.0.0/0 \
    etcd
```
Создал диски:
```
for i in {1,2,3};
do
openstack volume create etcd-$i \
--size 70 \
--image almalinux-9.7-202602051629.gite7a38aaf \
--availability-zone MS1 \
--bootable;
done
```
Создал серверы:
```
for i in {1,2,3}; do
openstack server create etcd-$i \
--volume etcd-$i \
--network diplom-net \
--flavor STD3-2-4 \
--key-name pg1-vWKyphUl \
--security-group etcd \
--availability-zone MS1;
done
```
Серверы для etcd созданы.

#### Сервера для Postgresql серверов.

сделал security group:
```
openstack security group create postgres
```
добавил правила в security group:
```
openstack security group rule create \
    --protocol tcp --dst-port 22 \
    --remote-ip 0.0.0.0/0 \
    postgres

openstack security group rule create \
    --protocol tcp --dst-port 5432 \
    --remote-ip 0.0.0.0/0 \
    postgres

openstack security group rule create \
    --protocol tcp --dst-port 8008 \
    --remote-ip 0.0.0.0/0 \
    postgres

openstack security group rule create \
    --protocol tcp --dst-port 6432 \
    --remote-ip 0.0.0.0/0 \
    postgres
```
Создал диски для Postgresql серверов:
```
for i in {1,2,3,4};
do
openstack volume create postgres-$i \
--size 50 \
--image almalinux-9.7-202602051629.gite7a38aaf \
--availability-zone MS1 \
--bootable;
done
```
Создал серверы Postgresql:
```
for i in {1,2,3,4}; do
openstack server create postgres-$i \
--volume postgres-$i \
--network diplom-net \
--flavor STD3-2-4 \
--key-name pg1-vWKyphUl \
--security-group etcd \
--availability-zone MS1;
done
```
Сервера для Postgresql созданы.

![OpenStack servers list](images/openstack-servers.png)
