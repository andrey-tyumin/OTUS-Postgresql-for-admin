### Тестирование кластера patroni.
#### Тестирование etcd.
Проверил состояние кластера перед тестом:
"Здоровье" узлов кластера:
```
 etcdctl --endpoints=192.168.77.249:2379,192.168.77.241:2379,192.168.77.105:2379 endpoint health
```
![etcd health](images/etcd-cluster-health-before.png)
Целостность данных:
 ```
 etcdctl --endpoints=192.168.77.249:2379,192.168.77.241:2379,192.168.77.105:2379 endpoint hashkv --write-out=table
```
![etcd hash](etcd-cluster-hash-before.png)
Узнал кто лидер:
```
etcdctl --endpoints=192.168.77.249:2379,192.168.77.241:2379,192.168.77.105:2379 endpoint status --write-out=table | grep "true" | awk '{print $2}'
```
![etcd leader](images/etcd-cluster-leader-before.png)
Отключение ведомого:
```
sudo systemctl stop etcd
```
Проверяем здоровье и целостность данных.
 ```
 etcdctl --endpoints=192.168.77.249:2379,192.168.77.241:2379,192.168.77.105:2379 endpoint health
 etcdctl --endpoints=192.168.77.249:2379,192.168.77.241:2379,192.168.77.105:2379 endpoint hashkv --write-out=table
```
![etcd consistent](images/etcd-cluster-slaveoff-consistent.png)
![etc health](images/etcd-cluster-slaveoff-health.png)
Включил etcd на ведомом обратно:
```
sudo systemctl start etcd
```
Снова проверил здоровье и целостность данных:
 ```
 etcdctl --endpoints=192.168.77.249:2379,192.168.77.241:2379,192.168.77.105:2379 endpoint health
 etcdctl --endpoints=192.168.77.249:2379,192.168.77.241:2379,192.168.77.105:2379 endpoint hashkv --write-out=table
```
![etcd slave on](images/etcd-cluster-slaveon.png)
Отключил мастера:
```
sudo systemctl stop etcd
```
Опять проверил здоровье и целостность данных.
 ```
 etcdctl --endpoints=192.168.77.249:2379,192.168.77.241:2379,192.168.77.105:2379 endpoint health
 etcdctl --endpoints=192.168.77.249:2379,192.168.77.241:2379,192.168.77.105:2379 endpoint hashkv --write-out=table
```
и кто новый лидер:
```
etcdctl --endpoints=192.168.77.249:2379,192.168.77.241:2379,192.168.77.105:2379 endpoint status --write-out=table | grep "true" | awk '{print $2}'
```
![etcd masteroff](images/etcd-cluster-masteroff.png)
Данные консистентны, новый лидер выбран.
Включил бывший мастер обратно:
```
sudo systemctl start etcd
```
Проверил все то же самое:
здоровье:
 ```
 etcdctl --endpoints=192.168.77.249:2379,192.168.77.241:2379,192.168.77.105:2379 endpoint health
```
целостность данных:
 ```
 etcdctl --endpoints=192.168.77.249:2379,192.168.77.241:2379,192.168.77.105:2379 endpoint hashkv --write-out=table
```
и кто новый лидер:
```
etcdctl --endpoints=192.168.77.249:2379,192.168.77.241:2379,192.168.77.105:2379 endpoint status --write-out=table | grep "true" | awk '{print $2}'
```
![etcd master on](images/etcd-cluster-masteron.png)
Данные консистентны, после включения "старого" лидера, он стал ведомым.  
Тестирование etcd успешно.

#### Тестирование patroni.
получил текущее состояние кластера:
```
patronictl -c /etc/patroni/patroni.yml list
```
![patroni first state](images/patroni-stop-leader-before.png)
остановил сервис patroni на лидере:
```
systemctl stop patroni
```
Снова проверил текущее состояние кластера:
```
patronictl -c /etc/patroni/patroni.yml list
```
![patroni stop leader](images/patroni-stop-leader-after.png)
лидер поменялся.

Запустил сервис patroni на ноде, где останавливал:
```
systemctl start patroni
```
![patroni start leader](images/patroni-start-leader.png)
Бывший лидер стал репликой, новый лидер остался лидером.

Ручное переключение мастера.  
Переключил мастер обратно с postgres-2 на postgres-1:
```
patronictl -c /etc/patroni/patroni.yml failover --candidate postgres-1
```
![patroni manual master change](images/patroni-manual-master-change.png)
Мастер успешно переместился на postgres-1
Тестирование patroni успешно.
