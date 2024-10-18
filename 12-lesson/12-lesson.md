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
Выдало ошибку про несуществующую схему
