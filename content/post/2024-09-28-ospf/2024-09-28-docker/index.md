---
title:  "Памятка по docker CLI"
date: 2024-09-28 16:34:00 +0300
categories:
  - DevOps
tags:
  - docker
---

## Базовые команды

### Запуск контейнера в фоне

    docker run -d nginx

### Запуск контейнера с shell

    docker run -it ubuntu bash

### Запуск контейнера, удаляющегося после остановки

    docker run --rm ubuntu bash

### Экспорт порта контейнера

    docker run -p 80:80 -d nginx

### Задать имя контейнера

    docker run --name frontend nginx

### Запустить остановленный контейнер

    docker start frontend

### Остановить контейнер

    docker stop frontend


# Управление контейнерами

### Показать работающие контейнеры

    docker ps

### Показать работающие и остановленные контейнеры

    docker ps -a

### Показать метаданные контейнера

    docker inspect c31337

### Показать доступные локально образы

    docker images

### Удалить все остановленные контейнеры

    docker rm $(docker ps --filter status=expired -q)

### Показать все контейнеры со специальной меткой

    docker ps --filter label=traefik.backend

### Показать конкретные метаданные контейнера

    docker inspect -f '{{ .NetworkSettings.IPAddress }}' c31337

### Запустить ничего не делающий контейнер

    docker run -d busybox /bin/sh -c "while true; do sleep 2; done"


# Сборка образа

### На основе Dockerfile в текущей директории

    docker build --tag my-image .

### «Жёсткая» пересборка

    docker build --no-cache my-image .

### Преобразовать контейнер в образ

    docker commit c31337 my-image

### Удалить все неиспользуемые образы

    docker rmi $(docker images -q -f "dangling=true")


# Отладка

### Зайти в работающий контейнер

    docker exec -ti c31337 bash

### Просмотр логов запущенного контейнера

    docker logs -f c31337

### Показать экспортированные порты

    docker port c31337


# Диски

### Создать локальный диск

    docker volume create --name my-volume

### Монтирование диска при старте контейнера

    docker run -v my-volume:/data nginx

### Удалить диск

    docker volume rm my-volume

### Показать все созданные диски

    docker volume ls


# Сети

### Создать локальную сеть

    docker network create my-net

### Подключить контейнер к сети при старте

    docker run -d --net my-net nginx

### Подключить работающий контейнер к сети

    docker network connect my-net c31337

### Отключить работающий контейнер от сети

    docker network disconnect my-net c31337


# Управление Docker Machine

### Запустить Docker Machine

    docker-machine start machine_name

### Остановить Docker Machine

    docker-machine stop machine_name

### Настроить Docker на работу с удалённой Docker Machine

    eval "$(docker-machine env machine_name)"


# Работа с Dockerfile

Сборка образа из файла `Dockerfile`, учитывая, что мы находимся
в папке, где лежит этот файл:

    docker build -t my_docker .

Через ключ `-t` назначаем имя нашему образу.

Точка в конце означает, что `Dockerfile` лежит в текущей директории.

### Просмотреть существующие на хосте образы Docker

    docker images

### Просмотреть существующие на хосте контейнеры Docker

    docker ps -a

`-a` — по умолчанию команда показывает только включённые контейнеры,
вместе с ключом выводит все контейнеры.


# Удаление и экспорт

### Удалить контейнер

    docker rm

### Удалить образ

    docker rmi

С ключом `--force` — принудительное удаление.

### Удалить все существующие контейнеры

    docker rm $(docker ps -a -q)

### Экспортировать Docker-образ

    sudo docker save -o linux-nginx.img linux-nginx

### Импортировать Docker-образ

    sudo docker load -i linux-nginx.img


# Запуск контейнера с Bash

Запустить контейнер и открыть в нём `bash`:

    docker run -it -d --name my_container 397bd34237 /bin/bash

Параметры:

- `run` — команда запуска контейнера.
- `-it` — перейти в контейнер и запустить внутри контейнера команду.
- `-d` — запустить контейнер в фоне и вывести его ID.
- `--name` — присвоить имя контейнеру.
- `397bd34237` — ID/имя образа.
- `/bin/bash` — выполняемая команда в контейнере.

