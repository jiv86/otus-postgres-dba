# Занятие 10. Блокировки 

## Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

### Ставим на ВМ из предыдущего занятия новый инстанс Postgres 15
```
nenar@otus-dba-vaccum:~$ sudo pg_createcluster 15 locks --start
Creating new PostgreSQL cluster 15/locks ...
/usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/locks --auth-local peer --auth-host scram-sha-256 --no-instructions
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

fixing permissions on existing directory /var/lib/postgresql/15/locks ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Ver Cluster Port Status Owner    Data directory               Log file
15  locks   5434 online postgres /var/lib/postgresql/15/locks /var/log/postgresql/postgresql-15-locks.log
```
### Меняем настройки Postgres для логирования блокировок

```
nenar@otus-dba-vaccum:~$ sudo -u postgres psql -p 5434
psql (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
Type "help" for help.

postgres=# SHOW log_lock_waits;
 log_lock_waits
----------------
 off
(1 row)

postgres=# SHOW deadlock_timeout;
 deadlock_timeout
------------------
 1s
(1 row)

postgres=# ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM
postgres=# ALTER SYSTEM SET deadlock_timeout = 200;
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# SHOW deadlock_timeout;
 deadlock_timeout
------------------
 200ms
(1 row)

postgres=# SHOW log_lock_waits;
 log_lock_waits
----------------
 on
(1 row)

```
### Воспроизводим блокировку
В первой консоли сначала создаем нужные объекты БД:
```
postgres=# create database demo_locks;
CREATE DATABASE
postgres=# \c demo_locks;
You are now connected to database "demo_locks" as user "postgres".
CREATE TABLE tmp_locks(id int, col text);
demo_locks=#  INSERT INTO tmp_locks(id, col) values (1, 'collumn1');
INSERT 0 1
demo_locks=#  INSERT INTO tmp_locks(id, col) values (2, 'collumn2');
INSERT 0 1
demo_locks=#  INSERT INTO tmp_locks(id, col) values (3, 'collumn3');
INSERT 0 1
demo_locks=# SELECT * FROM tmp_locks;
 id |   col
----+----------
  1 | collumn1
  2 | collumn2
  3 | collumn3
(3 rows)
```

Далее начинам транзакцию с обновлением

```
demo_locks=# begin;
BEGIN
demo_locks=*# update tmp_locks SET col = 'new row' where id = 1;
UPDATE 1
demo_locks=*#
```

Во второй
```
demo_locks=# begin;
BEGIN
demo_locks=*#  update tmp_locks SET col = 'new row in second session' where id = 1;
```
Выполнение транзакции во второй сессии повисает. Делаю COMMIT в первой консоли и во второй.
Проверяем лог `/var/log/postgresql/postgresql-15-locks.log`
```
nenar@otus-dba-vaccum:~$ sudo tail -n 13 /var/log/postgresql/postgresql-15-locks.log
2024-10-16 14:40:01.855 UTC [4493] postgres@demo_locks STATEMENT:  begin
        update tmp_locks SET col = 'new_row' where id=1;
2024-10-16 14:43:40.288 UTC [4056] LOG:  checkpoint starting: time
2024-10-16 14:43:40.501 UTC [4056] LOG:  checkpoint complete: wrote 3 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.203 s, sync=0.003 s, total=0.214 s; sync files=3, longest=0.002 s, average=0.001 s; distance=0 kB, estimate=3830 kB
2024-10-16 14:44:03.845 UTC [4793] postgres@demo_locks LOG:  process 4793 still waiting for ShareLock on transaction 740 after 200.080 ms
2024-10-16 14:44:03.845 UTC [4793] postgres@demo_locks DETAIL:  Process holding the lock: 4493. Wait queue: 4793.
2024-10-16 14:44:03.845 UTC [4793] postgres@demo_locks CONTEXT:  while updating tuple (0,1) in relation "tmp_locks"
2024-10-16 14:44:03.845 UTC [4793] postgres@demo_locks STATEMENT:  update tmp_locks SET col = 'new row in second session' where id = 1;
2024-10-16 14:45:59.973 UTC [4793] postgres@demo_locks LOG:  process 4793 acquired ShareLock on transaction 740 after 116327.651 ms
2024-10-16 14:45:59.973 UTC [4793] postgres@demo_locks CONTEXT:  while updating tuple (0,1) in relation "tmp_locks"
2024-10-16 14:45:59.973 UTC [4793] postgres@demo_locks STATEMENT:  update tmp_locks SET col = 'new row in second session' where id = 1;
2024-10-16 14:48:40.584 UTC [4056] LOG:  checkpoint starting: time
2024-10-16 14:48:40.703 UTC [4056] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.103 s, sync=0.003 s, total=0.120 s; sync files=2, longest=0.002 s, average=0.002 s; distance=0 kB, estimate=3447 kB
```
В логе видим сообщение об ожидании блокировки Sharelock на транзакции с id 740 и то что требуемая блокировка все же была получена через 116 сеунд
`process 4793 acquired ShareLock on transaction 740 after 116327.651 ms`

## Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.






## Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?





## Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
