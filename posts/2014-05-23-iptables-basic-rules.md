# Создание правил iptables

Когда сервер работает по правилам ***"всё разрешено, кроме"*** это конечно хорошо. Но часто все забывают добавлять те
самые правила ***"кроме"***. И сервер остается не защищенным от внешних атак. А если сервер в продакшене, то он всегда
должен работать по правилу ***"запрещено всё, кроме"***. Здесь разберем способ основной защиты сервера от атак из
внешнего мира.

Просмотреть существующие правила можно с помощью

    % sudo iptables -L

Вывод будет приблизительно такого вида:

    Chain INPUT (policy ACCEPT)
    target     prot opt source               destination

    Chain FORWARD (policy ACCEPT)
    target     prot opt source               destination

    Chain OUTPUT (policy ACCEPT)
    target     prot opt source               destination

Все таблицы пустые и не содержат в себе ни одного правила.

### Базовая настройка для web-сервера

Однозначно уже определились, какой функционал предоставляет сервер. Здесь приводится пример для web-сервера.

Разрешить принимать траффик уже установленным сессиям (когда сессия устанавливается с самого сервера, например пинг):

    % sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

Разрешить принимать трафик на определенный порт:

    % sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
    % sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

Разрешить весь трафик на интерфейсе `lo`, а то мало ли какому сервису понадобится:

    % sudo iptables -A INPUT -i lo -j ACCEPT

Разрешить входящий траффик с определенного ip:

    % sudo iptables -A INPUT -s 8.8.8.8/32 -j ACCEPT

А теперь запретить всё остальное:

    % sudo iptables -A INPUT -j DROP

Теперь можно наблюдать за работой правил с помощью команды:

    % sudo iptables -L -n -v --line-numbers

    Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
    num   pkts bytes target prot opt in out source    destination
    1        0     0 ACCEPT all  --  *  *   0.0.0.0/0 0.0.0.0/0   ctstate RELATED,ESTABLISHED
    2        0     0 ACCEPT tcp  --  *  *   0.0.0.0/0 0.0.0.0/0   tcp dpt:80
    3        0     0 ACCEPT tcp  --  *  *   0.0.0.0/0 0.0.0.0/0   tcp dpt:22
    4        0     0 ACCEPT all  --  lo *   0.0.0.0/0 0.0.0.0/0
    5        0     0 ACCEPT all  --  *  *   8.8.8.8   0.0.0.0/0
    6        0     0 DROP   all  --  *  *   0.0.0.0/0 0.0.0.0/0

    Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
    num   pkts bytes target prot opt in out source destination

    Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
    num   pkts bytes target prot opt in out source destination

### Редактирование iptables

Добавление правила в конец производится с помощью ключа `-A`.

    % sudo iptables -A INPUT $(NEW RULE)

Вставка правила перед определенной строкой (номера строк узнаются ключом `--line-numbers`) производится с помощью ключа `-I`.

    % sudo iptables -I INPUT 6 $(NEW RULE)

Исправление определенного правила делается с помощью ключа `-R`.

    % sudo iptables -R INPUT 6 $(NEW RULE)

Удаление определенного правила производится ключом `-D`.

    % sudo iptable -D INPUT 6

Полная очистка всех правил:

    % sudo iptables -F
    % sudo iptables -X

Добавление записи с комментарием:

    % sudo iptables -I INPUT 6 -s 8.8.8.8/32 -j ACCEPT -m comment --comment "Allow input traffic from Google DNS"


На много подробнее можно все узнать в `man iptables` и [тут](https://help.ubuntu.com/community/IptablesHowTo).


