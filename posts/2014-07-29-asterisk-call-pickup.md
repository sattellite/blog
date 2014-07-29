Перехват вызова - очень полезный функционал. Для перехвата входящего вызова в Asterisk есть 3 способа. Первый - использовать функционал `features`; второй - через переменные канала вызова; третий - специальная команда `PickUp()`.

Если разобрать перехват вызова в общих чертах, то получится следующее:

- Номерам настраивается группа вызова (`callgroup`)
- Номерам настраивается группа для перехвата (`pickupgroup`)
- Настраивается комбинация клавиш для перехвата вызова

## Настройка через использование конфигурационных файлов

В `sip.conf` для номеров настраивается численное представление `callgroup` и `pickupgroup` или именованное представление `namedcallgroup` и `namedpickupgroup`:

    [number1]
    callgroup=1-3
    namedcallgroup=sales
    pickupgroup=1-3,9
    namedpickupgroup=sales
    ... ... ...
    [numberX]
    callgroup=9,1-3
    namedcallgroup=manager
    pickupgroup=1-3,9
    namedpickupgroup=manager,sales

`callgroup` указывает к каким группам будет принадлежать вызов, а `pickupgroup` указывает из каких групп можно перехватывать вызов.
# Перехват входящего вызова в Asterisk

## Настройка через использование переменных канала

Через переменные канала производится настройка в `extensions.conf`. При обработке вызова указывается к какой именованной (`namedcallgroup` и `namedpickupgroup`) или численной (`callgroup` и `pickupgroup`) группе принадлежит данный вызов:

    same => n,Set(CHANNEL(callgroup)=1,5-7)
    same => n,Set(CHANNEL(namedcallgroup)=engineering,sales)

    same => n,Set(CHANNEL(pickupgroup)=1,6-8)
    same => n,Set(CHANNEL(namedpickupgroup)=engineering,sales)


В `features.conf` настраивается комбинация клавиш для перехвата вызова:

    [general]
    ...
    pickupexten = *8
    ...

Теперь при нажатии на `*8` будет перехвачен вызов на один из номеров в разрешенной группе для перехвата.

## Перехват вызова с помощью команды PickUp

В файле `extensions.conf` прописывается экстен и команда для перехвата вызова:

    same => n,PickUp()

Если команда используется без аргументов, то будет перехвачен вызов из разрешенной группы.

    same => n,PickUp(${EXTEN}@context)

Если при вызове команды указать номер и контекст, то будет осуществлена попытка перехвата конкретного номера в указанном контексте. Если не указать контекст, то будет использоваться текущий.

С помощью команды `PickUp` можно перехватывать вызовы на номера, которые не принадлежат ни к одной из `callgroup`.

*Вся информация собрана из полезных ресурсов: [ресурс #1](https://wiki.asterisk.org/wiki/display/AST/Call+Pickup), [ресурс #2](http://www.voip-info.org/wiki/view/Asterisk+callgroups+and+pickupgroups), [ресурс #3](https://wiki.asterisk.org/wiki/display/AST/Asterisk+11+Application_Pickup), [ресурс #4](http://www.voip-info.org/wiki/view/Asterisk+cmd+Pickup).*
