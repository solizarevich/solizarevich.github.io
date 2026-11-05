---
title: "systemd timers в Linux"
date: 2026-11-05 09:30:00 +0300
categories:
  - Linux
tags:
  - linux
  - systemd timers
  - видео
---

В предыдущих материалах мы разобрали два классических способа запуска задач по расписанию:

```text
cron → регулярные задачи
at   → одноразовые задачи
```

Но в современных Linux-системах на базе `systemd` есть ещё один механизм:

```text
systemd timers
```

Systemd timer позволяет запускать сервисы по времени или через определённые интервалы и управлять ими привычными инструментами `systemd`.

В этой статье разберём:

- что такое systemd timer;
- как связаны `.timer` и `.service`;
- как посмотреть существующие timers;
- как создать собственный timer;
- как работает `OnCalendar`;
- что такое `OnBootSec` и `OnUnitActiveSec`;
- зачем нужен `Persistent=true`;
- как проверять расписание;
- как смотреть логи выполнения;
- как запускать одноразовые timers через `systemd-run`;
- чем systemd timers отличаются от cron.

---

# Что такое systemd timer

Systemd timer — это специальный unit-файл systemd, который запускает другой unit по расписанию.

Чаще всего timer запускает соответствующий `.service`.

Например:

```text
backup.timer
      ↓
backup.service
      ↓
backup.sh
```

То есть timer отвечает за вопрос:

> Когда запускать?

А service отвечает:

> Что запускать?

Это важное отличие от `cron`, где расписание и команда обычно записываются одной строкой.

---

# Как связаны .timer и .service

По умолчанию systemd связывает timer и service с одинаковым именем.

Например:

```text
backup.timer
```

будет запускать:

```text
backup.service
```

Поэтому обычно создаётся пара файлов:

```text
/etc/systemd/system/backup.service
/etc/systemd/system/backup.timer
```

Схема получается такой:

```text
backup.timer
      ↓
расписание
      ↓
backup.service
      ↓
команда или скрипт
```

При необходимости timer может запускать unit с другим именем, но для начала удобнее придерживаться одинаковых названий.

---

# Как посмотреть существующие timers

Systemd уже использует timers для различных системных задач.

Посмотреть активные таймеры можно командой:

```bash
systemctl list-timers
```

Пример вывода:

```text
NEXT                        LEFT       LAST                        PASSED      UNIT
Tue 2026-11-03 23:30:00    20min      Tue 2026-11-03 22:30:00    40min ago   example.timer
```

Здесь можно увидеть:

```text
NEXT    → когда будет следующий запуск
LEFT    → сколько осталось
LAST    → когда timer запускался последний раз
PASSED  → сколько прошло времени
UNIT    → имя timer unit
```

Чтобы показать в том числе неактивные timer units:

```bash
systemctl list-timers --all
```

Это одна из самых полезных команд при работе с systemd timers.

---

# Создаём первый systemd service

Сначала создадим сервис.

Например:

```bash
sudo nano /etc/systemd/system/mytask.service
```

Содержимое:

```ini
[Unit]
Description=My scheduled task

[Service]
Type=oneshot
ExecStart=/usr/local/bin/mytask.sh
```

Здесь:

```text
Type=oneshot
```

означает, что сервис выполняет одну задачу и завершается.

А:

```text
ExecStart=
```

указывает команду или скрипт.

---

# Создаём тестовый скрипт

Например:

```bash
sudo nano /usr/local/bin/mytask.sh
```

Добавим:

```bash
#!/bin/bash

echo "Timer выполнен: $(date)" >> /tmp/mytimer.log
```

Дадим права:

```bash
sudo chmod +x /usr/local/bin/mytask.sh
```

Проверим вручную:

```bash
sudo /usr/local/bin/mytask.sh
```

Посмотрим результат:

```bash
cat /tmp/mytimer.log
```

Если появилась строка с текущей датой, скрипт работает.

---

# Проверяем service вручную

До создания timer полезно убедиться, что сам сервис запускается.

Обновим конфигурацию systemd:

```bash
sudo systemctl daemon-reload
```

Запустим сервис:

