---
tags: asterisk, linux, freebsd
noedit: true
---

# Проблемы точного времени в Asterisk

Источник точного времени крайне необходим для синхронизации любого медиа-потока. Asterisk не исключение. С помощью источника точного времени синхронизируются потоки во время разговора, воспроизводятся файлы, подавляется тишина и прочий функционал так или иначе затрагивающий передачу медиа-потока.

По умолчанию Asterisk установленный в Linux дистрибутиве использует `timerfd` (res_timing_timerfd.so). И с ним проблем нет. Но вот при установке Asterisk в FreeBSD он использует модуль `kqueue` (res_timing_kqueue.so). Начинает возникать куча странных проблем (например, загрузка процессора при воспроизведении moh, moh проигрывается слишком быстро/медленно, плохое качество голоса во время вызова, односторонняя слышимость или полная тишина во время разговора) и ошибок:

    [Nov  1 08:47:14] WARNING[100187][C-000000a4]: channel.c:1309 __ast_queue_frame: Exceptionally long queue length queuing to SIP/xxx

Asterisk не проходит тесты для интерфейса точного времени:

    asterisk*CLI> timing test
    Attempting to test a timer with 50 ticks per second.
    Using the 'kqueue' timing module for this test.
    Timer failed to acknowledge.
    Command 'timing test' failed.

Проверка используемого модуля для интерфейса точного времени:

    asterisk*CLI> module show like timing
    Module                         Description                              Use Count
    res_timing_kqueue.so           KQueue Timing Interface                  1
    res_timing_pthread.so          pthread Timing Interface                 0
    2 modules loaded

Asterisk всегда хорошо работает через `pthread`, надо использовать его. Для устранения подобной проблемы отключим модуль `res_timing_kqueue.so`. Для этого в modules.conf прописывается:

    noload => res_timing_kqueue.so

Снова проверяем используемые модули:

    asterisk*CLI> module show like timing
    Module                         Description                              Use Count
    res_timing_pthread.so          pthread Timing Interface                 1
    1 modules loaded

После такой манипуляции все становится хорошо. Каждый вызов не отбирает кучу процессорного времени, отлично воспроизводятся файлы.

Если есть необходимость использовать какой-то специфичный модуль для интерфейса времени, то надо указать его предзагрузку в `modules.conf`, чтобы модуль загрузился первым:

    preload => res_timing_dahdi.so


