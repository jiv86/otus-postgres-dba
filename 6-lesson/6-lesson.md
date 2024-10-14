# Занятие 6. Физический уровень PostgreSQL 

## Подготавливаем окружение
### Создание ВМ
В этот раз создадим ВМ в yandex-cloud чтобы иметь возможность подключатся к ней из разных мест.
Параметры ВМ: Ubuntu 22.04 LTS, Платформа Intel Ice Lake, Гарантированная доля vCPU 100%, ​vCPU 2, RAM 2 ГБ, HDD 20 ГБ

Подключаемся к ранее созданной ВМ при помощи ключа
``` bash
C:\Users\nenarokov.d>ssh 84.252.135.24 -l nenar -i C:\Users\nenarokov.d\ssh-key-yandex\dimon_rsa
nenar@otus-db-pg-vm-01:~$ 
```
### Ставим Postgres-15 и после проверяем что инстанс Postgres 15 main запущен 
``` bash
nenar@otus-db-pg-vm-01:~$ sudo apt-get update
nenar@otus-db-pg-vm-01:~$ sudo apt-get upgrade
nenar@otus-db-pg-vm-01:~$ sudo apt-get install postgresql-15 postgresql-contrib
nenar@otus-db-pg-vm-01:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
### Создаем тестовую базу, таблицу и наполняем ее данными
``` sql
nenar@otus-db-pg-vm-01:~$ sudo -u postgres psql
could not change directory to "/home/nenar": Permission denied
psql (15.8 (Ubuntu 15.8-1.pgdg22.04+1))
Type "help" for help.
postgres=# CREATE DATABASE otus_physical;
CREATE DATABASE
postgres=# \c otus_physical;
You are now connected to database "otus_physical" as user "postgres".
postgres=# CREATE DATABASE otus_physical;
CREATE DATABASE
postgres=# \c otus_physical;
You are now connected to database "otus_physical" as user "postgres".
otus_physical=# CREATE TABLE study (
    id text primary key,
    description text
);

CREATE TABLE search_result (
    id serial primary key,
    name text,
    study_id text references study(id)
);
CREATE TABLE
CREATE TABLE
otus_physical=# INSERT INTO
    study (id, description)
VALUES
    (
        'OBESITY',
        'Lorem ipsum dolor sit amet, consectetur adipiscing elit'
    ),
        (
        'DIABETES',
        'Lorem ipsum dolor sit amet, consectetur adipiscing elit'
    ),
        (
        'HYPERTENSION',
        'Lorem ipsum dolor sit amet, consectetur adipiscing elit'
    );
INSERT 0 3
otus_physical=# INSERT INTO
    search_result (name, study_id)
SELECT
    md5(random() :: text),
    'OBESITY'
FROM
    generate_series(1, 100) s(i);
INSERT 0 100
otus_physical=# INSERT INTO
    search_result (name, study_id)
SELECT
    md5(random() :: text),
    'HYPERTENSION'
FROM
    generate_series(1, 100) s(i);
INSERT 0 100
otus_physical=# INSERT INTO
    search_result (name, study_id)
SELECT
    md5(random() :: text),
    'DIABETES'
FROM
    generate_series(1, 100) s(i);
otus_physical=# select count(*) from search_result;
 count
-------
   300
(1 row)
```
### Добавляем в Яндекс Клауде к ВМ дополнительный диск размером 10 Гб
Смотрим сначала данные нашего инстанса
```
yc compute instance list
+----------------------+------------------+---------------+---------+---------------+-------------+
|          ID          |       NAME       |    ZONE ID    | STATUS  |  EXTERNAL IP  | INTERNAL IP |
+----------------------+------------------+---------------+---------+---------------+-------------+
| fv4gmfr6igfbnjaceb6n | otus-db-pg-vm-01 | ru-central1-d | RUNNING | 84.252.135.24 | 10.130.0.20 |
+----------------------+------------------+---------------+---------+---------------+-------------+

```
выводим список инстансов в клауде
Создаем диск в клауде
```
yc compute disk create --help
 yc compute disk create --name second-disk-otus-vm --size 10 --zone ru-central1-d --description "second disk for otus-db-pg-vm-01"
done (7s)
id: fv4r4c46t745570m6s9b
folder_id: b1gaqdqu489uuo27ke2k
created_at: "2024-10-14T08:44:51Z"
name: second-disk-otus-vm
description: second disk for otus-db-pg-vm-01
type_id: network-hdd
zone_id: ru-central1-d
size: "10737418240"
block_size: "4096"
status: READY
disk_placement_policy: {}
hardware_generation:
  legacy_features:
    pci_topology: PCI_TOPOLOGY_V1
```



Список дисков
```
 yc compute disk list
+----------------------+---------------------+-------------+---------------+--------+----------------------+-----------------+--------------------------------+
|          ID          |        NAME         |    SIZE     |     ZONE      | STATUS |     INSTANCE IDS     | PLACEMENT GROUP |          DESCRIPTION           |
+----------------------+---------------------+-------------+---------------+--------+----------------------+-----------------+--------------------------------+
| fv4r4c46t745570m6s9b | second-disk-otus-vm | 10737418240 | ru-central1-d | READY  |                      |                 | second disk for                |
|                      |                     |             |               |        |                      |                 | otus-db-pg-vm-01               |
| fv4tqvq23in83ba3had2 |                     | 21474836480 | ru-central1-d | READY  | fv4gmfr6igfbnjaceb6n |                 |                                |
+----------------------+---------------------+-------------+---------------+--------+----------------------+-----------------+--------------------------------+
```

Подключаем диск
``` bash
yc compute instance attach-disk --help
yc compute instance attach-disk otus-db-pg-vm-01 --disk-name second-disk-otus-vm --mode rw
done (11s)
id: fv4gmfr6igfbnjaceb6n
folder_id: b1gaqdqu489uuo27ke2k
created_at: "2024-08-24T18:24:29Z"
name: otus-db-pg-vm-01
zone_id: ru-central1-d
platform_id: standard-v3
resources:
  memory: "2147483648"
  cores: "2"
  core_fraction: "100"
