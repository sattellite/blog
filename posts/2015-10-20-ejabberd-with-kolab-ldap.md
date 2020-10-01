---
tags: kolab, ejabberd, ldap
noedit: true
---

# Ejabberd и LDAP

На тему как сделать авторизацию в ejabberd через LDAP написано очень много материала. Эта небольшая заметка возникла из-за необходимости подключить [ejabberd](https://ejabberd.im) к сервису групповой работы [Kolab](https://kolab.org), а также подробнее разобраться во всем написанном ~~и чуть подробнее чем обычно об этом пишут~~. А ещё пойдет в качестве хорошего способа запомнить всё прочитанное.

Основными источником информации является официальная документация. Она существует в двух вариантах: [красивенький](http://docs.ejabberd.im/admin/guide/configuration/#ldap) и [обычный](https://www.process-one.net/docs/ejabberd/guide_en.html#ldap). Рекомендую обычный, т.к. в нём все дополнительные вещи не заменяются escape-последовательностями.

## Подключение к LDAP

Для того, чтобы авторизовываться через LDAP необходимо проставить значение `auth_method` в `ldap`. Указать адрес LDAP-сервера, его порт, способ шифрования, а также основного пользователя под которым будет проходить аутентификация.

```yml
auth_method: ldap
ldap_servers:
  - "127.0.0.1"
ldap_port: 389
ldap_encrypt: none
ldap_rootdn: "uid=ejabberd-service,ou=Special Users,dc=domain,dc=tld"
ldap_password: "***"
```

Далее надо указать в каком виде хранятся логины пользователей в LDAP и фильтр, чтобы их можно было сопоставить:

```yml
ldap_uids:
  - "mail" : "%u@domain.tld"
  - "alias": "%u@domain.tld"
  - "uid"  : "%u"
ldap_filter: "(objectClass=inetOrgPerson)"
```

## Общий ростер из LDAP

А вот дальше начинается магия и волшебство модуля `mod_shared_roster_ldap`. Этот модуль практически всегда вызывает много вопросов и непонимания того как оно всё работает. Этот модуль несколько раз исключали, а потом снова добавляли в официальную сборку.

Пойдем по [документации](https://www.process-one.net/docs/ejabberd/guide_en.html#modsharedrosterldap):

1. Первым делом используется `ldap_rfilter` (roster filter), который возвращает все общие группы;
1. Далее к результату от `ldap_rfilter` применяется `ldap_ufilter`(user filter) для того, чтобы сформировать список пользователей. Для этого ему в помощь используются `ldap_userdesc` и `ldap_useruid` для определения выводимых полей пользователя;
1. Дополнительно к результату `ldap_rfilter` применяется `ldap_gfilter`(group filter), который определяет названия групп и принадлежность пользователя к какой-либо группе. Для описания группы дополнительно используются `ldap_groupattr` и `ldap_groupdesc`. А для определения принадлежности пользователя к группе дополнительно используются `ldap_memberattr`,`ldap_memberattr_format` и `ldap_memberattr_format_re`;
1. К `ldap_gfilter` и `ldap_ufilter` дополнительно применяется с приставкой AND `ldap_filter`.

Подробности как в итоге составляются фильтры можно найти в документации. Но как по мне - там ничего не понятно.

Для того, чтобы достать всех пользователей из Kolab'а и разнести их по группам составляется такой конфиг для `ldap_shared_roster_ldap`:

```yml
mod_shared_roster_ldap:
    ldap_base: "dc=domain,dc=tld"
    ldap_rfilter: "(&(objectclass=groupofuniquenames)(uniqueMember=uid=*,ou=People,dc=domain,dc=tld))"
    ldap_ufilter: "(&(objectClass=kolabinetorgperson)(uid=%u))"
    ldap_filter: "(objectClass=top)"
    ldap_memberattr: "uniqueMember"
    ldap_memberattr_format: "uid=%u,ou=People,dc=domain,dc=tld"
    ldap_userdesc: "cn"
    ldap_useruid: "uid"
    ldap_groupattr: "cn"
    ldap_groupdesc: "cn"
    ldap_user_cache_validity: 60
    ldap_group_cache_validity: 60
```

***Немного человеческого описания конфига:*** Базой для поиска выбирается корень, так как в Kolab пользователи и группы разнесены по разным "веткам"(ou). В `ou=People` хранится полная информация о каждом пользователе, а в `ou=Groups` хранится информация о группах, в которых пользователи указаны в аттрибуте `uniqueMember` в формате полного пути.

Сначала выбираются все группы в которых есть пользователи фильтром `(&(objectclass=groupofuniquenames)(uniqueMember=uid=*,ou=People,dc=domain,dc=tld))`.
Затем пользователи фильтруются фильтром `(&(objectClass=kolabinetorgperson)(uid=%u))` и задаются дополнительные атрибуты для пользователей `ldap_userdesc` и `ldap_useruid`, которые указывают, что надо использовать `cn` в качестве имени контакта и `uid` в качестве его jid.
Далее сам `ldap_gfilter` не назначается для того чтобы использовать полную информацию из `ldap_rfilter`. Но дополнительно назначаются параметры описания группы `ldap_groupattr` и `ldap_groupdesc` с значением `cn`, в котором находится название группы. И указывается каким способом извлечь пользователей из групп с помощью `ldap_memberattr` и `ldap_memberattr_format`. Пользователь находится в `uniqueMember` в формате `uid=%u,ou=People,dc=domain,dc=tld`.

Дополнительно можно указать время кеширования по пользователям и группам в секундах с помощью параметров `ldap_user_cache_validity` и `ldap_group_cache_validity`.

## Карточки контактов из LDAP

Получение корточек контактов из LDAP не представляет проблемы. Единственное что надо учесть при маппинге полей `vcard` - данные, указанные в `ldap_vcard_map` это те данные которые будут отображаться. Так что если надо изменить только некоторые аттрибуты, то надо всё равно составлять полный список полей.

Конфиг `mod_vcard_ldap` для Kolab:

```yml
mod_vcard_ldap:
    ldap_base: "ou=People,dc=domain,dc=tld"
    ldap_filter: "(objectClass=kolabinetorgperson)"
    ldap_vcard_map:
      "NICKNAME": {"%u": []}
      "FN": {"%s":["cn"]}
      "LAST": {"%s": ["sn"]}
      "FIRST": {"%s": ["givenName"]}
      "MIDDLE": {"%s": ["initials"]}
      "ORGNAME": {"%s": ["o"]}
      "ORGUNIT": {"": []}
      "CTRY": {"%s": ["c"]}
      "LOCALITY": {"%s": ["l"]}
      "STREET": {"%s": ["street"]}
      "REGION": {"%s": ["st"]}
      "PCODE": {"%s": ["postalCode"]}
      "TITLE": {"%s": ["title"]}
      "URL": {"%s": ["labeleduri"]}
      "DESC": {"%s": ["description"]}
      "TEL": {"%s": ["telephoneNumber"]}
      "EMAIL": {"%s": ["mail"]}
      "BDAY": {"%s": ["birthDay"]}
      "ROLE": {"%s": ["employeeType"]}
      "PHOTO": {"%s": ["jpegPhoto"]}
```

У самого Kolab по умолчанию в схеме у пользователя нет атрибута `jpegPhoto`, но его можно добавить :-)

## Итоговый конфиг ejabberd для работы с LDAP

Дополнительно стоит обратить внимание на то, что регистрация пользователей запрещена, т.к. этот функционал должен быть только у администраторов Kolab.

```yml
loglevel: 4
log_rotate_size: 10485760
log_rotate_date: ""
log_rotate_count: 1
log_rate_limit: 100
hosts:
  - "domain.tld"
listen:
  -
    port: 5222
    module: ejabberd_c2s
    max_stanza_size: 65536
    shaper: c2s_shaper
    access: c2s
    starttls: true
    certfile: "/etc/ssl/ejabberd.pem"
  -
    port: 5269
    module: ejabberd_s2s_in
    max_stanza_size: 131072
    shaper: s2s_shaper
  -
    port: 5280
    module: ejabberd_http
    request_handlers:
      "/websocket": ejabberd_http_ws
    web_admin: true
    http_poll: false
    http_bind: true
    captcha: false
s2s_access: all
auth_method: ldap
ldap_servers:
  - "127.0.0.1"
ldap_encrypt: none
ldap_port: 389
ldap_rootdn: "uid=ejabberd-service,ou=Special Users,dc=domain,dc=tld"
ldap_password: "***"
ldap_base: "ou=People,dc=domain,dc=tld"
ldap_uids:
  "uid"  : "%u"
ldap_filter: "(objectClass=inetOrgPerson)"
shaper:
  normal: 1000
  fast: 50000
max_fsm_queue: 1000
acl:
  admin:
    user:
      - "admin": "domain.tld"
  local:
    user_regexp: ""
  loopback:
    ip:
      - "127.0.0.0/8"
access:
  max_user_sessions:
    all: 10
  max_user_offline_messages:
    admin: 5000
    all: 100
  local:
    local: allow
  c2s:
    blocked: deny
    all: allow
  c2s_shaper:
    admin: none
    all: normal
  s2s_shaper:
    all: fast
  announce:
    admin: allow
  configure:
    admin: allow
  muc_admin:
    admin: allow
  muc_create:
    local: allow
  muc:
    all: allow
  pubsub_createnode:
    local: allow
  register:
    all: deny
  trusted_network:
    loopback: allow
language: "ru"
modules:
  mod_adhoc: []
  mod_announce:
    access: announce
  mod_blocking: []
  mod_caps: []
  mod_carboncopy: []
  mod_configure: []
  mod_disco: []
  mod_http_bind: []
  mod_last: []
  mod_mam:
    default: always
    cache_size: 1000
    cache_life_time: 86400
  mod_muc:
    host: "conference.@HOST@"
    access: muc
    access_create: muc
    access_persistent: muc
    access_admin: muc_admin
    history_size: 20
    max_user_conferences: 100
    min_message_interval: 0.5
    min_presence_interval: 5
    max_users_presence: 50
    max_users: 200
    default_room_options:
      allow_change_subj: true
      allow_private_messages: false
      allow_private_messages_from_visitors: nobody
      allow_query_users: false
      allow_user_invites: true
      allow_visitor_nickchange: false
      allow_visitor_status: false
      anonymous: false
      mam: true
      members_by_default: true
      persistent: false
      public: true
      public_list: false
  mod_muc_admin: []
  mod_offline:
    access_max_user_messages: max_user_offline_messages
  mod_ping: []
  mod_pres_counter:
    count: 5
    interval: 60
  mod_privacy: []
  mod_private: []
  mod_proxy65:
    access: all
    shaper: none
  mod_pubsub:
    access_createnode: pubsub_createnode
    ignore_pep_from_offline: true
    last_item_cache: false
    plugins:
      - "flat"
      - "hometree"
      - "pep"
  mod_time: []
  mod_version: []
  mod_roster: []
  mod_shared_roster_ldap:
    ldap_base: "dc=domain,dc=tld"
    ldap_rfilter: "(&(objectclass=groupofuniquenames)(uniqueMember=uid=*,ou=People,dc=domain,dc=tld))"
    ldap_ufilter: "(&(objectClass=kolabinetorgperson)(uid=%u))"
    ldap_filter: "(objectClass=top)"
    ldap_memberattr: "uniqueMember"
    ldap_memberattr_format: "uid=%u,ou=People,dc=domain,dc=tld"
    ldap_userdesc: "cn"
    ldap_useruid: "uid"
    ldap_groupattr: "cn"
    ldap_groupdesc: "cn"
    ldap_user_cache_validity: 60
    ldap_group_cache_validity: 60
  mod_vcard_ldap:
    ldap_base: "ou=People,dc=domain,dc=tld"
    ldap_filter: "(objectClass=kolabinetorgperson)"
    ldap_vcard_map:
      "NICKNAME": {"%u": []}
      "FN": {"%s":["cn"]}
      "LAST": {"%s": ["sn"]}
      "FIRST": {"%s": ["givenName"]}
      "MIDDLE": {"%s": ["initials"]}
      "ORGNAME": {"%s": ["o"]}
      "ORGUNIT": {"": []}
      "CTRY": {"%s": ["c"]}
      "LOCALITY": {"%s": ["l"]}
      "STREET": {"%s": ["street"]}
      "REGION": {"%s": ["st"]}
      "PCODE": {"%s": ["postalCode"]}
      "TITLE": {"%s": ["title"]}
      "URL": {"%s": ["labeleduri"]}
      "DESC": {"%s": ["description"]}
      "TEL": {"%s": ["telephoneNumber"]}
      "EMAIL": {"%s": ["mail"]}
      "BDAY": {"%s": ["birthDay"]}
      "ROLE": {"%s": ["employeeType"]}
      "PHOTO": {"%s": ["jpegPhoto"]}
  mod_vcard_xupdate: []
  mod_carboncopy: []
allow_contrib_modules: true
```

