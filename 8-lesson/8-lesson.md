# Занятие 8. MVCC, vacuum и autovacuum. 
## Подготовка ВМ
Создадим в Яндекс Клауде ВМ с требуемыми параметрами и именем `otus-dba-vacuum`
SSD 15Гб, 4Гб RAM, 2vCPU 100%
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
## Устанавливаем пароль пользователю postgres СУБД

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

## Запускаем тест на БД testdb_vacuum, имитация 8 клиентов, вывод показателей каждые 6 секунд, продолжительность бэнчмарка 60 секунд, запуск от имени юзера postgres


``` bash
nenar@otus-dba-vaccum:~$ pgbench -c8 -P 6 -T 60 -U postgres testdb_vacuum
Password:
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 573.0 tps, lat 13.789 ms stddev 12.026, 0 failed
progress: 12.0 s, 638.5 tps, lat 12.527 ms stddev 10.947, 0 failed
progress: 18.0 s, 383.2 tps, lat 20.835 ms stddev 20.208, 0 failed
progress: 24.0 s, 575.3 tps, lat 13.915 ms stddev 12.819, 0 failed
progress: 30.0 s, 581.0 tps, lat 13.739 ms stddev 14.828, 0 failed
progress: 36.0 s, 684.8 tps, lat 11.718 ms stddev 9.528, 0 failed
progress: 42.0 s, 735.7 tps, lat 10.878 ms stddev 17.540, 0 failed
progress: 48.0 s, 420.0 tps, lat 19.044 ms stddev 20.140, 0 failed
progress: 54.0 s, 513.5 tps, lat 15.464 ms stddev 13.855, 0 failed
progress: 60.0 s, 604.0 tps, lat 13.306 ms stddev 13.115, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 34262
number of failed transactions: 0 (0.000%)
latency average = 13.995 ms
latency stddev = 14.750 ms
initial connection time = 65.411 ms
tps = 571.459137 (without initial connection time)
```
## Применение настроек из материалов к занятию
* Предлагается установить следующие настройки относительно стандартных
* сравним:

| Название параметра             | Значение по умолчанию | Предлагаемое значение | 
| ------------------------------ | --------------------- | --------------------- |
| `max_connections`              | 100                   | 40                    |
| `shared_buffers`               | 128MB                 | 1GB                   |
| `effective_cache_size`         | 4GB                   | 3GB                   |
| `maintenance_work_mem`         | 64MB                  | 512MB                 |
| `checkpoint_completion_target` | 0.9                   | 0.9                   |
| `wal_buffers`                  | 4MB                   | 16MB                  |
| `default_statistics_target`    | 100                   | 500                   |
| `random_page_cost`             | 4                     | 4                     |
| `effective_io_concurrency`     | 1                     | 2                     |
| `work_mem`                     | 4096kB                | 6553kB                |
| `min_wal_size`                 | 80MB                  | 4GB                   |
| `max_wal_size`                 | 1GB                   | 16GB                  |

* Применили рекомендованные параметры
  Убеждаемся что они действительно применились
  
  ```
  nenar@otus-dba-vaccum:~$ sudo -u postgres psql
  postgres=# select name, setting, unit from pg_settings where name in ('max_connections', 'shared_buffers', 'effective_cache_size', 'maintenance_work_mem', 'checkpoint_completion_target', 
  'wal_buffers', 'default_statistics_target', 'random_page_cost', 'effective_io_concurrency', 'work_mem', 'min_wal_size', 'max_wal_size');
             name             | setting | unit
  ------------------------------+---------+------
   checkpoint_completion_target | 0.9     |
   default_statistics_target    | 500     |
   effective_cache_size         | 393216  | 8kB
   effective_io_concurrency     | 2       |
   maintenance_work_mem         | 524288  | kB
   max_connections              | 40      |
   max_wal_size                 | 16384   | MB
   min_wal_size                 | 4096    | MB
   random_page_cost             | 4       |
   shared_buffers               | 131072  | 8kB
   wal_buffers                  | 2048    | 8kB
   work_mem                     | 6553    | kB
   (12 rows)
  ```
* ПРИМЕНИЛИСЬ УСПЕШНО.

  ## Запускем тест повторно после изменения настроек кластера

 ``` bash
nenar@otus-dba-vaccum:~$ pgbench -c8 -P 6 -T 60 -U postgres testdb_vacuum
Password:
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 500.2 tps, lat 15.794 ms stddev 16.429, 0 failed
progress: 12.0 s, 487.3 tps, lat 16.395 ms stddev 15.266, 0 failed
progress: 18.0 s, 667.2 tps, lat 12.004 ms stddev 11.976, 0 failed
progress: 24.0 s, 472.0 tps, lat 16.926 ms stddev 14.482, 0 failed
progress: 30.0 s, 578.0 tps, lat 13.847 ms stddev 11.090, 0 failed
progress: 36.0 s, 445.8 tps, lat 17.949 ms stddev 15.927, 0 failed
progress: 42.0 s, 541.2 tps, lat 14.763 ms stddev 15.596, 0 failed
progress: 48.0 s, 688.7 tps, lat 11.625 ms stddev 11.202, 0 failed
progress: 54.0 s, 653.0 tps, lat 12.237 ms stddev 10.426, 0 failed
progress: 60.0 s, 748.2 tps, lat 10.703 ms stddev 8.981, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 34697
number of failed transactions: 0 (0.000%)
latency average = 13.820 ms
latency stddev = 13.211 ms
initial connection time = 69.056 ms
tps = 578.687952 (without initial connection time)

 ```
* Результат теста: среднее значение TPS незначительно выросло, менее чем на 1%.
  
> ПРИМЕЧАНИЕ: Возможно тест должен был показать более яркое изменение производительности. Но результат такой.
