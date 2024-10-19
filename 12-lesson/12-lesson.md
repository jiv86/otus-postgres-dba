# Занятие 12. Резервное копирование и восстановление
## Подготовка окружения
Используем ВМ из YC из [Занятия 8](8-lesson/8-lesson.md)
## Создадим для этого занятия новый инстанс Postgres 15
```
nenar@otus-dba-vaccum:~$ sudo pg_createcluster 15 backup --start
Creating new PostgreSQL cluster 15/backup ...
/usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/backup --auth-local peer --auth-host scram-sha-256 --no-instructions
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

fixing permissions on existing directory /var/lib/postgresql/15/backup ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Ver Cluster Port Status Owner    Data directory                Log file
15  backup  5435 online postgres /var/lib/postgresql/15/backup /var/log/postgresql/postgresql-15-backup.log
```
### Создаем БД и схему в ней
```
nenar@otus-dba-vaccum:~$ sudo -u postgres psql -p 5435
psql (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
Type "help" for help.
postgres=# create database backup_restore;
CREATE DATABASE
backup_restore=# create schema exercise12;
CREATE SCHEMA
backup_restore=# create table exercise12.homework(id serial, text text);
CREATE TABLE
backup_restore=# insert into exercise12.homework(text) select md5(random()::text) from generate_series
(1,100);
INSERT 0 100

```

### Под линукс пользователем Postgres создадим каталог для бэкапов

```
nenar@otus-dba-vaccum:~$ sudo -su postgres
cd ~
mkdir pg_backup_dir

```
### Создаем логический бэкап с использованием утилиты COPY
```
backup_restore=# \copy exercise12.homework to '/var/lib/postgresql/pg_backup_dir/homework_table.sql'
COPY 100
```
### Восстановим в другую вновь созданную таблицу homework_2 данные из бэкапа
```
backup_restore=# create table exercise12.homework_2(id serial, text text);
CREATE TABLE
backup_restore=# \copy exercise12.homework_2 from '/var/lib/postgresql/pg_backup_dir/homework_table.sq
l';
COPY 100
```

### Используя утилиту pg_dump создадим бэкпап в формате custom со сжатием

