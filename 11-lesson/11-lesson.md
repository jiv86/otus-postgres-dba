# Занятие 11. Настройка PostgreSQL

## Настройка окружения

### Создаем ВМ на VMWare Workstation
ВМ будет со следующими параметрами Ubuntu Server 24.04 LTS, 14 vCPU, 72 Гб RAM, 70 Гб NVMe

### Установим Postgres 15

```
dimon@postgres15-benchmark:~$ sudo apt-get update
dimon@postgres15-benchmark:~$ sudo apt-get upgrade
dimon@postgres15-benchmark:~$ sudo apt install dirmngr ca-certificates software-properties-common apt-transport-https lsb-release curl -y
dimon@postgres15-benchmark:~$ curl -fSsL https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor | sudo tee /usr/share/keyrings/postgresql.gpg > /dev/null
dimon@postgres15-benchmark:~$ echo deb [arch=amd64,arm64,ppc64el signed-by=/usr/share/keyrings/postgresql.gpg] http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main | sudo tee /etc/apt/sources.list.d/postgresql.list
dimon@postgres15-benchmark:~$ sudo apt update
dimon@postgres15-benchmark:~$ sudo apt install postgresql-client-15 postgresql-15
dimon@postgres15-benchmark:~$ pg_lsclusters

Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

### Проводим тест pgbench со стандартными настройками

#### Инициализируем pgbench на базе `test`

```
dimon@postgres15-benchmark:~$ sudo -iu postgres
postgres@postgres15-benchmark:~$ pgbench -h localhost -p 5432 -U postgres -i test
Password:
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.01 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.14 s (drop tables 0.00 s, create tables 0.00 s, client-side generate 0.09 s, vacuum 0.02 s, primary keys 0.02 s).
```
#### Проводим тест pgbench 14 клиентов, 1 поток, на базе `test`, продолжительность 300 секунд, вывод результатов каждые 60 сек

```
dimon@postgres15-benchmark:~$ sudo -iu postgres
postgres@postgres15-benchmark:~$ pgbench -c14 -P 60 -T 300 -U postgres test
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 60.0 s, 2033.5 tps, lat 6.866 ms stddev 5.189, 0 failed
progress: 120.0 s, 2077.8 tps, lat 6.724 ms stddev 5.126, 0 failed
progress: 180.0 s, 2079.8 tps, lat 6.717 ms stddev 5.186, 0 failed
progress: 240.0 s, 2072.9 tps, lat 6.740 ms stddev 5.297, 0 failed
progress: 300.0 s, 2069.0 tps, lat 6.752 ms stddev 5.234, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 14
number of threads: 1
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 620002
number of failed transactions: 0 (0.000%)
latency average = 6.759 ms
latency stddev = 5.207 ms
initial connection time = 30.231 ms
tps = 2066.816667 (without initial connection time)
postgres@postgres15-benchmark:~$
```
**Результат: 2067 TPS**

**Делаем тест 14 клиентов 14 потоков**

```
postgres@postgres15-benchmark:~$ pgbench -c14 -j 14 -P 60 -T 300 -U postgres test
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 60.0 s, 2128.2 tps, lat 6.576 ms stddev 5.310, 0 failed
progress: 120.0 s, 2163.5 tps, lat 6.471 ms stddev 5.247, 0 failed
progress: 180.0 s, 2144.4 tps, lat 6.528 ms stddev 5.270, 0 failed
progress: 240.0 s, 2155.2 tps, lat 6.496 ms stddev 5.261, 0 failed
progress: 300.0 s, 2167.2 tps, lat 6.459 ms stddev 5.280, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 14
number of threads: 14
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 645531
number of failed transactions: 0 (0.000%)
latency average = 6.506 ms
latency stddev = 5.274 ms
initial connection time = 8.826 ms
tps = 2151.751100 (without initial connection time)
```
**Результат: 2151 TPS**
### Настраиваем кластер на максимальную производительность 

#### Рассчет параметров

Для расчета воспользуемся сервисом https://pgconfigurator.cybertec.at/
Исходим из следующих параметров.

| Название параметра                                                       | Значение                       | 
| ------------------------------------------------------------------------ | ------------------------------ |
| Select your version of PostgreSQL:                                       | 15                             | 
| GB of RAM in your server:                                                | 64                             | 
| Number of CPUs (= cores):                                                | 14                             | 
| Disk Type:                                                               | SSD                            | 
| Number of disks:                                                         | 10                             | 
| How would you describe your workload?                                    | OLTP                           | 
| How many concurrent open connections do you expect?                      | 500                            |
| How many replicas do you need?                                           | 1                              | 
| Which backup method are you planning to use?                             | pg_basebackup: Binary backups  | 
| Do you want to activate wal recycling?                                   | YES                            | 
| Can you lose single transactions in case of a crash?                     | YES                            | 
| Are you willing to try out experimental features for better performance? | YES                            | 

Сервис предлагает следующие настройки для данной конфигурации

| Название параметра                                                       | Значение по умолчанию          | Значение для применения
| ------------------------------------------------------------------------ | ------------------------------ | ------------------------
| `max_connections`                                                        | 100                            | 500
| `superuser_reserved_connections`                                         | 3                              | 3
| `shared_buffers`                                                         | '128MB'                        | '16384MB'
| `work_mem`                                                               | '4MB'                          | '32MB
| `maintenance_work_mem`                                                   | '64MB'                         | '520MB'
| `huge_pages`                                                             | try                            | try
| `effective_cache_size`                                                   | '4GB'                          | '45GB'
| `random_page_cost`                                                       | 4.0                            | 1.25
| `shared_preload_libraries`                                               |                                | 'pg_stat_statements' 
| `track_io_timing`                                                        | off                            | on
| `track_functions`                                                        | none                           | pl
| `wal_level`                                                              | replica                        | replica
| `max_wal_senders`                                                        | 10                             | 10
| `synchronous_commit`                                                     | on                             | off
| `checkpoint_timeout`                                                     | '5min'                         | '15min'
| `checkpoint_completion_target`                                           | 0.9                            | 0.9
| `max_wal_size`                                                           | 1GB                            | '10GB'
| `min_wal_size`                                                           | 80MB                           | '5120MB'
| `archive_mode`                                                           | off                            | on
| `archive_command`                                                        |                                | '/bin/true'
| `wal_compression`                                                        | off                            | on
| `wal_buffers`                                                            | -1                             | -1
| `wal_keep_size`                                                          | 0                              | 22080
| `bgwriter_delay`                                                         | 200ms                          | 200ms
| `bgwriter_lru_maxpages`                                                  | 100                            | 100
| `bgwriter_lru_multiplier`                                                | 2.0                            | 2.0
| `bgwriter_flush_after`                                                   | 512kB                          | 0
| `max_worker_processes`                                                   | 8                              | 14
| `max_parallel_workers_per_gather`                                        | 2                              | 7
| `max_parallel_maintenance_workers`                                       | 2                              | 7
| `max_parallel_workers`                                                   | 8                              | 14
| `parallel_leader_participation`                                          | on                             | on
| `enable_partitionwise_join`                                              | off                            | on
| `enable_partitionwise_aggregate`                                         | off                            | on
| `jit`                                                                    | on                             | on
| `max_slot_wal_keep_size`                                                 | -1                             | 1000
| `track_wal_io_timing`                                                    | off                            | on
| `maintenance_io_concurrency`                                             | 10                             | 500
| `wal_recycle`                                                            | on                             | on


<details><summary><b><i>Вывод из сервиса pgconfigurator.cybertec.at</b></i></summary>

```
# DISCLAIMER - Software and the resulting config files are provided AS IS - IN NO EVENT SHALL
# BE THE CREATOR LIABLE TO ANY PARTY FOR DIRECT, INDIRECT, SPECIAL, INCIDENTAL, OR CONSEQUENTIAL
# DAMAGES, INCLUDING LOST PROFITS, ARISING OUT OF THE USE OF THIS SOFTWARE AND ITS DOCUMENTATION.