```bash
sudo systemctl start mytask.service
```

Проверим:

```bash
systemctl status mytask.service
```

И посмотрим наш лог:

```bash
cat /tmp/mytimer.log
```

Если service работает вручную, можно переходить к timer.

---

# Создаём .timer

Теперь создадим:

```bash
sudo nano /etc/systemd/system/mytask.timer
```

Например:

```ini
[Unit]
Description=Run mytask every day

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

Теперь у нас есть два файла:

```text
mytask.service
mytask.timer
```

Timer будет запускать service.

---

# OnCalendar

`OnCalendar` задаёт календарное расписание.

Например:

```ini
OnCalendar=daily
```

означает запуск каждый день.

Можно использовать:

```ini
OnCalendar=hourly
```

или:

```ini
OnCalendar=weekly
```

Но можно указывать и конкретное расписание.

Например:

```ini
OnCalendar=*-*-* 02:00:00
```

Это запуск каждый день в 02:00.

---

# Каждый день в 09:30

```ini
OnCalendar=*-*-* 09:30:00
```

---

# По будням

Например:

```ini
OnCalendar=Mon..Fri 09:00
```

Timer будет запускаться с понедельника по пятницу в 09:00.

---

# Как проверить выражение OnCalendar

Одна из очень удобных возможностей systemd:

```bash
systemd-analyze calendar
```

Например:

```bash
systemd-analyze calendar "*-*-* 02:00:00"
```

Команда покажет, как systemd понимает расписание и когда наступит следующий запуск.

Ещё пример:

```bash
systemd-analyze calendar "Mon..Fri 09:00"
```

Это намного удобнее, чем гадать, правильно ли написано выражение.

---

# Запускаем timer

После создания файлов:

```bash
sudo systemctl daemon-reload
```

Теперь включим timer:

```bash
sudo systemctl enable --now mytask.timer
```

Здесь:

```text
enable → запускать timer автоматически после загрузки

--now  → запустить timer прямо сейчас
```

Проверяем:

```bash
systemctl status mytask.timer
```

---

# Проверяем timer в списке

```bash
systemctl list-timers
```

Ищем:

```text
mytask.timer
```

Можно сразу отфильтровать:

```bash
systemctl list-timers mytask.timer
```

Теперь мы можем увидеть время следующего запуска.

---

# Как остановить timer

Остановить:

```bash
sudo systemctl stop mytask.timer
```

Отключить автозапуск:

```bash
sudo systemctl disable mytask.timer
```

Сразу остановить и отключить:

```bash
sudo systemctl disable --now mytask.timer
```

---

# Persistent=true

Одна из полезных возможностей календарных systemd timers:

```ini
Persistent=true
```

Представим timer:

```ini
OnCalendar=daily
```

который должен выполняться каждый день в 02:00.

Но компьютер в это время был выключен.

При обычном расписании запуск будет пропущен.

С:

```ini
Persistent=true
```

systemd запоминает предыдущие срабатывания timer.

Если во время простоя системы должен был произойти запуск, после следующей активации timer systemd может выполнить пропущенную задачу.

Это особенно удобно для:

- ноутбуков;
- рабочих станций;
- серверов, которые периодически выключаются;
- резервного копирования;
- задач обслуживания.

Важно: `Persistent=` относится к timers, использующим `OnCalendar=`.

---

# OnBootSec

Не все timers привязаны к часам и календарю.

Можно запускать задачу через определённое время после загрузки системы.

Например:

```ini
[Timer]
OnBootSec=5min
```

Это означает:

```text
запустить задачу через 5 минут после загрузки
```

---

# OnUnitActiveSec

Можно запускать задачу через определённый интервал после предыдущего запуска.

Например:

```ini
[Timer]
OnBootSec=5min
OnUnitActiveSec=30min
```

Логика:

```text
загрузка системы
      ↓
5 минут
      ↓
первый запуск
      ↓
30 минут
      ↓
следующий запуск
      ↓
30 минут
      ↓
