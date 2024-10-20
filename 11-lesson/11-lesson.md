# Занятие 11. Настройка PostgreSQL

## Настройка окружения

### Создаем ВМ на VMWare Workstation
ВМ будет со следующими параметрами Ubuntu Server 24.04 LTS, 14 vCPU, 72 Гб RAM, 70 Гб NVMe

### Установка Postgres 15

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
