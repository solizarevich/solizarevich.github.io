---
title: "Настройка скорости и дуплекса сетевой карты в Linux через ethtool"
date: 2024-10-02 10:15:00 +0300
categories:
  - Linux
tags:
  - linux
  - ethtool
  - network
---


[Команда ethtool в Linux]([https://dzen.ru/away?to=https%3A%2F%2Fwww.opennet.ru%2Fman.shtml%3Ftopic%3Dethtool%26category%3D8%26russian%3D0](https://www.opennet.ru/man.shtml?topic=ethtool&category=8&russian=0)) позволяет изменять параметры сетевого интерфейса, такие как скорость передачи данных, режим дуплекса и автоопределение.

Пример команды:

**sudo ethtool -s eth0 speed 1000 duplex full autoneg on**

Эта команда устанавливает скорость 1 Гбит/с, полный дуплекс и активирует автоопределение.

![](1.jpeg)
