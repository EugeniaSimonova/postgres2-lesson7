# postgres2-lesson7

Создала VM в Yandex Cloud и поставила PostgreSQL 15.
1. проверьте что кластер запущен через sudo -u postgres pg_lsclusters:
````
$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
````
2. Зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым:
````
$ sudo -su postgres
postgres@eek-ubuntu:/home/jenny-vm$ psql

postgres=# create table test(c1 text);
CREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
````
3. остановите postgres например через `sudo -u postgres pg_ctlcluster 15 main stop`

````
postgres@eek-ubuntu:/home/jenny-vm$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
````
4. создайте новый диск размером например 10GB и добавьте свеже-созданный диск к виртуальной машине.
   Добавила диск в Yandex Cloud

5. Проинициализируйте диск согласно инструкции и подмонтировать файловую систему:
   
````
$ sudo parted -l | grep Error
Error: /dev/vdb: unrecognised disk label

~$ lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda    252:0    0  40G  0 disk 
├─vda1 252:1    0   1M  0 part 
└─vda2 252:2    0  40G  0 part /
vdb    252:16   0  10G  0 disk 

$ sudo parted /dev/vdb mklabel gpt
Information: You may need to update /etc/fstab.

$ sudo parted -a opt /dev/vdb mkpart primary ext4 0% 100%
Information: You may need to update /etc/fstab.

$ lsblk                                              
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda    252:0    0  40G  0 disk 
├─vda1 252:1    0   1M  0 part 
└─vda2 252:2    0  40G  0 part /
vdb    252:16   0  10G  0 disk 
└─vdb1 252:17   0  10G  0 part 

$ sudo mkfs.ext4 -L data1 /dev/vdb1
mke2fs 1.45.5 (07-Jan-2020)
Creating filesystem with 2620928 4k blocks and 655360 inodes
Filesystem UUID: b2172a0f-3e1f-4aa5-8456-6eab593238c7
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 

$sudo mkdir -p /mnt/data

$ sudo mount -o defaults /dev/vdb1 /mnt/data

$ df -h -x tmpfs
Filesystem      Size  Used Avail Use% Mounted on
udev            3.9G     0  3.9G   0% /dev
/dev/vda2        40G  2.9G   35G   8% /
/dev/vdb1       9.8G   24K  9.3G   1% /mnt/data

$ sudo nano /etc/fstab
````
````
#/etc/fstab content

/dev/vdb1 /mnt/data ext4 defaults 0 2
````
6. Перезагрузились.

````
$ sudo df -h -x tmpfs
Filesystem      Size  Used Avail Use% Mounted on
udev            3.9G     0  3.9G   0% /dev
/dev/vda2        40G  2.9G   35G   8% /
/dev/vdb1       9.8G   24K  9.3G   1% /mnt/data
````

7. сделайте пользователя postgres владельцем /mnt/data - `sudo chown -R postgres:postgres /mnt/data/`;
перенесите содержимое /var/lib/postgresql/15 в /mnt/data - `sudo mv /var/lib/postgresql/15 /mnt/data`;
попытайтесь запустить кластер - `sudo -u postgres pg_ctlcluster 15 main start`;
напишите получилось или нет и почему.
````
Error: /var/lib/postgresql/15/main is not accessible or does not exist
````
Не получилось, потому что содержимое каталога `/var/lib/postgres/15`, который является каталогом данных, перенесено

8. Задание: найти конфигурационный параметр в файлах раположенных в `/etc/postgresql/15/main` который надо поменять и поменять его

нужно поменять параметр `data_directory` в _postgresql.conf_

```
$ sudo nano /etc/postgresql/15/main/postgresql.conf
```

```
data_directory = '/mnt/data/15/main' 
```

9. попытайтесь запустить кластер - `sudo -u postgres pg_ctlcluster 15 main start`
напишите получилось или нет и почему

Запустить кластер получилось, теперь data_directory --  `/mnt/data/15/main`.

````
pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
15  main    5432 online postgres /mnt/data/15/main /var/log/postgresql/postgresql-15-main.log
````
Содержимое таблицы на месте:

````
sudo -su postgres
postgres@eek-ubuntu:/home/jenny-vm$ psql
postgres=# select * from test;
 c1 
----
 1
(1 row)
````

