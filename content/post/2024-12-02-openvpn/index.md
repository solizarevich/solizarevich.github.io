---
title:  "Простой способ создать OpenVPN сеть"
date: 2024-12-02 11:20:00 +0300
categories:
  - Network
tags:
  - vpn
  - network
---

Многие в наше тяжелое время слежения всего и вся хотят немного обезопасить себя от внешнего воздействия. Одним из выходов из этой ситуации является VPN сеть до какой-нибудь VDS. Под катом один из простейших способов создать такую сеть.
Всё будем поднимать на Debian.
 
 **Вводные данные** 
 
test - имя нашего сервера
192.168.10.254 - "внешний" IP сервера
192.168.1.254 - "внутренний" IP сервера
192.168.1.0/24 - "внутрення" сеть сервера
192.168.2.254 - OpenVPN адрес сервера 
192.168.2.0/24 - OpenVPN сеть
192.168.11.0/24 - "внутренняя" сеть клиента
user - пользователь, который сможет забрать ключи и конфиги с сервера
 
**Некоторые параметры конфигов**

    mode — режим работы впн
    proto — протокол передачи данных
    port — порт на котором работает впн
    tls-server — включение TLS режима авторизации
    dev — имя сетевого интерфейса
    daemon — режим работы
    tls-auth, ca, cert, key, dh — путь к соответствующим ключам
    ifconfig — IP адрес интерфейса, описанного dev
    ifconfig-pool — пул адресов клиентов
    route — описание маршрутов
    verb — уровень отладки
    cipher — тип шифрования сети
    persist-key — запрет перечитывания ключей через SIGUSR1
    log — файл лога
    comp-lzo — использование сжатия LZO
    client-to-client — режим, при котором клиенты видят друг друга
    client-config-dir — указание директории, в которой находятся описания клиентов
    status — указание пути к логу состояния
    tls-client — tls режим авторизации клиента
     remote — внешний адрес сервера OpenVPN
     pull — опция для клиента, который подключается к серверу с многими клиентами
     auth-retry — описание реагирования клиента на различные ошибки

**Установим софт**
 
 apt update && apt install openvpn easy-rsa
 
Создадим необходимые каталоги
 
 mkdir /etc/openvpn/{keys,conf,scr,clean,hosts}
 
