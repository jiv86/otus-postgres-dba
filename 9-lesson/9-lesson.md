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
* Перезапустим кластер 15/main для применения изменений `sudo pg_ctlcluster 15 main restart`
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



