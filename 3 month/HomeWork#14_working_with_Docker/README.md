# Домашнее задание "Docker"

## Задание

- Установите Docker на хост машину https://docs.docker.com/engine/install/ubuntu/
- Установите Docker Compose - как плагин, или как отдельное приложение
- Создайте свой кастомный образ nginx на базе alpine. После запуска nginx должен отдавать кастомную страницу (достаточно изменить дефолтную страницу nginx)
- Определите разницу между контейнером и образом
- Ответьте на вопрос: Можно ли в контейнере собрать ядро?
---

## Выполнение задания

Для выполнения задания будет использоваться ОС Ubuntu 24.04.4 LTS

### 1. Установим Docker на хостовую машину.
```bash
root@client:/home/user# curl -fsSL https://get.docker.com -o get-docker.sh
root@client:/home/user# sudo sh get-docker.sh

```

Проверяем установку
```bash
root@client:/home/user# docker version
Client: Docker Engine - Community
 Version:           29.6.0
 API version:       1.55
 Go version:        go1.26.4
 Git commit:        fb59821
 Built:             Thu Jun 18 19:57:31 2026
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          29.6.0
  API version:      1.55 (minimum version 1.40)
  Go version:       go1.26.4
  Git commit:       70eaf5e
  Built:            Thu Jun 18 19:57:31 2026
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          v2.2.5
  GitCommit:        e53c7c1516c3b2bff98eb76f1f4117477e6f4e66
 runc:
  Version:          1.3.6
  GitCommit:        v1.3.6-0-g491b69ba
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
root@client:/home/user# docker compose version
Docker Compose version v5.1.4
root@client:/home/user# docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
4f55086f7dd0: Pull complete 
d5e71e642bf5: Download complete 
Digest: sha256:96498ffd522e70807ab6384a5c0485a79b9c7c08ca79ba08623edcad1054e62d
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

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```
для работы без root можно добавить пользователя в группу docker:
```bash
root@client:/home/user# usermod -aG docker user
```
### 3. Создайте свой кастомный образ nginx на базе alpine.

Создаем директорию для проекта
```bash
root@client:/home/user# mkdir ~/my-nginx && cd ~/my-nginx
```

Создаем файл Dockerfile
```bash
root@client:~/my-nginx# nano Dockerfile


FROM nginx:alpine
# Удаляем стандартный index.html
RUN rm -rf /usr/share/nginx/html/*
# Создаём свою страницу
RUN echo "<h1>Привет от моего Nginx!</h1>" > /usr/share/nginx/html/index.html
# Открываем порт
EXPOSE 80
```

