# Занятие 6. Физический уровень PostgreSQL 
## Подготавливаем окружение
### Создание ВМ
В этот раз создадим ВМ в yandex-cloud чтобы иметь возможность подключатся к ней из разных мест.
Параметры ВМ: Ubuntu 22.04 LTS, Платформа Intel Ice Lake, Гарантированная доля vCPU 100%, ​vCPU 2, RAM 2 ГБ, HDD 20 ГБ
Подключаемся к ранее созданной ВМ при помощи ключа
``` bash
C:\Users\nenarokov.d>ssh 84.252.135.24 -l nenar -i C:\Users\nenarokov.d\ssh-key-yandex\dimon_rsa
nenar@otus-db-pg-vm-01:~$ 
```