следующий запуск
```

Так можно создавать периодические задачи без календарного расписания.

---

# Calendar timers и monotonic timers

У systemd timers есть два основных подхода к расписанию.

## Календарное время

Используем:

```ini
OnCalendar=
```

Например:

```ini
OnCalendar=*-*-* 02:00:00
```

Это означает:

> каждый день в два часа ночи.

---

## Интервал относительно события

Используем:

```ini
OnBootSec=
OnActiveSec=
OnUnitActiveSec=
OnUnitInactiveSec=
```

Например:

```ini
OnBootSec=10min
```

означает:

> через десять минут после загрузки.

А:

```ini
OnUnitActiveSec=1h
```

означает:

> через час после предыдущей активации unit.

---

# Где смотреть логи

Одна из сильных сторон systemd timers — интеграция с journal.

Сам timer:

```bash
journalctl -u mytask.timer
```

Сервис:

```bash
journalctl -u mytask.service
```

Последние события:

```bash
journalctl -u mytask.service -n 50
```

Следить в реальном времени:

```bash
journalctl -u mytask.service -f
```

Это удобно при поиске ошибок.

---

# Проверяем состояние service

```bash
systemctl status mytask.service
```

Проверяем timer:

```bash
systemctl status mytask.timer
```

И список timers:

```bash
systemctl list-timers
```

Получается удобный набор диагностики:

```text
systemctl status
systemctl list-timers
journalctl
```

---

# Если изменили .timer или .service

После редактирования unit-файлов нужно сообщить systemd об изменениях:

```bash
sudo systemctl daemon-reload
```

После этого можно перезапустить timer:

```bash
sudo systemctl restart mytask.timer
```

Это важный шаг, о котором часто забывают.

---

# Одноразовая задача через systemd-run

Необязательно всегда вручную создавать `.service` и `.timer`.

Для временных задач есть:

```bash
systemd-run
```

Например:

```bash
systemd-run --on-active=5m /usr/local/bin/mytask.sh
```

Systemd создаст временные timer и service units.

Посмотреть их можно через:

```bash
systemctl list-timers
```

---

# Запуск по календарю через systemd-run

Можно использовать:

```bash
systemd-run --on-calendar="tomorrow" /usr/local/bin/mytask.sh
```

Таким образом `systemd-run` позволяет быстро создать задачу по расписанию без постоянных unit-файлов.

Это в некотором смысле похоже на использование `at`, но уже через инфраструктуру systemd.

---

# Cron или systemd timers

Один из главных вопросов:

> Нужно ли теперь отказываться от cron?

Нет.

У обоих вариантов есть свои преимущества.

## Cron

Простой пример:

```text
0 2 * * * /home/user/backup.sh
```

Плюсы:

- простой;
- привычный;
- короткая запись;
- работает на огромном количестве Unix/Linux-систем;
- отлично подходит для небольших периодических задач.

---

## Systemd timer

Здесь обычно есть:

```text
backup.timer
backup.service
```

Плюсы:

- интеграция с `systemctl`;
- логи через `journalctl`;
- зависимости между units;
- управление окружением и пользователем через service unit;
- `Persistent=true`;
- календарные и относительные timers;
- единая система управления сервисами и задачами.

---

# Что выбрать

Для простой задачи:

```text
каждый день запустить небольшой скрипт
```

`cron` может быть самым простым решением.

Если задача уже является частью инфраструктуры systemd, нужны зависимости, журналирование, контроль состояния и более гибкое управление — systemd timer часто оказывается удобнее.

Поэтому правильнее думать не:

> systemd timers заменили cron.

А:

> это два разных инструмента для похожих задач.

---

# Cron, at и systemd timers

Теперь наш блок планирования задач можно собрать полностью.

## cron

```text
регулярные задачи
```

Например:

```text
каждый день
каждые 5 минут
по понедельникам
```

---

## at

```text
одноразовая задача
```

Например:

```text
через 20 минут
сегодня в 23:00
завтра утром
```

---

## systemd timers

```text
задачи, интегрированные с systemd
```

Например:

```text
запустить service каждый день
через 5 минут после загрузки
каждые 30 минут после предыдущего запуска
```

---

# Практический пример целиком

Создаём service:

```bash
sudo nano /etc/systemd/system/backup.service
```

```ini
[Unit]
Description=Backup task

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

