---
tags: docker
noedit: true
---

# Удаление контейнеров и образов в Docker

Эта заметка немного выбивается из того цикла, который я бы хотел написать. Все дело в том, что ранее я пытался пользоваться `docker`'ом и это было не совсем удачно с кучей непонятностей и неприятностей. Сейчас же я решил снова попробовать использовать `docker` и начать все с начала.
А предварительно необходимо удалить все следы старых попыток.

> Все эти заметки в первую очередь предназначаются для меня, на тот случай, если я снова заброшу `docker` по каким-то причинам, а через некоторое время решу вернуться к нему.

## Удаление контейнеров и образов

Чтобы посмотреть все загруженные образы используется команда `docker images`:

    $ docker images

    REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
    ubuntu              latest              5ba9dab47459        36 hours ago        192.7 MB
    dougbtv/asterisk    latest              5d813382453e        7 weeks ago         814.7 MB
    stl-mysql           latest              cdeabd5abcbc        3 months ago        323.9 MB
    tutum/mysql         latest              90952d8d3872        3 months ago        322.6 MB
    <none>              <none>              d9a0b1eeff3f        4 months ago        435.5 MB
    ubuntu              14.04               96864a7d2df3        4 months ago        204.4 MB
    stackbrew/ubuntu    14.04               96864a7d2df3        4 months ago        204.4 MB
    fedora              rawhide             5cc9e91966f7        8 months ago        372.7 MB

Для того, чтобы удалить образ используется команда `docker rmi` и указывается `<IMAGE ID>`:

    $ docker rmi 5cc9e91966f7

    Untagged: fedora:rawhide
    Deleted: 5cc9e91966f7b9c84c44682cb76886d95f2d5c5e7e689770410af3f27471641a
    Deleted: 5af93badfee23d37c0314fee9e579fb908b1643740e59d68d7bd413bc590e2fe
    Deleted: 28767a75160cc0b314ed0f1ebb30a7b88f63faab61383e8591ff965449d1a0b2
    Deleted: 1633c925334475c2ce397a6659efd67fdf476cbdd424aba1dafc4d59dcbdb0e9
    Deleted: b7de3133ff989df914ae9382a1e8bb6771aeb7b07c5d7eeb8ee266b1ccff5709
    Deleted: ef52fb1fe61037a1b531698c093205f214ade751c781e30ce4f9a7d33020a0f2

Иногда образ нельзя удалить из-за того, что он используется каким-либо контейнером. Например, при удалении образа `tutum/mysql` он будет ругаться, что его использует контейнер `0ffd6237cd27`:

    $ docker rmi 90952d8d3872

    Error response from daemon: Conflict, cannot delete 90952d8d3872 because the container 0ffd6237cd27 is using it, use -f to force
    FATA[0000] Error: failed to remove one or more images

Для начала надо удалить контейнер использующий этот образ. Для того чтобы посмотреть все существующие контенеры используется команда `docker ps -a`:

    $ docker ps -a

    CONTAINER ID        IMAGE                     COMMAND                CREATED             STATUS                        PORTS               NAMES
    d388a1d9fce2        ubuntu:latest             "/bin/bash"            16 minutes ago      Exited (130) 14 minutes ago                       clever_hopper
    cd8cf44c24cd        ubuntu:latest             "/bin/bash"            16 hours ago        Exited (-1) 41 minutes ago                        ubuntu
    129dac32e556        dougbtv/asterisk:latest   "/bin/sh -c 'asteris   16 hours ago        Exited (0) 16 hours ago                           asterisk
    0ffd6237cd27        tutum/mysql:latest        "/run.sh"              3 months ago        Exited (-1) 3 months ago                          angry_pare
    4a9570d307c5        stl-mysql:latest          "/bin/sh -c 'mysql -   3 months ago        Exited (1) 3 months ago                           evil_wozniak
    9945be3bb4dd        stl-mysql:latest          "/bin/sh -c 'mysql -   3 months ago        Exited (1) 3 months ago                           romantic_turing
    2e9bc4317db1        d9a0b1eeff3f              "/bin/sh -c 'cd /con   4 months ago        Exited (1) 4 months ago                           silly_bell
    33213e783792        stackbrew/ubuntu:14.04    "/bin/echo 'Hello wo   4 months ago        Exited (0) 4 months ago                           goofy_wozniak
    60f0388a2a15        stackbrew/ubuntu:14.04    "/bin/echo 'Hello wo   4 months ago        Exited (0) 4 months ago                           happy_einstein

Из вывода этой команды можно получить информацию, что существует контейнер `0ffd6237cd27` использующий образ `tutum/mysql`. Это подтверждает ошибку при удалении образа. Надо удалить контейнер с помощью команды `docker rm`:

    $ docker rm 0ffd6237cd27

    0ffd6237cd27

А теперь можно спокойно удалить образ `tutum/mysql`:

    $ docker rmi 90952d8d3872

    Untagged: tutum/mysql:latest

Если образ приватный, то есть создан с помощью `docker build` и он отказывается удаляться, то в данной ситуации ничего лучше чем сделать `--force` удаление не нашел:

    $ docker rmi -f 96864a7d2df3

    Untagged: stackbrew/ubuntu:14.04

Вот и все по удалению контейнеров и образов. Если есть какие-то вопросы или пожелания, то оставляйте их в комментариях.


