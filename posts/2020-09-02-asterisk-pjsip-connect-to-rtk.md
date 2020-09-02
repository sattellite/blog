# Подключение номера по PJSIP от Ростелекома

У меня нет доступа к кабинету управления ВАТС РТК, поэтому я не могу сказать как правильно сконфигурировать данные в кабинете, чтобы вызовы шли на один номер, который мы будет подключать к Asterisk.

Конфигурация `pjsip.conf`:

```
...

[transport-udp]
type=transport
protocol=udp
bind=0.0.0.0:5060

[rtk]
type=registration
transport=transport-udp
outbound_auth=rtk
client_uri=sip:RTKusername@RTKdomain.rt.ru
server_uri=sip:RTKdomain.rt.ru
retry_interval=60
expiration=120
fatal_retry_interval=5
max_retries=10000
forbidden_retry_interval=1
contact_user=RTKusername
line=yes
endpoint=rtk

[rtk]
type=auth
auth_type=userpass
username=RTKusername
password=RTKpassword

[rtk]
type=aor
contact=sip:RTKusername@RTKdomain.rt.ru

[rtk]
type=endpoint
transport=transport-udp
context=incoming
disallow=all
allow=alaw
from_user=RTKusername
from_domain=RTKdomain.rt.ru
outbound_auth=rtk
aors=rtk
rtp_symmetric=yes
rewrite_contact=yes
force_rport=yes
send_rpid=yes
direct_media=no

...
```

Для прохождения исходящих вызовов в секции `endpoint` обязательно надо указать `from_user` и `from_domain`, чтобы их SBC пропустил звонок.

И пример как отправлять звонки в сторону РТК из `extensions.conf`:

```
...

[common]
exten => _XXXX,1,Dial(PJSIP/${EXTEN},,Ttfg)

[operators]
include => common

exten => _XXXXXX,1,Dial(PJSIP/${EXTEN}@rtk)

...
```


// TODO: Описать как обходить проблему с инвайтом от РТК без указания кодека.
