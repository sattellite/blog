# Подключение номера по PJSIP от Ростелекома

У меня нет доступа к кабинету управления ВАТС РТК, поэтому я не могу сказать как правильно сконфигурировать данные в кабинете, чтобы вызовы шли на один номер, который мы будет подключать к Asterisk.


## Регистрация номера в РТК

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
contact_user=rtk
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
qualify_frequency=30

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
send_rpid=yes
trust_id_inbound=yes
inband_progress=yes
force_rport=yes
rtp_symmetric=yes
rewrite_contact=yes
direct_media=no

...
```

Для прохождения исходящих вызовов в секции `endpoint` обязательно надо указать `from_user` и `from_domain`, чтобы их SBC пропустил `INVITE`.

### Обход NAT

Для ситуации, когда Asterisk находится за NAT в секции `endpoint` надо использовать опции `force_rport=yes`, `rtp_symmetric=yes`, `rewrite_contact=yes` и `direct_media=no`.

Если есть возможность на натирующем шлюзе установить статичный порт для Asterisk'а, то можно в секции `transport` прописать опции сетей, чтобы Asterisk самостоятельно указывал нужный IP.

```
[transport-udp]
type=transport
...
local_net=10.128.0.0/12
local_net=127.0.0.1/32
external_media_address=81.23.14.140
external_signaling_address=81.23.14.140
```

## Маршрутизация звонков в/из РТК

У виртуальной АТС Ростелекома есть проблема, которой много лет, про которую знают их инженеры и которую они не могут исправить - при входящем звонке АТС РТК присылает INVITE без указания кодека. Это, конечно, всё укладывается в спецификацию SIP/SDP, но на некоторых инсталляциях Atserisk'у срывает "башню". Заключается это в том, что Asterisk по умолчанию может переключиться на кодек slin, а АТС РТК шлёт только в кодеке alaw и дальнейшие пересогласования кодеков почему-то не решаюти эту проблему. Я ловил такую проблему на Asterisk версий 10, 12 и 14.

Для устранения проблемы можно использовать хак - запускать early media. Это позволит до ответа на вызов договориться о кодеке. Для этого предварительно в секции `endpoint` для РТК была включена опция `inband_progress`.


И пример как отправлять звонки в сторону РТК из `extensions.conf`:

```
...
[incoming]
exten => rtk,1,Ringing()
 same =>     n,Progress()
 same =>     n,Answer()
 same =>     n,Playback(hello-world)

[outgoing]
exten => _XXXXXX,1,Dial(PJSIP/${EXTEN}@rtk)

...
```