Скопируем RSA ключи
 
 cp -r /usr/share/easy-rsa/* /etc/openvpn/scr/
 
 Приступим к серверной части
 
 cd /etc/openvpn/scr/
 
Изменим необходимые строки на ваши данные
 
 nano vars

    export KEY_COUNTRY="RU" 
    export KEY_PROVINCE="Kirov Region" 
    export KEY_CITY="Kirov" 
    export KEY_ORG="Interra Ltd." 
    export KEY_EMAIL="info@interra.com.ru"
    #export KEY_OU="MyOrganizationalUnit"

Генерируем ключи
 

     source ./vars
    ./clean-all
    ./build-ca
    ./build-dh
    ./build-key-server servername

 
 nano /etc/openvpn/scr/keys/index.txt.attr

    unique_subject = no

**Создадим TLS-auth ключ**
 

     openvpn --genkey --secret auth.key

 
Переносим данные ключи в папку с ключами сервера, например, в /etc/openvpn/keys/test-serv/

    auth.key  ca.crt  dh2048.pem  test-serv.crt  test-serv.key

**Создадим конфиг сервера** 
 
 nano /etc/openvpn/conf/test-serv.conf

    mode server
    tls-server
    proto udp
    dev tap100
    port 1202
    daemon
    tls-auth /etc/openvpn/keys/test-serv/auth.key 0
    ca /etc/openvpn/keys/test-serv/ca.crt
    cert /etc/openvpn/keys/test-serv/test-serv.crt
    key /etc/openvpn/keys/test-serv/test-serv.key
    dh /etc/openvpn/keys/test-serv/dh2048.pem
    ifconfig 192.168.2.254 255.255.255.0
    ifconfig-pool 192.168.2.1 192.168.2.100
    route 192.168.11.0 255.255.255.0 192.168.2.1
    verb 3
    cipher DES-EDE3-CBC
    persist-key
    log /var/log/ovpn-srv.log
    comp-lzo
    keepalive 20 70
    client-to-client
    client-config-dir /etc/openvpn/hosts
    status /var/log/ovpn-srv-status.log

**Сделаем симлинк конфига** 
 
 ln -s /etc/openvpn/conf/test-serv.conf /etc/openvpn/
 
 Для клиентской части
 
Для удобства генерации ключей и конфигов сделаем следующее:
 
 mkdir -p /etc/openvpn/clean/keys/test
 
 nano /etc/openvpn/clean/clean.conf

    tls-client
    proto udp
    remote 192.168.10.254   #внешний адрес нашего сервера
    dev tap10
    port 1202
    pull
    tls-auth /etc/openvpn/keys/test/auth.key 1
    ca /etc/openvpn/keys/test/ca.crt
    cert /etc/openvpn/keys/test/clean-test.crt
    key /etc/openvpn/keys/test/clean-test.key
    cipher DES-EDE3-CBC
    comp-lzo
    route 192.168.1.0 255.255.255.0 192.168.2.254
    log /var/log/ovpn-test.log
    auth-retry nointeract

cp /etc/openvpn/scr/keys/auth.key /etc/openvpn/clean/keys/test
cp /etc/openvpn/scr/keys/ca.crt /etc/openvpn/clean/keys/test
cp /etc/openvpn/scr/keys/dh2048.pem /etc/openvpn/clean/keys/test
 
 **Скрипт автоматической генерации ключей**
 
 nano /usr/local/bin/add-vpn.sh

    #!/bin/bash
     
    # globals
    COLOR_WHITE='\e[1;37m'
    COLOR_GREEN='\e[1;33m'
    COLOR_END='\e[0m'
     
    PROMPT=$COLOR">> "$COLOR_END
     
    CLEAN_DIR=/etc/openvpn/clean
    CONF_FILE=/etc/openvpn/clean/clean.conf
    KEYS_DIR=/etc/openvpn/clean/keys/test
    SRC_DIR=/etc/openvpn/scr
    TMP_DIR=`mktemp -d`
    EDITOR=nano

    echo 'Files will be stored in $TMP_DIR'
    cp -r "$CLEAN_DIR"/keys "$TMP_DIR"/keys
    # beautiful functions
    read_colored() {
        # args: text_prompt, variable
        echo -e $1
        echo -e -n $COLOR_WHITE'>> '
        read $2
        echo -e -n $COLOR_END
    }
    echo 'What the hostname is?'
    read_colored "Key name will be $COLOR_GREEN<hostname>-test$COLOR_END" HOSTNAME
    HOSTNAMECT=$HOSTNAME-test
    cd "$SRC_DIR"
    source ./vars
    ./build-key $HOSTNAMECT
    cp "$SRC_DIR"/keys/$HOSTNAMECT.* "$TMP_DIR"/keys/test
    cat "$CONF_FILE" | sed "s/clean-test/$HOSTNAMECT/" > "$TMP_DIR"/$HOSTNAMECT.conf
    #$EDITOR /etc/dnsmasq.hosts
    #/etc/init.d/dnsmasq restart
    echo 'ifconfig-push 192.168.2.XXX 255.255.255.0' > /etc/openvpn/hosts/$HOSTNAMECT
    $EDITOR /etc/openvpn/hosts/$HOSTNAMECT
    chown user:user $TMP_DIR -R
    mv $TMP_DIR /tmp/openvpn-$HOSTNAME
    echo -e "Your files available in $COLOR_WHITE/tmp/openvpn-$HOSTNAME$COLOR_END"

Меняете user на своего пользователя, которому нужно будет забирать ключи
 
Строка EDITOR=nano указывает на то, каким редактором будут открываться конфиги, которые нужно поправить
 
Если хотите резолвить имена ВПН, то воспользуйтесь, например, dnsmasq и раскомментируйте строку #$EDITOR /etc/dnsmasq.hosts с введением соответствующего конфига в /etc/dnsmasq.conf (addn-hosts=/etc/dnsmasq.hosts)
 
 chmod 777 /usr/local/bin/add-vpn.sh
 
Запускаете скрипт, соглашаясь со всем, что вам предложат (не забудьте нажать "y", когда будет предложено, иначе не будут сгенерированны ключи)
 
После выполнения скрипта, все файлы, необходимые для впн, будут лежать в /tmp, их желательно запаковать и отправить на клиентскую машинку
 
Дальше добавляем все это дело в автозагрузку любым удобным вам способом
 
Хочу заметить, что IP адреса брались наугад, поэтому подставляйте свои

