---
tags: linux, fedora, gre, network
noedit: true
---

# Создание туннеля в Fedora Linux

Причин, по которым может понадобиться туннель крайне много. Статей о том как его настраивать ещё больше.
Здесь я собрал несколько статей в одну про создание `GRE` туннеля, чтобы, с учетом всех мелочей, был полный мануал по настройке.

<img src="/images/tunnel.jpg" alt="http://en.wikipedia.org/wiki/List_of_tunnels_in_New_Zealand#mediaviewer/File:Okau_Road_tunnel.jpg" title="Okau Road tunnel">

Для начала необходимо подгрузить соответствующий модуль ядра и проверить, что он загрузился:

    # modprobe ip_gre
    # lsmod|grep gre

    ip_gre                 18245  0
    ip_tunnel              23081  1 ip_gre
    gre                    13535  1 ip_gre

Заранее надо условиться, что адрес сервера 1.2.3.4, адрес клиента 5.6.7.8, внутренний адрес сервера 192.168.10.1, внутренний адрес на клиенте 192.168.10.2.

### Создаем туннель на сервере

Добавляется интерфейс туннеля `gre1` и назначается ему адрес:

    # ip tunnel add gre1 mode gre remote 5.6.7.8 local 1.2.3.4 ttl 255
    # ip link set gre1 up
    # ip addr add 192.168.10.1/24 dev gre1

Проверим, что создался туннель, устройство и необходимый маршрут:

    # ip tunnel show gre1
    gre1: gre/ip  remote 5.6.7.8  local 1.2.3.4  ttl 255

    # ip link show gre1
    19: gre1@NONE: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1476 qdisc noqueue state UNKNOWN mode DEFAULT group default
        link/gre 1.2.3.4 peer 5.6.7.8

    # ip route show
    default via 10.10.10.1 dev p2p1  proto static
    192.168.10.0/24 dev gre1  proto kernel  scope link  src 192.168.10.1

### Создаем туннель от клиента до сервера

Процесс создания абсолютно такой же как и на сервере, только меняются адреса местами.

Добавляется интерфейс туннеля `gre1` и назначается ему адрес:

    # ip tunnel add gre1 mode gre remote 1.2.3.4 local 5.6.7.8 ttl 255
    # ip link set gre1 up
    # ip addr add 192.168.10.2/24 dev gre1

Проверим, что создался туннель, устройство и необходимый маршрут:

    # ip tunnel show gre1
    gre1: gre/ip  remote 1.2.3.4  local 5.6.7.8  ttl 255

    # ip link show gre1
    19: gre1@NONE: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1476 qdisc noqueue state UNKNOWN mode DEFAULT group default
        link/gre 5.6.7.8 peer 1.2.3.4

    # ip route show
    default via 20.20.20.1 dev p2p1  proto static
    192.168.10.0/24 dev gre1  proto kernel  scope link  src 192.168.10.1

Проверим работоспособность туннеля:

    # ping 192.168.10.1

    PING 192.168.10.1 (192.168.10.1) 56(84) bytes of data.
    64 bytes from 192.168.10.1: icmp_seq=1 ttl=64 time=58.7 ms
    64 bytes from 192.168.10.1: icmp_seq=2 ttl=64 time=58.6 ms
    64 bytes from 192.168.10.1: icmp_seq=3 ttl=64 time=58.9 ms
    64 bytes from 192.168.10.1: icmp_seq=4 ttl=64 time=58.5 ms
    ^C
    --- 192.168.10.1 ping statistics ---
    4 packets transmitted, 4 received, 0% packet loss, time 3005ms
    rtt min/avg/max/mdev = 58.507/58.717/58.930/0.285 ms

### Автоматизация процесса установления туннеля

Добавим подключение модуля ядра для GRE при старте системы:

    # echo "/sbin/modprobe ip_gre > /dev/null 2>&1" > /etc/sysconfig/modules/ip_gre.modules && chmod 755 /etc/sysconfig/modules/ip_gre.modules

Автоматизируем поднятие интерфейса со всеми необходимыми настройками в файле `/etc/sysconfig/network-scripts/ifcfg-gre1`:

    TYPE=GRE
    DEVICE=gre1
    BOOTPROTO="none"
    ONBOOT="yes"
    PEER_OUTER_IPADDR=1.2.3.4
    MY_OUTER_IPADDR=5.6.7.8
    PEER_INNER_IPADDR=192.168.10.1
    MY_INNER_IPADDR=192.168.10.2
    TTL=255

А теперь можно проверить, что настройка интерфейса была произведена правильно с помощью команд `ifup` и `ifdown`. Сначала отключим туннель, а затем включим его:

    # ifdown gre1 && ifup gre1

После этого туннель должен будет перезапуститься и можно проверить его работоспособность.

### Автоматическое добавление необходимых маршрутов для туннеля

Для автоматического добавления маршрутов для туннеля можно использовать файл `/etc/sysconfig/network-scripts/route-gre1`. Маршруты в нем пишутся как и с помощью утилиты `ip`, но с опусканием `ip route add`:

    8.8.8.8 dev gre1
    8.8.4.4 dev gre1

На этом всё.


#### Используемые статьи:

* [http://habrahabr.ru/post/47230/](http://habrahabr.ru/post/47230/)
* [http://juliano.info/en/Blog:Memory_Leak/Bridges_and_tunnels_in_Fedora](http://juliano.info/en/Blog:Memory_Leak/Bridges_and_tunnels_in_Fedora)
* [http://centoshowtos.org/network-security/gre-tunnel/](http://centoshowtos.org/network-security/gre-tunnel/)
* [http://ask.xmodulo.com/create-gre-tunnel-linux.html](http://ask.xmodulo.com/create-gre-tunnel-linux.html)

