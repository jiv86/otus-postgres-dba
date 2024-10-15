# Занятие 8. MVCC, vacuum и autovacuum. 
## Подготовка ВМ
Создадим в Яндекс Клауде ВМ с требуемыми параметрами и именем `otus-dba-vacuum`
## Установка Postgres 15

``` bash
nenar@otus-db-vacuum:~$ sudo apt-get update
nenar@otus-db-vacuum:~$ sudo apt-get upgrade
nenar@otus-db-vacuum:~$ sudo apt install dirmngr ca-certificates software-properties-common apt-transport-https lsb-release curl -y
nenar@otus-db-vacuum:~$ curl -fSsL https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor | sudo tee /usr/share/keyrings/postgresql.gpg > /dev/null
nenar@otus-db-vacuum:~$ echo deb [arch=amd64,arm64,ppc64el signed-by=/usr/share/keyrings/postgresql.gpg] http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main | sudo tee /etc/apt/sources.list.d/postgresql.list
nenar@otus-db-vacuum:~$ sudo apt update
nenar@otus-db-vacuum:~$ sudo apt install postgresql-client-15 postgresql-15
nenar@otus-db-vacuum:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
## Создаем БД 

``` bash
nenar@otus-db-vacuum:~$ sudo -u postgres psql
psql (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
Type "help" for help.
postgres=# create database testdb_vacuum;
CREATE DATABASE
```
## Устанавливаем пароь пользователю postgres СУБД

```
nenar@otus-dba-vaccum:~$ sudo -u postgres psql
postgres=# ALTER USER postgres WITH PASSWORD 'postgres789';
ALTER ROLE
postgres=#
```
## Инициализируем pgbench на БД testdb_vacuum

``` bash
nenar@otus-dba-vaccum:~$ sudo -iu postgres
postgres@otus-dba-vaccum:~$ pgbench -h localhost -p 5432 -U postgres -i testdb_vacuum
Password:
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.03 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 1.19 s (drop tables 0.00 s, create tables 0.02 s, client-side generate 0.87 s, vacuum 0.03 s, primary keys 0.28 s).
postgres@otus-dba-vaccum:~$

```