status: RUNNING
metadata_options:
  gce_http_endpoint: ENABLED
  aws_v1_http_endpoint: ENABLED
  gce_http_token: ENABLED
  aws_v1_http_token: DISABLED
boot_disk:
  mode: READ_WRITE
  device_name: fv4tqvq23in83ba3had2
  auto_delete: true
  disk_id: fv4tqvq23in83ba3had2
secondary_disks:
  - mode: READ_WRITE
    device_name: fv4r4c46t745570m6s9b
    disk_id: fv4r4c46t745570m6s9b
network_interfaces:
  - index: "0"
    mac_address: d0:0d:10:b3:f6:69
    subnet_id: fl8ln9ss2mea88k1libd
    primary_v4_address:
      address: 10.130.0.20
      one_to_one_nat:
        address: 84.252.135.24
        ip_version: IPV4
serial_port_settings:
  ssh_authorization: INSTANCE_METADATA
gpu_settings: {}
fqdn: otus-db-pg-vm-01.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}
hardware_generation:
  legacy_features:
    pci_topology: PCI_TOPOLOGY_V1

```
Диск подключился успешно и получил идентификатор vdb, судя по выводу lsblk
```
nenar@otus-db-pg-vm-01:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0  63.3M  1 loop /snap/core20/1822
loop1    7:1    0  63.9M  1 loop /snap/core20/2318
loop2    7:2    0 111.9M  1 loop /snap/lxd/24322
loop3    7:3    0    87M  1 loop /snap/lxd/29351
loop4    7:4    0  49.8M  1 loop /snap/snapd/18357
loop5    7:5    0  38.8M  1 loop /snap/snapd/21759
vda    252:0    0    20G  0 disk
├─vda1 252:1    0     1M  0 part
└─vda2 252:2    0    20G  0 part /
vdb    252:16   0    10G  0 disk
```
**Ставим Gparted**
``` bash
sudo apt update
sudo apt install parted
```
**Ищем неразмеченное устройство в выводе Gparted**

``` bash
nenar@otus-db-pg-vm-01:~$ sudo parted -l
Error: /dev/vdb: unrecognised disk label
Model: Virtio Block Device (virtblk)
Disk /dev/vdb: 10.7GB
Sector size (logical/physical): 512B/4096B
Partition Table: unknown
Disk Flags:
```
**Устанавливаем тип таблицы разделов**
``` bash
nenar@otus-db-pg-vm-01:~$ sudo parted /dev/vdb mklabel gpt
Information: You may need to update /etc/fstab.
```

**Сооздаем новый раздел**

``` bash
nenar@otus-db-pg-vm-01:~$ sudo parted -a opt /dev/vdb mkpart primary ext4 0% 100%
nenar@otus-db-pg-vm-01:~$ sudo parted -a opt /dev/vdb mkpart primary ext4 0% 100%
Information: You may need to update /etc/fstab.
```
> Описание параметров
> -a opt - оптимальное выравнивание
>  mkpart primary ext4  - создать загрузочный раздел с использованием ФС ext4
> 0% 100% - означает что раздел от начала до конца  диска

**Создаем ФС на созданном разделе**
``` bash
nenar@otus-db-pg-vm-01:~$ sudo mkfs.ext4 -L datapartition /dev/vdb1
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 2620928 4k blocks and 655360 inodes
Filesystem UUID: 26c6e23f-c1bd-4947-97ae-1c68aabf8806
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done


nenar@otus-db-pg-vm-01:~$ sudo lsblk -fs
NAME  FSTYPE   FSVER LABEL         UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0 squashfs 4.0                                                            0   100% /snap/core20/1822
loop1 squashfs 4.0                                                            0   100% /snap/core20/2318
loop2 squashfs 4.0                                                            0   100% /snap/lxd/24322
loop3 squashfs 4.0                                                            0   100% /snap/lxd/29351
loop4 squashfs 4.0                                                            0   100% /snap/snapd/18357
loop5 squashfs 4.0                                                            0   100% /snap/snapd/21759
vda1
└─vda
vda2  ext4     1.0                 ed465c6e-049a-41c6-8e0b-c8da348a3577   14.3G    23% /
└─vda
vdb1  ext4     1.0   datapartition 26c6e23f-c1bd-4947-97ae-1c68aabf8806
└─vdb
```
Монтируем раздел в /mnt/data 
случай с временныйм монтированием `sudo mount -o defaults /dev/vdb1 /mnt/data`

```
sudo nano /etc/fstab
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/vda2 during curtin installation
/dev/disk/by-uuid/ed465c6e-049a-41c6-8e0b-c8da348a3577 / ext4 defaults 0 1
/dev/vdb1 /mnt/data ext4 defaults 0 2
```
> /mnt/data is the path where the disk is being mounted.
> 
> ext4 connotes that this is an Ext4 partition.
> 
> defaults means that this volume should be mounted with the default options, such as read-write support.
> 
> 0 2 signifies that the filesystem should be validated by the local machine in case of errors, but as a 2nd priority, after your root volume.

