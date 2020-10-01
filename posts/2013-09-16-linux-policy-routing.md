---
tags: linux, iptables, network
noedit: true
---

# Linux Policy-Based Routing

Заметка из выдержек LARTC.

Имеется сервер/десктоп с двумя сетевыми интерфейсами. Каждый интерфейс подключен к своей сети. И возникает ситуация, что ответ должен отправляться в сеть из которой пришел запрос, а не через default gw. Такая маршрутизации называется policy based. В Linux она настраивается с помощью утилиты iproute2.

Первый интерфейс с адресом 10.120.120.10/16 и шлюзом 10.120.0.1. Второй интерфейс с адресом 172.16.16.172/16 и шлюзом 172.16.0.1.

Добавим две таблицы маршрутизации с произвольными именами (в моем случае это просто первый октет сети):

    /usr/sbin/ip route add default via 10.120.0.1 table 10
    /usr/sbin/ip route add default via 172.16.0.1 table 172

Добавим два правила маршрутизации, указывающие на эти таблицы:

    /usr/sbin/ip rule add from 10.120.120.10 table 10
    /usr/sbin/ip rule add from 172.16.16.172 table 172

Краткое описание: все что уходит с адреса 10.120.120.10 попадает в таблицу 10, а у нее шлюзом указан 10.120.0.1 и точно такая же ситуация со вторым интерфейсом.

В итоге мы получаем source based маршрутизацию с default gw и policy based маршрутизацию для отправки ответов с того интерфейса, на который пришел запрос.

    # /usr/sbin/ip route show table 10

      default via 10.120.0.1 dev p2p1

    # /usr/sbin/ip route show table 172

      default via 172.16.0.1 dev em1

    # /usr/sbin/ip route show table main

      default via 10.120.0.1 dev p2p1  proto static
      10.120.0.0/16 dev p2p1  proto kernel  scope link  src 10.120.120.10  metric 1
      172.16.0.0/16 dev em1  proto kernel  scope link  src 172.16.16.172  metric 1

    # /usr/sbin/ip rule show

      0:  from all lookup local
      32764:  from 172.16.16.172 lookup 172
      32765:  from 10.120.120.10 lookup 10
      32766:  from all lookup main
      32767:  from all lookup default

#### Автоматический запуск в Fedora Linux

Если добавить эти строки в `/etc/rc.d/rc.local`, то оно не всегда будет срабатывать так как надо. Сервис `networking` считается рабочим когда запущен хотя бы один интерфейс, а второй еще нет. Поэтому пойдем другим путем.

Известно, что при поднятии интерфейса запускается `/etc/sysconfig/network-scripts/ifup-post`, который в свою очередь проверяет наличие скрипта `/sbin/ifup-local`:

    # grep ifup-local /etc/sysconfig/network-scripts/ifup-post

    if [ -x /sbin/ifup-local ]; then
        /sbin/ifup-local ${DEVICE}

Создаем скрипт `/sbin/ifup-local` следующего содержания:
    # cat /sbin/ifup-local

    #!/bin/bash
    if [[ "$1" == "p2p1" ]]; then
      /usr/sbin/ip route add default via 10.120.0.1 table 10
      /usr/sbin/ip rule add from 10.120.120.10 table 10
    fi

    if [[ "$1" == "em1" ]]; then
      /usr/sbin/ip route add default via 172.16.0.1 table 172
      /usr/sbin/ip rule add from 172.16.16.172 table 172

    fi

И делаем его исполняемым:

    # chmod +x /sbin/ifup-local

Включаем сервис `network` и перезагружаемся:

    # chkconfig network on
    # reboot