### Запустить контейнер, который уже запускался

    docker start ID-контейнера


# Bash-скрипт для запуска Docker-образа

    #!/bin/bash

    CONFIG=/home/isavel/ELK+KAFKA/for_docker_image/config/
    PATTERN=/home/isavel/ELK+KAFKA/for_docker_image/pattern/
    LOG=/home/isavel/ELK+KAFKA/for_docker_image/logs

    docker run \
      --restart=always -it \
      -p 443:443 -p 80:80 -p 5601:5601 \
      -e "TZ=Europe/Moscow" \
      -v $CONFIG:/etc/logstash/conf.d/ \
      -v $LOG:/var/log/my_log/ \
      -v $PATTERN:/opt/logstash/patterns/ \
      --name sberteh \
      5e89f9aa5754

Описание:

- `docker run` — команда запуска.
- `--restart=always` — в случае падения перезапускать контейнер.
- `-p 443:443 -p 80:80 -p 5601:5601` — проброс портов из Docker на localhost.
- Формат порта: `port_localhost:port_docker_image`.
- `-e "TZ=Europe/Moscow"` — указать timezone контейнера.
- `-v $CONFIG:/etc/logstash/conf.d/` — проброс директории с локальной машины.
- `-v $LOG:/var/log/my_log/` — проброс директории логов.
- `-v $PATTERN:/opt/logstash/patterns/` — проброс директории patterns.
- `--name sberteh` — название контейнера.
- `5e89f9aa5754` — ID запускаемого образа.


# Копирование файлов внутрь контейнера

    docker cp some_files.conf docker_container:/home/docker/


# Выполнить Bash внутри уже запущенного контейнера

    docker exec -it name_of_container /bin/bash


# Экспорт и импорт образа

Выгрузить образ в файл:

    docker save -o=file.tar CONTAINER

Импортировать его:

    docker load --input=file.tar


# Загрузка Docker-образа в локальный репозиторий

Присвоить тег:

    docker tag [name_image] [repo_name]:[port]/[name_image]

Отправить образ:

    docker push [repo_name]:[port]/[name_image]


# Установка Docker

## Linux

    curl -sSL https://get.docker.com/ | sh

## Mac

Скачать DMG:

    https://download.docker.com/mac/stable/Docker.dmg

## Windows

Использовать MSI-инсталлятор:

    https://download.docker.com/win/stable/InstallDocker.msi


# Реестры и репозитории Docker

## Вход в реестр

    docker login

Вход в конкретный реестр:

    docker login localhost:8080

## Выход из реестра

    docker logout

Выход из конкретного реестра:

    docker logout localhost:8080


# Поиск образа

    docker search nginx

Поиск с фильтрами:

    docker search nginx --filter stars=3 --no-trunc busybox


# Pull — загрузка образа из реестра

    docker pull nginx

Примеры:

    docker pull eon01/nginx

    docker pull localhost:5000/myadmin/nginx


# Push — загрузка образа в реестр

    docker push eon01/nginx

Пример с локальным реестром:

    docker push localhost:5000/myadmin/nginx


# Первые действия с контейнерами

## Создание контейнера

    docker create -t -i eon01/infinite --name infinite

## Первый запуск контейнера

    docker run -it --name infinite -d eon01/infinite


# Переименование контейнера

    docker rename infinite infinity


# Удаление контейнера

    docker rm infinite


# Обновление контейнера

    docker update --cpu-shares 512 -m 300M infinite


# Запуск и остановка контейнеров

## Запуск остановленного контейнера

    docker start nginx

## Остановка

    docker stop nginx

## Перезагрузка

    docker restart nginx

## Пауза

Приостановка всех процессов контейнера:

    docker pause nginx

## Снятие паузы

    docker unpause nginx

## Блокировка до остановки контейнера

    docker wait nginx

## Отправка SIGKILL

    docker kill nginx

## Отправка другого сигнала

Например, `HUP`:

    docker kill -s HUP nginx


# Подключение к существующему контейнеру

    docker attach nginx


# Получение информации о контейнерах

## Работающие контейнеры

    docker ps

## Все контейнеры

    docker ps -a