# Connectivity
max_connections = 500
superuser_reserved_connections = 3

# Memory Settings
shared_buffers = '16384 MB'
work_mem = '32 MB'
maintenance_work_mem = '520 MB'
huge_pages = try # NB! requires also activation of huge pages via kernel params, see here for more: https://www.postgresql.org/docs/current/static/kernel-resources.html#LINUX-HUGE-PAGES
effective_cache_size = '45 GB'
effective_io_concurrency = 500 # concurrent IO only really activated if OS supports posix_fadvise function
random_page_cost = 1.25 # speed of random disk access relative to sequential access (1.0)

# Monitoring
shared_preload_libraries = 'pg_stat_statements' # per statement resource usage stats
track_io_timing=on # measure exact block IO times
track_functions=pl # track execution times of pl-language procedures if any

# Replication
wal_level = replica # consider using at least 'replica'
max_wal_senders = 10
synchronous_commit = off

# Checkpointing:
checkpoint_timeout = '15 min'
checkpoint_completion_target = 0.9
max_wal_size = '10240 MB'
min_wal_size = '5120 MB'

# WAL archiving
archive_mode = on # having it on enables activating P.I.T.R. at a later time without restart›
archive_command = '/bin/true' # not doing anything yet with WAL-s


# WAL writing
wal_compression = on
wal_buffers = -1 # auto-tuned by Postgres till maximum of segment size (16MB by default)
wal_keep_size = '22080 MB'


