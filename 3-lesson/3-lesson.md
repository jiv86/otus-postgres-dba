# Установка PostgreSQL

## Подготовка стенда

Использум ВМ из [предыдущего домашнего задания](../2-lesson/2-lesson.md#подготовка-виртуальной-машины-и-установка-субд).

Устанавливаем DockerEngine по [инструкции по установке из apt-репозитория](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)
``` shell
# Add Docker's official GPG key:
dimon@pg-stand-01:~$sudo apt-get update
dimon@pg-stand-01:~$sudo apt-get install ca-certificates curl
dimon@pg-stand-01:~$sudo install -m 0755 -d /etc/apt/keyrings
dimon@pg-stand-01:~$sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
dimon@pg-stand-01:~$sudo chmod a+r /etc/apt/keyrings/docker.asc
```
