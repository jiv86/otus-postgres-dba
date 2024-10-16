# Занятие 9. Журналы
## Подготовка ВМ
Используем ВМ в YC из [предыдущего занятия ](../8-lesson/8-lesson.md) 
## Изменение периодичности чекпойнтов

``` bash
nenar@otus-dba-vaccum:~$ sudo nano /etc/postgresql/15/main/postgresql.conf

#------------------------------------------------------------------------------
# WRITE-AHEAD LOG
#------------------------------------------------------------------------------
# - Settings -
# - Checkpoints -

checkpoint_timeout = 30s                # range 30s-1d
```
Перезапустим кластер 15/main для применения изменений `sudo pg_ctlcluster 15 main restart`
Инициализируем pgbench для базы postgres

  ``` bash
  nenar@otus-dba-vaccum:~$ pgbench -h localhost -p 5432 -U postgres -i postgres
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
  done in 0.47 s (drop tables 0.02 s, create tables 0.05 s, client-side generate 0.15 s, vacuum 0.11 s, primary keys 0.14 s).
  ```
  Теперь можно запускать бэнчмарки.
  
  ## Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.
  ### Проверяем размер директории pg_wal до, получаем 49 МБ
  ```
  nenar@otus-dba-vaccum:~$ sudo du -h /var/lib/postgresql/15/main/pg_wal
  4.0K    /var/lib/postgresql/15/main/pg_wal/archive_status
  49M     /var/lib/postgresql/15/main/pg_wal
  ```
  ### Воспользуемся также статистикой из системного представления `pg_stat_bgwriter`
  ```
  nenar@otus-dba-vaccum:~$ sudo -u postgres psql
  Password for user postgres:
  psql (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
  Type "help" for help.

  postgres=#  select * from pg_stat_bgwriter \gx
  -[ RECORD 1 ]---------+------------------------------
  checkpoints_timed     | 61
  checkpoints_req       | 1
  checkpoint_write_time | 27699
  checkpoint_sync_time  | 22
  buffers_checkpoint    | 1719
  buffers_clean         | 0
  maxwritten_clean      | 0
  buffers_backend       | 1691
  buffers_backend_fsync | 0
  buffers_alloc         | 1873
  stats_reset           | 2024-10-16 09:21:04.037855+00
  ```
### Посмотрим теущее положение в буфере WAL
```
select pg_current_wal_insert_lsn(), pg_current_wal_lsn(), pg_current_wal_flush_lsn();
postgres=# select pg_current_wal_insert_lsn(), pg_current_wal_lsn(), pg_current_wal_flush_lsn();
pg_current_wal_insert_lsn | pg_current_wal_lsn | pg_current_wal_flush_lsn
---------------------------+--------------------+--------------------------
0/70E1220                 | 0/70E1220          | 0/70E1220
(1 row)
```
Значения pg_current_wal_insert_lsn() - положение вставки в WAL, pg_current_wal_lsn()- положение последней записи, отданной для записи ОС, pg_current_wal_flush_lsn() -положение последней записи сохраненной на диск совпадают. В базе нет данных несохраненных в WAL, из-за отсутсвия активности нагрузки.

### Запускаем тест pgbench на 600 секунд, 10 клиентов, вывод статистики каждые 40 сек
```
nenar@otus-dba-vaccum:~$ pgbench -h localhost -p 5432 -U postgres -c 8 -P 40 -T 600 postgres
Password:
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 40.0 s, 364.2 tps, lat 21.875 ms stddev 20.152, 0 failed
progress: 80.0 s, 432.8 tps, lat 18.486 ms stddev 17.521, 0 failed
progress: 120.0 s, 386.4 tps, lat 20.718 ms stddev 21.458, 0 failed
progress: 160.0 s, 422.2 tps, lat 18.944 ms stddev 18.629, 0 failed
progress: 200.0 s, 411.5 tps, lat 19.411 ms stddev 18.239, 0 failed
progress: 240.0 s, 382.5 tps, lat 20.926 ms stddev 19.892, 0 failed
progress: 280.0 s, 477.1 tps, lat 16.762 ms stddev 15.736, 0 failed
progress: 320.0 s, 400.5 tps, lat 19.967 ms stddev 18.570, 0 failed
progress: 360.0 s, 392.8 tps, lat 20.366 ms stddev 19.890, 0 failed
progress: 400.0 s, 464.5 tps, lat 17.222 ms stddev 15.854, 0 failed
progress: 440.0 s, 394.7 tps, lat 20.234 ms stddev 19.736, 0 failed
progress: 480.0 s, 415.6 tps, lat 19.270 ms stddev 18.700, 0 failed
progress: 520.0 s, 418.4 tps, lat 19.116 ms stddev 17.651, 0 failed
progress: 560.0 s, 372.3 tps, lat 21.468 ms stddev 20.084, 0 failed
progress: 600.0 s, 351.0 tps, lat 22.800 ms stddev 22.934, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 243469
number of failed transactions: 0 (0.000%)
latency average = 19.707 ms
latency stddev = 19.020 ms
initial connection time = 100.764 ms
tps = 405.828238 (without initial connection time)
```
### Проверяем размер директории pg_wal после, получаем 65 МБ

```
nenar@otus-dba-vaccum:~$ sudo du -h /var/lib/postgresql/15/main/pg_wal
4.0K    /var/lib/postgresql/15/main/pg_wal/archive_status
65M     /var/lib/postgresql/15/main/pg_wal
```
### Смотрим статистику bgwriter
```
nenar@otus-dba-vaccum:~$ sudo -u postgres psql
could not change directory to "/home/nenar": Permission denied
Password for user postgres:
psql (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
Type "help" for help.

postgres=# select * from pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 135
checkpoints_req       | 1
checkpoint_write_time | 593087
checkpoint_sync_time  | 440
buffers_checkpoint    | 42612
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 3414
buffers_backend_fsync | 0
buffers_alloc         | 3922
stats_reset           | 2024-10-16 09:21:04.037855+00

postgres=# select pg_current_wal_insert_lsn(), pg_current_wal_lsn(), pg_current_wal_flush_lsn();
 pg_current_wal_insert_lsn | pg_current_wal_lsn | pg_current_wal_flush_lsn
 pg_current_wal_insert_lsn | pg_current_wal_lsn | pg_current_wal_flush_lsn
---------------------------+--------------------+--------------------------
 0/205D3E70                | 0/205D3E70         | 0/205D3E70
(1 row)

postgres-#
```
Посмотрим разницу между lsn до `0/70E1220 ` и после теста `0/205D3E70`
``` sql
postgres=# select pg_wal_lsn_diff('0/205D3E70','0/70E1220');
 pg_wal_lsn_diff
-----------------
       424619088
(1 row)

postgres=# select pg_size_pretty(pg_wal_lsn_diff('0/205D3E70','0/70E1220'));
 pg_size_pretty
----------------
 405 MB
(1 row)
```