```
nenar@otus-dba-vaccum:~$ sudo -u postgres pg_dump --file /var/lib/postgresql/pg_backup_dir/backup_restore.sql --host localhost --port 5435 --format=c -C --verbose backup_restore
Password:
pg_dump: last built-in OID is 16383
pg_dump: reading extensions
pg_dump: identifying extension members
pg_dump: reading schemas
pg_dump: reading user-defined tables
pg_dump: reading user-defined functions
pg_dump: reading user-defined types
pg_dump: reading procedural languages
pg_dump: reading user-defined aggregate functions
pg_dump: reading user-defined operators
pg_dump: reading user-defined access methods
pg_dump: reading user-defined operator classes
pg_dump: reading user-defined operator families
pg_dump: reading user-defined text search parsers
pg_dump: reading user-defined text search templates
pg_dump: reading user-defined text search dictionaries
pg_dump: reading user-defined text search configurations
pg_dump: reading user-defined foreign-data wrappers
pg_dump: reading user-defined foreign servers
pg_dump: reading default privileges
pg_dump: reading user-defined collations
pg_dump: reading user-defined conversions
pg_dump: reading type casts
pg_dump: reading transforms
pg_dump: reading table inheritance information
pg_dump: reading event triggers
pg_dump: finding extension tables
pg_dump: finding inheritance relationships
pg_dump: reading column info for interesting tables
pg_dump: finding table default expressions
pg_dump: flagging inherited columns in subtables
pg_dump: reading partitioning data
pg_dump: reading indexes
pg_dump: flagging indexes in partitioned tables
pg_dump: reading extended statistics
pg_dump: reading constraints
pg_dump: reading triggers
pg_dump: reading rewrite rules
pg_dump: reading policies
pg_dump: reading row-level security policies
pg_dump: reading publications
pg_dump: reading publication membership of tables
pg_dump: reading publication membership of schemas
pg_dump: reading subscriptions
pg_dump: reading large objects
pg_dump: reading dependency data
pg_dump: saving encoding = UTF8
pg_dump: saving standard_conforming_strings = on
pg_dump: saving search_path =
pg_dump: saving database definition
pg_dump: dumping contents of table "exercise12.homework"
pg_dump: dumping contents of table "exercise12.homework_2"
```
### Используя утилиту restore восстановим в новую БД "test" только вторую таблицу
```
postgres=# create database test;
CREATE DATABASE

postgres@otus-dba-vaccum:~$ pg_restore -U postgres --data-only --dbname test --table=homework_2  --host localhost --port 5435 --schema=exercise12 /var/lib/postgresql/pg_backup_dir/backup_restore.sql --verbose
pg_restore: connecting to database for restore
Password:
pg_restore: processing data for table "exercise12.homework_2"
pg_restore: while PROCESSING TOC:
pg_restore: from TOC entry 3398; 0 16407 TABLE DATA homework_2 postgres
pg_restore: error: could not execute query: ERROR:  schema "exercise12" does not exist
Command was: COPY exercise12.homework_2 (id, text) FROM stdin;
pg_restore: warning: errors ignored on restore: 1

```
Выдало ошибку про несуществующую схему. Хм...
Пробуем ресторить в два этапа, с созданием схемы и заливкой оставшихся данных
```
sudo -u postgres pg_restore -d otus_test_restore2 -p 5435 --schema-only -j 2 /var/lib/postgresql/pg_backup_dir/backup_restore.sql --verbose
sudo -u postgres pg_restore -d otus_test_restore2 -p 5435  --data-only --schema exercise12 --table homework_2 -j 2 /var/lib/postgresql/pg_backup_dir/backup_restore.sql --verbose
```
Подробный вывод
```
nenar@otus-dba-vaccum:~$ sudo -u postgres pg_restore -d otus_test_restore2 -p 5435 --schema-only -j 2 /var/lib/postgresql/pg_backup_dir/backup_restore.sql --verbose
pg_restore: connecting to database for restore
pg_restore: processing item 3401 ENCODING ENCODING
pg_restore: processing item 3402 STDSTRINGS STDSTRINGS
pg_restore: processing item 3403 SEARCHPATH SEARCHPATH
pg_restore: processing item 3404 DATABASE backup_restore
pg_restore: processing item 6 SCHEMA exercise12
pg_restore: creating SCHEMA "exercise12"
pg_restore: processing item 216 TABLE homework
pg_restore: creating TABLE "exercise12.homework"
pg_restore: processing item 218 TABLE homework_2
pg_restore: creating TABLE "exercise12.homework_2"
pg_restore: processing item 217 SEQUENCE homework_2_id_seq
pg_restore: creating SEQUENCE "exercise12.homework_2_id_seq"
pg_restore: processing item 3405 SEQUENCE OWNED BY homework_2_id_seq
pg_restore: creating SEQUENCE OWNED BY "exercise12.homework_2_id_seq"
pg_restore: processing item 215 SEQUENCE homework_id_seq
pg_restore: creating SEQUENCE "exercise12.homework_id_seq"
pg_restore: processing item 3406 SEQUENCE OWNED BY homework_id_seq
pg_restore: creating SEQUENCE OWNED BY "exercise12.homework_id_seq"
pg_restore: processing item 3251 DEFAULT homework id
pg_restore: creating DEFAULT "exercise12.homework id"
pg_restore: processing item 3252 DEFAULT homework_2 id
pg_restore: creating DEFAULT "exercise12.homework_2 id"
pg_restore: entering main parallel loop
pg_restore: skipping item 3396 TABLE DATA homework
pg_restore: skipping item 3398 TABLE DATA homework_2
pg_restore: skipping item 3407 SEQUENCE SET homework_2_id_seq
pg_restore: skipping item 3408 SEQUENCE SET homework_id_seq
pg_restore: finished main parallel loop

nenar@otus-dba-vaccum:~$ sudo -u postgres pg_restore -d otus_test_restore2 -p 5435  --data-only --schema exercise12 --ta
ble homework_2 -j 2 /var/lib/postgresql/pg_backup_dir/backup_restore.sql --verbose
could not change directory to "/home/nenar": Permission denied
pg_restore: connecting to database for restore
pg_restore: processing item 3401 ENCODING ENCODING
pg_restore: processing item 3402 STDSTRINGS STDSTRINGS
pg_restore: processing item 3403 SEARCHPATH SEARCHPATH
pg_restore: processing item 3404 DATABASE backup_restore
pg_restore: processing item 6 SCHEMA exercise12
pg_restore: processing item 216 TABLE homework
pg_restore: processing item 218 TABLE homework_2
pg_restore: processing item 217 SEQUENCE homework_2_id_seq
pg_restore: processing item 3405 SEQUENCE OWNED BY homework_2_id_seq
pg_restore: processing item 215 SEQUENCE homework_id_seq
pg_restore: processing item 3406 SEQUENCE OWNED BY homework_id_seq
pg_restore: processing item 3251 DEFAULT homework id
pg_restore: processing item 3252 DEFAULT homework_2 id
pg_restore: entering main parallel loop
pg_restore: skipping item 3396 TABLE DATA homework
pg_restore: launching item 3398 TABLE DATA homework_2
pg_restore: skipping item 3407 SEQUENCE SET homework_2_id_seq
pg_restore: skipping item 3408 SEQUENCE SET homework_id_seq
pg_restore: processing data for table "exercise12.homework_2"
pg_restore: finished item 3398 TABLE DATA homework_2
pg_restore: finished main parallel loop
```
Похоже на то что такой вариант сработал и не выдал ворнингов..

Проверяем: данные есть во второй таблице и нет в первой как и было в задании.

```
postgres=# \c otus_test_restore2;
You are now connected to database "otus_test_restore2" as user "postgres".
otus_test_restore2=# select * from exercise12.homework_2 limit 10;
 id |               text
----+----------------------------------
  1 | 0eb99a76a6be5f3cb09a4373e968580f
  2 | 58f80634b0a51550237d1c59504099c6
  3 | 915084aaa68f057f8a3e48d26e692401
  4 | a1736fd481557b841dc1a6ea61a602ae
  5 | 70e322a3eb5e0c95bce65a444aa73a6c
  6 | 5f89ffb69caf855a0a5c7b528908fb7c
  7 | b7ddff2a56095ab534b6decd6be2ca0e
  8 | 74f883c29b29e4aa4cf8caca6fac6f47
  9 | ca8ae2ac2effe3bde1d99bfb8d7501ca
 10 | c64f4e5c1b4f135171727f8118dae0d8
(10 rows)

otus_test_restore2=# select * from exercise12.homework limit 10;
 id | text
----+------
(0 rows)

```
