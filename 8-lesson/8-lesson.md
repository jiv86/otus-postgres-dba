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