# Background writer
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 100
bgwriter_lru_multiplier = 2.0
bgwriter_flush_after = 0

# Parallel queries:
max_worker_processes = 14
max_parallel_workers_per_gather = 7
max_parallel_maintenance_workers = 7
max_parallel_workers = 14
parallel_leader_participation = on

# Advanced features
enable_partitionwise_join = on
enable_partitionwise_aggregate = on
jit = on
max_slot_wal_keep_size = '1000 MB'
track_wal_io_timing = on
maintenance_io_concurrency = 500
wal_recycle = on


# General notes:
# Note that not all settings are automatically tuned.
# Consider contacting experts at
# https://www.cybertec-postgresql.com
# for more professional expertise.
```
</details>

### Применяем указанные параметры в файле конфигурации и перезапускаем инсанс Postgres

### Повторяем тест pgbench, получаем следующие результаты
* Тест 14 клиентов, 1 воркер
```
dimon@postgres15-benchmark:~$ sudo -iu postgres
postgres@postgres15-benchmark:~$ pgbench -c14 -P 60 -T 300 -U postgres test
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 60.0 s, 2993.8 tps, lat 4.660 ms stddev 3.777, 0 failed
progress: 120.0 s, 2976.3 tps, lat 4.690 ms stddev 3.792, 0 failed
progress: 180.0 s, 2913.4 tps, lat 4.792 ms stddev 3.895, 0 failed
progress: 240.0 s, 2943.1 tps, lat 4.743 ms stddev 3.834, 0 failed
progress: 300.0 s, 2945.5 tps, lat 4.739 ms stddev 3.834, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 14
number of threads: 1
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 886347
number of failed transactions: 0 (0.000%)
latency average = 4.724 ms
latency stddev = 3.827 ms
initial connection time = 30.035 ms
tps = 2954.697154 (without initial connection time)
```
**Результат: 2955 TPS**


* тест 14 клиентов 14 воркеров
```
postgres@postgres15-benchmark:~$ pgbench -c14 -j 14 -P 60 -T 300 -U postgres test
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 60.0 s, 4845.3 tps, lat 2.889 ms stddev 2.594, 0 failed
progress: 120.0 s, 4988.0 tps, lat 2.806 ms stddev 2.475, 0 failed
progress: 180.0 s, 4974.3 tps, lat 2.814 ms stddev 2.465, 0 failed
progress: 240.0 s, 4910.5 tps, lat 2.851 ms stddev 2.478, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 14
number of threads: 14
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 1485683
number of failed transactions: 0 (0.000%)
latency average = 2.827 ms
latency stddev = 2.490 ms
initial connection time = 9.229 ms
tps = 4952.302716 (without initial connection time)
```
**Результат: 4952 TPS**

### ВЫВОД: 
> В результате применения настроек прирост показателей транзакций в секунду в тесте `pgbench` составил 42% в однопоточном тесте и 130% в многопоточном режиме (14 потоков). 
> Наиболее значимый эффект дает включение асинхронного коммита `synchronous_commit = off`

