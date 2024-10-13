# Установка PostgreSQL

## Подготовка стенда

Использум ВМ из [предыдущего домашнего задания](../2-lesson/2-lesson.md#подготовка-виртуальной-машины-и-установка-субд).

Устанавливаем DockerEngine по [инструкции по установке из apt-репозитория](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)

**Add Docker's official GPG key:**
``` shell

dimon@pg-stand-01:~$sudo apt-get update
dimon@pg-stand-01:~$sudo apt-get install ca-certificates curl
dimon@pg-stand-01:~$sudo install -m 0755 -d /etc/apt/keyrings
dimon@pg-stand-01:~$sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
dimon@pg-stand-01:~$sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Добавляем apt-репозитоий с пакетами докер
``` bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

```

Ставим последнюю версию пакетов докер

``` bash
dimon@pg-stand-01:~$sudo apt-get update
dimon@pg-stand-01:~$sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Проверим корректность установки Docker на примере образа Hello-World

``` bash
dimon@pg-stand-01:~$sudo docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
c1ec31eb5944: Pull complete
Digest: sha256:d211f485f2dd1dee407a80973c8f129f00d54604d2c90732e8e320e5038a0348
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash
```