Создаём timer:

```bash
sudo nano /etc/systemd/system/backup.timer
```

```ini
[Unit]
Description=Run backup every day

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Обновляем systemd:

```bash
sudo systemctl daemon-reload
```

Включаем:

```bash
sudo systemctl enable --now backup.timer
```

Проверяем:

```bash
systemctl list-timers
```

Смотрим состояние:

```bash
systemctl status backup.timer
```

Логи:

```bash
journalctl -u backup.service
```

Готово.

---

# Краткая шпаргалка systemd timers

Посмотреть timers:

```bash
systemctl list-timers
```

Все timers:

```bash
systemctl list-timers --all
```

Проверить timer:

```bash
systemctl status mytask.timer
```

Запустить:

```bash
sudo systemctl start mytask.timer
```

Включить и запустить:

```bash
sudo systemctl enable --now mytask.timer
```

Остановить:

```bash
sudo systemctl stop mytask.timer
```

Перечитать unit-файлы:

```bash
sudo systemctl daemon-reload
```

Проверить календарное выражение:

```bash
systemd-analyze calendar "Mon..Fri 09:00"
```

Посмотреть логи:

```bash
journalctl -u mytask.service
```

Создать временный timer:

```bash
systemd-run --on-active=5m /usr/local/bin/mytask.sh
```

---

# Что важно знать для LPIC-1

Systemd timers входят в тему **LPIC-1 Objective 107.2 — автоматизация административных задач по расписанию**.

В рамках этой темы важно понимать:

```text
cron
at
systemd timers
systemctl
systemd-run
```

Для systemd timers полезно уметь:

- понимать назначение `.timer` unit;
- понимать связь `.timer` и `.service`;
- посмотреть timers;
- запустить и включить timer;
- понимать принцип календарного и интервального расписания.

---

# Что стоит запомнить

Самая важная схема:

```text
.timer
  ↓
.service
  ↓
команда или скрипт
```

Основные команды:

```bash
systemctl list-timers
systemctl status name.timer
systemctl enable --now name.timer
journalctl -u name.service
systemd-analyze calendar
systemd-run
```

И основные параметры:

```ini
OnCalendar=
OnBootSec=
OnUnitActiveSec=
Persistent=true
```

Не нужно запоминать весь синтаксис systemd timers.

Гораздо важнее понимать принцип и знать, где проверить расписание и состояние задачи.

---

# Презентация и видео по теме

[Скачать презентацию](https://github.com/solizarevich/solizarevich.github.io/blob/main/content/post/2026-11-05-timers/systemd_timers_linux_presentation.pdf)

➡️[Rutube](https://rutube.ru/video/e7ade74b6e1281c611101e737267dde7/)

➡️[VK](https://vkvideo.ru/video-234521173_456239991)

➡️[YouTube](https://youtu.be/A-fc8LHPbVw)


---

## Документация

- [systemd.timer — официальная документация](https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html)  
  Timer units, `OnCalendar`, `OnBootSec`, `OnUnitActiveSec`, `Persistent` и другие параметры расписания.

- [systemctl — официальная документация](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html)  
  Управление `.timer` и `.service` units, `systemctl list-timers`, `start`, `stop`, `enable`, `status`.

- [systemd.time — синтаксис времени](https://www.freedesktop.org/software/systemd/man/latest/systemd.time.html)  
  Форматы времени и календарных выражений, используемых в `OnCalendar`.

- [systemd-analyze — официальная документация](https://www.freedesktop.org/software/systemd/man/latest/systemd-analyze.html)  
  Проверка и разбор конфигурации systemd. Команда `systemd-analyze calendar` помогает проверить выражение `OnCalendar` и увидеть время следующего запуска.

- [systemd-run — официальная документация](https://www.freedesktop.org/software/systemd/man/latest/systemd-run.html)  
  Создание временных service и timer units без ручного создания файлов.

- [LPIC-1 Exam Objectives — Objective 107.2](https://www.lpi.org/our-certifications/exam-101-102-objectives/)  
  Требования LPIC-1 по автоматизации задач: `cron`, `at` и systemd timers.
