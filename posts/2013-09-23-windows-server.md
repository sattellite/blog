# Базовая настройка Windows Server

![image](/images/fat_unicorn.png)

Пример базовой настройки Windows Server. Все будет происходить на Windows Server 2003.

Сам процесс установки мало интересен и все решения на возникшие, в его ходе, вопросы легко находятся в google.com.

Как правильно поднять [терминальный сервер](http://ru.wikipedia.org/wiki/%D0%A2%D0%B5%D1%80%D0%BC%D0%B8%D0%BD%D0%B0%D0%BB%D1%8C%D0%BD%D1%8B%D0%B9_%D1%81%D0%B5%D1%80%D0%B2%D0%B5%D1%80) можно почитать тут [http://networkcomp.ru/?p=385](http://networkcomp.ru/?p=385) (или копия в [Evernote](https://www.evernote.com/shard/s13/sh/56fd47fd-313c-4818-a08e-0e92650f937f/b02c763053ce37ee0a7791517a27c692))

После "поднятия" терминального сервера рекомендую к группе "Пользователи уделенного рабочего стола" добавить группу "Прошедшие проверку", чтобы при последующем создании пользователей не добавлять каждому эту группу.

![image](/images/win-serv-2k3-group-terminal.png)

Далее в глобальных политиках выбрать `Конфигурация компьютера → Конифгурация Windows → Параметры безопасности → Политики ограниченного использования программ` и с помощью ПКМ включить их. По умолчанию они имеют настройки, которые все разрешают, но надо это исправить.

![image](/images/win-serv-2k3-group-gpo-1.png)

Выбираем `Принудительный` и выставляем применять ко всем пользователям, кроме администраторов.

![image](/images/win-serv-2k3-group-gpo-2.png)

В `Уровнях безопасности` выбираем `Не разрешено` и делаем его по умолчанию. Тем самым мы всех пользователей ограничили в их правах.

![image](/images/win-serv-2k3-group-gpo-3.png)

Теперь можно смело добавлять разрешающие правила к уже существующим в `Дополнительных правилах`. Хорошая инструкция есть [на сайте Microsoft](http://support.microsoft.com/kb/324036/ru).

![image](/images/win-serv-2k3-group-gpo-4.png)

Как нас и предупреждает MS - мы не защитились от вирусов. Но значительно снизили вероятность их появления и распространения на сервере.

####Посмотреть uptime сервера
Способ подсмотрен [здесь](http://www.xakep.ru/post/42885/).

В консоли набрать `net statistics server` и посмотреть собственно с какого момента собирается статистика.

####Посмотреть логи обновлений

Нажать Win+R и ввести `windowsupdate.log`

####Неверный файл Update.inf/не запущена служба криптографии при обновлении

Странное поведение, конечно, но это ж Windows. Все включено, все перезагружено 10 раз, но проблема не уходит. Лечится
уделаением в реестре в ветке `HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\SystemCertificates\TrustedPublisher\Safer`
ключа `AuthenticodeFlags - dword:00000001`. И всё, теперь обновления ставятся без излишних проблем.
