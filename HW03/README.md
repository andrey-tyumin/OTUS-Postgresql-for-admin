### ДЗ по теме: "Физический уровень PostgreSQL"

---

#### Установка и настройка PostgreSQL

Цель:

создавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему;  
переносить содержимое базы данных PostgreSQL на дополнительный диск;  
переносить содержимое БД PostgreSQL между виртуальными машинами;
  

Описание/Пошаговая инструкция выполнения домашнего задания:

-   создайте виртуальную машину c Ubuntu 22.04/24.04 LTS в ЯО/Virtual Box/Docker
-   поставьте на нее PostgreSQL через sudo apt
-   проверьте что кластер запущен через sudo -u postgres pg_lsclusters
-   зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым  
    postgres=# create table test(c1 text);  
    postgres=# insert into test values('1');  
    \q
-   остановите postgres например через sudo -u postgres pg_ctlcluster 17 main stop
-   создайте новый диск к ВМ размером 10GB
-   добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk
-   проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb -  [https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux](https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux "https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux")
-   перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)
-   сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/
-   перенесите содержимое /var/lib/postgres/17 в /mnt/data - mv /var/lib/postgresql/15/mnt/data
-   попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 17 main start
-   напишите получилось или нет и почему
-   задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/17/main который надо поменять и поменяйте его
-   напишите что и почему поменяли
-   попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 17 main start
-   напишите получилось или нет и почему
-   зайдите через через psql и проверьте содержимое ранее созданной таблицы
-   **задание со звездочкой**  *:  
    не удаляя существующий инстанс ВМ сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.

---

ВМ установил в vk cloud.
Параметры вм: Ubuntu 24.04, 1 cpu, 4Mb ram, 30Gb hdd.
Обновил, установил Postgresql.
```bash
apt update & apt upgrade -y
apt install postgresql
```
Проверил:
```bash
pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```
Переключился на пользователя postgres:
```bash
su - postgres
```
Подключился к СУБД, создал таблицу, внес данные:
```sql
psql
postgres=# create table test(c1 text);
CREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
```
Вышел, остановил кластер:
```bash
pg_ctlcluster 16 main stop
```
---

Создал диск 10Гб и подключил к инстансу.
```bash
root@ubuntu-std3-1-4-30gb:/home/ubuntu# lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sr0      11:0    1  538K  0 rom  
vda     253:0    0   30G  0 disk 
├─vda1  253:1    0   29G  0 part /
├─vda14 253:14   0    4M  0 part 
├─vda15 253:15   0  106M  0 part /boot/efi
└─vda16 259:0    0  913M  0 part /boot
vdb     253:16   0   10G  0 disk 
root@ubuntu-std3-1-4-30gb:/home/ubuntu# fdisk /dev/vdb

Welcome to fdisk (util-linux 2.39.3).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS (MBR) disklabel with disk identifier 0xe3d8b1bd.

Command (m for help): p
Disk /dev/vdb: 10 GiB, 10737418240 bytes, 20971520 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xe3d8b1bd

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 
First sector (2048-20971519, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-20971519, default 20971519): 

Created a new partition 1 of type 'Linux' and of size 10 GiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

root@ubuntu-std3-1-4-30gb:/home/ubuntu# mkfs.ext4 /dev/vdb1
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 2621184 4k blocks and 655360 inodes
Filesystem UUID: bdefbfa5-3e29-4248-83ae-950b4ce4298e
Superblock backups stored on blocks: 
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 

root@ubuntu-std3-1-4-30gb:/home/ubuntu# mount /dev/vdb1 /mnt
root@ubuntu-std3-1-4-30gb:/home/ubuntu# findmnt --real -o +AVAIL,SIZE
TARGET        SOURCE     FSTYPE OPTIONS                                                                                               AVAIL   SIZE
/             /dev/vda1  ext4   rw,relatime,discard,errors=remount-ro,commit=30                                                       25.4G    28G
├─/boot       /dev/vda16 ext4   rw,relatime                                                                                          701.7M 880.4M
│ └─/boot/efi /dev/vda15 vfat   rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro  98.2M 104.3M
└─/mnt        /dev/vdb1  ext4   rw,relatime                                                                                            9.2G   9.7G
```
Прописывать в /etc/fstab не стал - т.к. "на один раз", и перезагрузки не требуется для работы сервиса.

Остановил кластер.

