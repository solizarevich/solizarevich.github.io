---
title:  "Памятка по docker CLI"
date: 2024-09-28 16:34:00 +0300
categories:
  - DevOps
tags:
  - docker
---

![](1.jpeg)

**добавить username в группу docker (без этого пользователю придется** работать с docker через sudo):

**$ sudo usermod -aG docker username**

**запуск образа ubuntu с последующим помещением в контейнер, вход в bash этого контейнера:**

**$ docker run -it ubuntu bash**

**root@f4099c025760:/#**

**-i — открывает интерактивную сессию**

**-t — осуществляет взаимодействие с терминалом контейнера**

**отобразить активные контейнеры:**

**$ docker ps**

**отобразить все контейнеры (в том числе, неактивные):**

**docker ps -a**

**запустить контейнер с именем container_name (столбец NAMES в выводе docker ps):**

**docker start container_name**

**задать имя хоста контейнера:**

**$ docker run -h blablabla -it** ubuntu bash

**получить подробную информацию о контейнере container_name:**

**$ docker inspect container_name**

**задать имя контейнера:**

**$ docker run --name MYN**AME -it ubuntu bash

**показать логи событий с контейнера:**

**$ docker logs container_name**

**полностью удалить контейнер, используя его container_name (но не скачанный образ):**

**$ docker rm container_name**

**полностью удалить контейнер, используя его ID:**

**$ docker rm -v container_ID**

**вывести ID всех контейнеров (флаг -q выводи ID):**

**$ docker ps -aq**

**полностью удалить все завершенные контейнеры (флаг -f задает условия поиска):**

**$ docker rm -v $(docker ps -aq -f status=exited)**

**запустить контейнер в фоновом режиме (демонизировать):**

**$ docker run -it -d ubuntu bash**

**выполнить проброс портов (-p HOST_PORT:CONTAINER_PORT):**

**$ docker run -it -d -p 8080:80 ubuntu bash**

**отобразить список скачанных образов:**

**$ docker images**

**сохранить состояние контейнера в образ (пользователь username, контейнер container_name):**

**$ docker commit username/container_name**

**удалить образ:**

**$ docker rmi im**age_ID

**войти в оболочку bash уже запущенного контейнером с идентификаторов container_id**

**$ docker exec -it container_id bash**

**создать связь между запускаемым контейнером и some_container:**

**$ docker run --link some_container:link_name -it ubuntu bash**

Чуть подробнее о создании связи: допустим, есть запущенный контейнер с именем MY_CONTAINER:

$ docker ps

CONTAINER ID IMAGE COMMAND CREATED STATUS **PORTS** NAMES

860c93385b82 ubuntu "bash" 4 seconds ago Up 2 seconds MY_CONTAINER

**создадим связь NEW_LINK с этим контейнером:**

**$ docker run --link MY_CONTAINER:NEW_LINK -i**t **ubuntu bash**

**root@10dfcd26190b:/# cat /etc/hosts | grep NE**W_**LINK**

**172.17.0.2 NEW_LINK 860c93385b82 MY_CONT**AI**NER**

При создании связи в новом контейнере образуется вышеуказанная запись в файле /etc/hosts