Собераем образ
```bash
root@client:~/my-nginx# docker build -t my-nginx:latest .
[+] Building 260.1s (7/7) FINISHED                                                                                                                                                  docker:default
 => [internal] load build definition from Dockerfile                                                                                                                                          0.0s
 => => transferring dockerfile: 320B                                                                                                                                                          0.0s
 => [internal] load metadata for docker.io/library/nginx:alpine                                                                                                                               9.0s
 => [internal] load .dockerignore                                                                                                                                                             0.0s
 => => transferring context: 2B                                                                                                                                                               0.0s
 => [1/3] FROM docker.io/library/nginx:alpine@sha256:20316569d8f81a160065d7d2a5eeffc7ca97d79022462ee255fd23fa103a6b5c                                                                       248.1s
 => => resolve docker.io/library/nginx:alpine@sha256:20316569d8f81a160065d7d2a5eeffc7ca97d79022462ee255fd23fa103a6b5c                                                                         0.0s
 => => sha256:747ffc4966ddc47bf2726c4920fae8ff32701317d10fdb5b5a6b072be77cd224 20.30MB / 20.30MB                                                                                            247.3s
 => => sha256:e0b71e755d6728c93d01b0598ddef188cb372ac2bbc7b6fba195346ddd716544 1.40kB / 1.40kB                                                                                                1.4s
 => => sha256:d4676175f3221c45de528abd3d6017838074e02b67c310a827918d8a87be4e8b 1.21kB / 1.21kB                                                                                                1.6s
 => => sha256:2ff91eb02cdcdfe760a1a09792015aa54f4330d7e1e946d019c988494fccfa41 403B / 403B                                                                                                    1.6s
 => => sha256:f982d9d7587e2065b05f1ddf9fb49d287c6ffc877c610446eb02db3a4c91019d 953B / 953B                                                                                                    0.2s
 => => sha256:9d80b52aca3c36b340585ac903125d1ea10a27752e4eded2a3eda4865897bc96 4.36MB / 4.36MB                                                                                               47.2s
 => => sha256:696fe85c76cd21407aede3c8c09ab3102f8736ecfe8b05d20a422bb6569473fa 627B / 627B                                                                                                    0.2s
 => => sha256:6a0ac1617861a677b045b7ff88545213ec31c0ff08763195a70a4a5adda577bb 3.86MB / 3.86MB                                                                                               46.6s
 => => extracting sha256:6a0ac1617861a677b045b7ff88545213ec31c0ff08763195a70a4a5adda577bb                                                                                                     0.1s
 => => extracting sha256:9d80b52aca3c36b340585ac903125d1ea10a27752e4eded2a3eda4865897bc96                                                                                                     0.2s
 => => extracting sha256:696fe85c76cd21407aede3c8c09ab3102f8736ecfe8b05d20a422bb6569473fa                                                                                                     0.0s
 => => extracting sha256:f982d9d7587e2065b05f1ddf9fb49d287c6ffc877c610446eb02db3a4c91019d                                                                                                     0.0s
 => => extracting sha256:2ff91eb02cdcdfe760a1a09792015aa54f4330d7e1e946d019c988494fccfa41                                                                                                     0.0s
 => => extracting sha256:d4676175f3221c45de528abd3d6017838074e02b67c310a827918d8a87be4e8b                                                                                                     0.0s
 => => extracting sha256:e0b71e755d6728c93d01b0598ddef188cb372ac2bbc7b6fba195346ddd716544                                                                                                     0.0s
 => => extracting sha256:747ffc4966ddc47bf2726c4920fae8ff32701317d10fdb5b5a6b072be77cd224                                                                                                     0.6s
 => [2/3] RUN rm -rf /usr/share/nginx/html/*                                                                                                                                                  1.4s
 => [3/3] RUN echo "<h1>Привет от моего Nginx!</h1>" > /usr/share/nginx/html/index.html                                                                                                       0.5s
 => exporting to image                                                                                                                                                                        0.8s
 => => exporting layers                                                                                                                                                                       0.3s
 => => exporting manifest sha256:fd5d7982f5c293200e577f9a02bfc438c0bf1a162ed503d1d4e9ac20e25ba742                                                                                             0.0s
 => => exporting config sha256:5eeda67d7a2449965b283036ef5c08f179c543484b562ed9dd520705cded0141                                                                                               0.0s
 => => exporting attestation manifest sha256:b950970c1b279f5597d7b9ae2048a828ea57ca56170819dbb811b84d5efced96                                                                                 0.1s
 => => exporting manifest list sha256:b119d0ccf8a934f6d03a7a594e7883bcb41eaaa4ff52f1ecb50b0126be941508                                                                                        0.1s
 => => naming to docker.io/library/my-nginx:latest                                                                                                                                            0.0s
 => => unpacking to docker.io/library/my-nginx:latest                                                                                                                                         0.2s

```

Проверяем, что образ создался
```bash
root@client:~/my-nginx# docker images | grep my-nginx
my-nginx:latest      b119d0ccf8a9        101MB         28.5MB  
```
Запустим контейнер для тестирования
```bash
root@client:~/my-nginx# docker run -d -p 8080:80 --name test-nginx my-nginx:latest
fff91054ae5c1ca8d519c7dcd5d0d41495967e2c6315d9ee9156aaeec941991e
```

Проверряем работу
```bash
root@client:~/my-nginx# curl http://localhost:8080
<h1>Привет от моего Nginx!</h1>
```

Остановим и удалим тестовый контейнер
```bash
root@client:~/my-nginx# docker stop test-nginx
test-nginx
root@client:~/my-nginx# docker rm test-nginx
test-nginx
```

Образ – это неизменяемый шаблон, который содержит всё необходимое для запуска приложения: операционную систему, библиотеки, исполняемые файлы, конфигурации и переменные окружения. Образ хранится в реестре (например, Docker Hub),его нельзя изменить, можно только пересобрать заново. Образ состоит из слоёв, каждый слой – это результат выполнения одной инструкции в Dockerfile.

Контейнер – это запущенный экземпляр образа. Он создаётся на основе образа и добавляет к нему слой записи (изменяемый). Контейнер изолирован от хоста и других контейнеров: у него своё сетевое пространство, файловая система, дерево процессов и ресурсы. Контейнер можно запускать, останавливать, удалять, вносить внутрь изменения (создавать файлы, устанавливать пакеты и т. д.), но при удалении контейнера все изменения, сделанные внутри него, теряются.

В контейнере нельзя собрать ядро в том смысле, чтобы оно работало как ядро для этого контейнера. Контейнеры используют ядро хоста, и это их фундаментальное отличие от виртуальных машин.