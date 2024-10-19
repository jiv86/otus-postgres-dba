# Занятие 7. Логический уровень PostgreSQL 
## Готовим окружение

1. создайте новый кластер PostgresSQL 14
### создаем ВМ `otus-logical` в Яндекс Клауде 2vCPU 100%, 4Гб RAM, 20 Гб HDD, Ubuntu 22.04

### Устанавливаем на эту ВМ Postgres 14, при этом создается и запускатся кластер(инстанс) main
```
тenar@otus-logical:~$ sudo apt-get update
nenar@otus-logical:~$ sudo apt-get upgrade
nenar@otus-logical:~$ sudo apt install dirmngr ca-certificates software-properties-common apt-transport-https lsb-release curl -y
nenar@otus-logical:~$ curl -fSsL https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor | sudo tee /usr/share/keyrings/postgresql.gpg > /dev/null
nenar@otus-logical:~$ echo deb [arch=amd64,arm64,ppc64el signed-by=/usr/share/keyrings/postgresql.gpg] http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main | sudo tee /etc/apt/sources.list.d/postgresql.list
nenar@otus-logical:~$ sudo apt update
nenar@otus-logical:~$ sudo apt install postgresql-client-14 postgresql-14
nenar@otus-logical:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```

2. зайдите в созданный кластер под пользователем postgres
   
``` bash
nenar@otus-logical:~$ sudo -u postgres psql
psql (14.13 (Ubuntu 14.13-1.pgdg22.04+1))
Type "help" for help.

postgres=#

```
3. создайте новую базу данных testdb
4. зайдите в созданную базу данных под пользователем postgres
5. создайте новую схему testnm
6. создайте новую таблицу t1 с одной колонкой c1 типа integer
7. вставьте строку со значением c1=1
8. создайте новую роль readonly
9. дайте новой роли право на подключение к базе данных testdb
10. дайте новой роли право на использование схемы testnm
11. дайте новой роли право на select для всех таблиц схемы testnm
12. создайте пользователя testread с паролем test123
13. дайте роль readonly пользователю testread
    
**выполнение вышеобозначенных пунктов в консоли psql**
``` bash

postgres=# create database testdb;
CREATE DATABASE
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# create schema testnm;
CREATE SCHEMA
testdb=# create role readonly;
CREATE ROLE
testdb=# create table t1 (c1 integer);
CREATE TABLE
testdb=# insert into t1 (c1) values (1);
INSERT 0 1
testdb=# grant connect on database testdb to readonly;
GRANT
testdb=#  grant usage on schema testnm to readonly;
GRANT
testdb=# grant select on all tables in schema testnm to readonly;
GRANT
testdb=# create user testread with password 'test123';
CREATE ROLE
testdb=# grant readonly to testread;
GRANT ROLE

```

## Эксперменты с правами на выборку данных

14. зайдите под пользователем testread в базу данных testdb
Пробуем подключится
```
FATAL:  Peer authentication failed for user "testread"
```
Поменял в pg_hba.conf метод аунтификации с peer на scram-sha-256
Перезапустил кластер для применения
```
# "local" is for Unix domain socket connections only
local   all             all                                     scram-sha-256
nenar@logical:~$  sudo pg_ctlcluster 14 main restart
```
Снова пробуем подключится
``` bash
nenar@otus-logical:~$ psql -U testread -d testdb
Password for user testread:
psql (14.13 (Ubuntu 14.13-1.pgdg22.04+1))
Type "help" for help.

testdb=>
```
Успешно.
15. сделайте select * from t1;
```
testdb=> select * from t1;
ERROR:  relation "t1" does not exist
LINE 1: select * from t1;
```
16. получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)

  **НЕТ, НЕ ПОЛУЧИЛОСЬ.**

17. напишите что именно произошло в тексте домашнего задания
    **Выборку пытались делать без указания схемы (при выборке использовался параметр `search_path`в котором имеется только схема `public`), а права давали на схему testnm**
19. у вас есть идеи почему? ведь права то дали?
20. посмотрите на список таблиц
    **ПОСМОТРЕЛ**
       ```
       testdb=> \d
       List of relations
       Schema | Name | Type  |  Owner
      --------+------+-------+----------
       public | t1   | table | postgres
      (1 row)
       ```
22. подсказка в шпаргалке под пунктом 20
23. а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)

     **Нет прав на чтение таблицы t1 у пользователя с ролью readonly т.к. у данной роли права есть только в схеме testnm а запрос на создание таблицы мы выполнили в 6 пункте
    без явного указания имени схемы, соответсвенно в схему public**

    
24. вернитесь в базу данных testdb под пользователем postgres
25. удалите таблицу t1
26. создайте ее заново но уже с явным указанием имени схемы testnm
27. вставьте строку со значением c1=1
```
nenar@otus-logical:~$ sudo -u postgres psql
psql (14.13 (Ubuntu 14.13-1.pgdg22.04+1))
Type "help" for help.

postgres=#
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# drop table t1;
DROP TABLE
testdb=# create table testnm.t1 (c1 integer);
CREATE TABLE
testdb=# insert into testnm.t1 (c1) values (1);
INSERT 0 1
```

26. зайдите под пользователем testread в базу данных testdb
27. сделайте select * from testnm.t1;

```
select * from testnm.t1;
ERROR:  permission denied for table t1
```

28. получилось?
    ***НЕТ, не получилось**
 
29. есть идеи почему? если нет - смотрите шпаргалку

    **Не хватает прав так как мы делали grant select on all tables in schema testnm TO readonly, а после мы пересоздали таблицу t1**
31. как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку

    **Один из вариантов после пересоздания объектов переназначать права через grant select on all tables in schema testnm to readonly;**, но это неподходит
    под определение "чтобы не повторялось в будущем", права на вновь созданные объекты в таком случае придется снова восстанавливать
33. сделайте select * from testnm.t1;
34. получилось?

    **НЕТ**
36. есть идеи почему? если нет - смотрите шпаргалку
    Выполняем по рекомендации
    ```
    testdb=# ALTER default privileges in SCHEMA testnm grant SELECT on TABLES to readonly;
    ALTER DEFAULT PRIVILEGES
    ```
37. сделайте select * from testnm.t1;
45. получилось?

    **ДА**
47. ура!
48. теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
49. а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?
50. есть идеи как убрать эти права? если нет - смотрите шпаргалку
51. если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды
52. теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);
53. расскажите что получилось и почему
