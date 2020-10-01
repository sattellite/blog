---
tags: kolab, cyrus
noedit: true
---

# Просмотр SIEVE-правил пользователя в Cyrus

Продолжаю серию заметок по работе с Kolab GroupWare. Очень интересный продукт, плотно занимаюсь его изучением и настройкой уже в течении нескольких месяцев. Некоторые моменты работы с ним  решил записывать, так как нужно не часто, но в целом пригодится. И чтобы потом не вспоминать что там и как - конспектировать.
Все материалы можно найти по тегу [kolab](/tags/kolab/).

В качестве imap-сервера в Kolab используется [Cyrus](https://cyrusimap.org). Для обработки писем используется реализация системы фильтров, которая называется [SIEVE](https://cyrusimap.org/mediawiki/index.php/Cyrus_Sieve) и описана в [RFC 3028](https://tools.ietf.org/html/rfc3028). Очень странно, что по умолчанию для SIEVE не используется кодировка UTF-8. В связи с этим возникает проблема при использовании non-Latin символов, например, переместить письмо в директорию "Прочее", а она не обрабатывается и не может быть создана. Для этого необходимо добавить в /etc/imapd.conf опцию `sieve_utf8fileinto: 1` и все станет хорошо.

А для того, чтобы посмотреть/управлять SIEVE-правилами пользователя нужно воспользоваться утилитой `sieveshell`. Но есть особенность - она работает только в `plaintext` режиме. Для этого временно разрешаем использовать его в `cyrus-imapd` исправив в `/etc/imapd.conf` значение `allowplaintext` с `no` на `yes`.

```
~ # sieveshell -a cyrus-admin -u sattellite@domain.tld 127.0.0.1
connecting to 127.0.0.1
Please enter your password:
> list
roundcube
USER  <- active script
> get USER
# USER Management Script
#
# This script includes the various active sieve scripts
# it is AUTOMATICALLY GENERATED. DO NOT EDIT MANUALLY!
#
# For more information, see http://wiki.kolab.org/KEP:14#USER
#

require ["include"];
include :personal "roundcube";
> get roundcube
require ["copy","fileinto","vacation"];
# EDITOR Roundcube (Managesieve)
# EDITOR_VERSION 8.4
# rule:[strangers]
if address :is "from" "stranger@gmail.com"
{
	fileinto "От незнакомца";
}
# rule:[прочитано про sieve]
if header :contains "subject" "Managesieve"
{
	vacation :addresses "sattellite@domain.tld" :subject "Какой сиева?" "Задолбал фигню писать";
}
```

Всё что можно сделать с помощью `sieveshell` читаем в `man sieveshell`.

Главное после не забыть вернуть значение `allowplaintext` в `no`.