PGDATA установлен в /var/lib/postgresql/16/main - это видно  по выводу pg_lsclusters:
```bash
root@ubuntu-std3-1-4-30gb:/home/ubuntu# pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 down   postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```
Перенес каталог в /mnt:
```bash
root@ubuntu-std3-1-4-30gb:/home/ubuntu# mv /var/lib/postgresql/16 /mnt
root@ubuntu-std3-1-4-30gb:/home/ubuntu# ls -al /mnt
total 28
drwxr-xr-x  4 postgres postgres  4096 Jan  2 17:30 .
drwxr-xr-x 22 root     root      4096 Jan  2 15:10 ..
drwxr-xr-x  3 postgres postgres  4096 Jan  2 16:59 16
drwx------  2 postgres postgres 16384 Jan  2 17:14 lost+found
root@ubuntu-std3-1-4-30gb:/home/ubuntu# ls -al /var/lib/postgresql/
total 16
drwxr-xr-x  2 postgres postgres 4096 Jan  2 17:30 .
drwxr-xr-x 45 root     root     4096 Jan  2 16:59 ..
-rw-------  1 postgres postgres   42 Jan  2 17:04 .bash_history
-rw-------  1 postgres postgres   61 Jan  2 17:04 .psql_history
```
Попробовал запустить кластер:
```bash
root@ubuntu-std3-1-4-30gb:/home/ubuntu# pg_ctlcluster 16 main start
Error: /var/lib/postgresql/16/main is not accessible or does not exist
```
Ошибка, т.к. data каталог не найден по старому пути.
В файле  /etc/postgresql/16/main/postgresql.conf поменял параметр data_directory :
```
#data_directory = '/var/lib/postgresql/16/main'         # use data in another directory
data_directory = '/mnt/16/main'
```
Запустил кластер:
```bash
pg_ctlcluster 16 main start
```
Проверил:
```bash
root@ubuntu-std3-1-4-30gb:/home/ubuntu# pg_lsclusters
Ver Cluster Port Status Owner    Data directory Log file
16  main    5432 online postgres /mnt/16/main   /var/log/postgresql/postgresql-16-main.log
```
Кластер запустился, т.к. теперь указан верный путь к каталогу с данными.

Задание со '*'.
Скопировал вм, 
```bash
apt update & apt upgrade -y
```
Установил Postgresql:
```bash
apt install postgresql
```
Проверил:
```bash
root@second:/home/ubuntu# pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```
Остановил:
```bash
root@second:/home/ubuntu# pg_ctlcluster 16 main stop
root@second:/home/ubuntu# pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 down   postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```
На первом хосте также остановил кластер и отмонтировал раздел на втором диске:
```bash
ubuntu@ubuntu-std3-1-4-30gb:~$ sudo su
root@ubuntu-std3-1-4-30gb:/home/ubuntu# pg_ctlcluster 16 main stop
root@ubuntu-std3-1-4-30gb:/home/ubuntu# pg_lsclusters
Ver Cluster Port Status Owner    Data directory Log file
16  main    5432 down   postgres /mnt/16/main   /var/log/postgresql/postgresql-16-main.log
root@ubuntu-std3-1-4-30gb:/home/ubuntu# umount /mnt
```
Отключил диск от первого инстанса, подключил ко второму:
```bash
root@second:/home/ubuntu# lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sr0      11:0    1  538K  0 rom  
vda     253:0    0   30G  0 disk 
├─vda1  253:1    0   29G  0 part /
├─vda14 253:14   0    4M  0 part 
├─vda15 253:15   0  106M  0 part /boot/efi
└─vda16 259:1    0  913M  0 part /boot
vdb     253:16   0   10G  0 disk 
└─vdb1  253:17   0   10G  0 part 
root@second:/home/ubuntu# mount /dev/vdb1 /mnt
root@second:/home/ubuntu# findmnt --real -o +AVAIL,SIZE
TARGET        SOURCE     FSTYPE OPTIONS                                                                                               AVAIL   SIZE
/             /dev/vda1  ext4   rw,relatime,discard,errors=remount-ro,commit=30                                                       25.4G    28G
├─/boot       /dev/vda16 ext4   rw,relatime                                                                                          701.7M 880.4M
│ └─/boot/efi /dev/vda15 vfat   rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro  98.2M 104.3M
└─/mnt        /dev/vdb1  ext4   rw,relatime                                                                                            9.2G   9.7G
root@second:/home/ubuntu# ls -al /mnt
total 28
drwxr-xr-x  4 postgres postgres  4096 Jan  2 17:30 .
drwxr-xr-x 22 root     root      4096 Jan  2 17:45 ..
drwxr-xr-x  3 postgres postgres  4096 Jan  2 16:59 16
drwx------  2 postgres postgres 16384 Jan  2 17:14 lost+found
```
Также поменял параметр data_directory в файле /etc/postgresql/16/main/postgresql.conf, запустил кластер, проверил:
```bash
root@second:/home/ubuntu# vi /etc/postgresql/16/main/postgresql.conf
root@second:/home/ubuntu# pg_ctlcluster 16 main start
root@second:/home/ubuntu# pg_lsclusters
Ver Cluster Port Status Owner    Data directory Log file
16  main    5432 online postgres /mnt/16/main   /var/log/postgresql/postgresql-16-main.log
```
Кластер успешно запустился. Data directory указан /mnt/16/main - т.е. на внешнем, перенесенном с 1-й вм, диске.
