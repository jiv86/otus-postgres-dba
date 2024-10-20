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
#### Проводим тест pgbench 14 на базе `test`, продолжительность 600 секунд, вывод результатов каждые 60 сек

```
postgres@postgres15-benchmark:~$ pgbench -c14 -P 60 -T 600 -U postgres test
pgbench (15.8 (Ubuntu 15.8-1.pgdg24.04+1))
starting vacuum...end.
progress: 60.0 s, 2103.3 tps, lat 6.638 ms stddev 5.014, 0 failed
progress: 120.0 s, 2102.4 tps, lat 6.644 ms stddev 4.994, 0 failed
progress: 180.0 s, 2091.2 tps, lat 6.680 ms stddev 5.067, 0 failed
progress: 240.0 s, 2092.0 tps, lat 6.678 ms stddev 5.214, 0 failed
progress: 300.0 s, 2092.9 tps, lat 6.675 ms stddev 5.281, 0 failed
progress: 360.0 s, 1972.6 tps, lat 7.082 ms stddev 5.595, 0 failed
progress: 420.0 s, 2099.5 tps, lat 6.654 ms stddev 5.253, 0 failed
progress: 480.0 s, 2011.7 tps, lat 6.945 ms stddev 5.449, 0 failed
progress: 540.0 s, 2092.7 tps, lat 6.675 ms stddev 5.110, 0 failed
progress: 600.0 s, 2094.3 tps, lat 6.670 ms stddev 5.100, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 14
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 1245175
number of failed transactions: 0 (0.000%)
latency average = 6.731 ms
latency stddev = 5.210 ms
initial connection time = 27.452 ms
tps = 2075.357013 (without initial connection time)
```
**В среднем получается 2075 TPS**

### Настраиваем кластер на максимальную производительность 

#### Рассчет параметров

Для расчета воспользуемся сервисом https://pgconfigurator.cybertec.at/
Исходим из следующих параметров.


####
