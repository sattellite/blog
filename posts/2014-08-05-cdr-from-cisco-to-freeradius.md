# Настройка FreeRadius для логирования CDR от Cisco VoIP

Есть Cisco для VoIP, которая отправляет вызовы на множество операторов. Звонок проходит через dial-peer'ов по указанным в них префиксам и в случае факапа одного из внешних операторов узнать через кого из них ушёл вызов не представляется возможным по обычным логам.  Для полноценного логирования всех звонков проходящих через Cisco  с множеством dial-peer'ов необходимо использовать RADIUS, чтобы точно знать через какого dial-peer'а вышел звонок.

### Cisco

На Cisco необходимо включить отправку accounting сообщений в RADIUS. Авторизацию и аутентификацию использовать не будем.

#### Настройка как клиента RADIUS'а

    enable
    configure terminal
    aaa new-model
    aaa accounting connection h323 start-stop radius
    radius-server host <ip address> acct-port 1813
    radius-server key <password>
    exit

#### Настройка отправки accounting сообщений

    enable
    configure terminal
    radius-server vsa send accounting
    gw-accounting aaa
    acct-template callhistory-detail
    end

Подробнее о настройке Cisco для отправки сообщений в RADIUS можно почитать [тут](http://www.cisco.com/c/en/us/td/docs/ios/voice/cdr/developer/guide/cdrdev/cdradius.html#pgfId-1057687).

### RADIUS

RADIUS сервер будет поднят на `Debian Linux` и все полученные данные будут складываться в базу данных PostgreSQL. Смысла в Start пакетах нет, т.к. абсолютно полная информация о завершившихся вызовах содержится в Stop пакетах. Поэтому обрабатывать будем только их.

Для начала установим freerardius:

    sudo apt-get install -y freeradius freeradius-postgresql

#### radiusd.conf

Теперь приступим к настройке. В `radiusd.conf`:

- оставляем только одну секцию `listen`, в которая принимает accounting
- `$INCLUDE clients.conf`
- в секции modules включаем sql.conf: `$INCLUDE sql.conf`

Полное содержимое файла:

    prefix = /usr
    exec_prefix = /usr
    sysconfdir = /etc
    localstatedir = /var
    sbindir = ${exec_prefix}/sbin
    logdir = /var/log/freeradius
    raddbdir = /etc/freeradius
    radacctdir = ${logdir}/radacct
    name = freeradius
    confdir = ${raddbdir}
    run_dir = ${localstatedir}/run/${name}
    db_dir = ${raddbdir}
    libdir = /usr/lib/freeradius
    pidfile = ${run_dir}/${name}.pid
    user = freerad
    group = freerad
    max_request_time = 30
    cleanup_delay = 5
    max_requests = 1024
    listen {
            type = acct
            ipaddr = *
            port = 0
    }
    hostname_lookups = no
    allow_core_dumps = no
    regular_expressions     = yes
    extended_expressions    = yes
    log {
            destination = files
            file = ${logdir}/radius.log
            syslog_facility = daemon
            stripped_names = no
            auth = no
            auth_badpass = no
            auth_goodpass = no
    }
    checkrad = ${sbindir}/checkrad
    security {
        max_attributes = 200
        reject_delay = 1
        status_server = yes
    }
    proxy_requests  = yes
    $INCLUDE proxy.conf
    $INCLUDE clients.conf
    thread pool {
            start_servers = 5
            max_servers = 32
            min_spare_servers = 3
            max_spare_servers = 10
            max_requests_per_server = 0
    }
    modules {
            $INCLUDE ${confdir}/modules/
            $INCLUDE eap.conf
            $INCLUDE sql.conf
    }
    instantiate {
            exec
            expr
            expiration
            logintime
    }
    $INCLUDE policy.conf
    $INCLUDE sites-enabled/

#### clients.conf

В `clients.conf` добавляем нашу Cisco:

    client <ipaddr> {
        ipaddr          = <ipaddr>
        secret          = <password>
        shortname       = Cisco-VoIP
    }

В `secret` указывается тот пароль, что был ранее указан при конфигурации Cisco.

#### sql.conf

В `sql.conf` указываем сервер, логин, пароль, таблицу базы данных, в которую будем складывать полученный CDR. Также из таблиц оставляем только ту, в которую будут складывать Stop пакеты. Полное содержимое файла:

    sql {
        database = "postgresql"
        driver = "rlm_sql_${database}"
        server = "aaa.bbb.ccc.ddd"
        login = "radius"
        password = "sup3rs3cr3tpa$$w0rd"
        radius_db = "cdr"
        acct_table2 = "cdr"
        deletestalesessions = yes
        sqltrace = no
        sqltracefile = ${logdir}/sqltrace.sql
        num_sql_socks = 5
        connect_failure_retry_delay = 60
        lifetime = 0
        max_queries = 0
        $INCLUDE sql/${database}/dialup.conf
    }

#### dialup.conf

Перейдем к подключаемому `dialup.conf`. В этом файле оставим только одну секцию, которая отвечает за обработку Stop пакетов. Полное содержимое файла:

    sql_user_name = "%{User-Name}"

    accounting_stop_query = "INSERT INTO ${acct_table2} \
      (src,dst,start,connect,stop,duration,cause,localip,remoteip,mediaip) \
      VALUES('%{User-Name}', '%{Called-Station-Id}', \
      to_timestamp(overlay('%{h323-setup-time}' placing '' from 13 for 11), 'HH24:MI:SS.MS Mon DD YYYY')::timestamp, \
      to_timestamp(overlay('%{h323-connect-time}' placing '' from 13 for 11), 'HH24:MI:SS.MS Mon DD YYYY')::timestamp, \
      to_timestamp(overlay('%{h323-disconnect-time}' placing '' from 13 for 11), 'HH24:MI:SS.MS Mon DD YYYY')::timestamp, \
      '%{Acct-Session-Time}'::BIGINT, \
      '%{h323-disconnect-cause}', \
      NULLIF('%{NAS-IP-Address}', '')::inet, \
      NULLIF('%{h323-remote-address}', '')::inet, \
      NULLIF('%{remote-media-address}', '')::inet)"

Здесь составляется собственный запрос к БД, чтобы нужным мне способом заполнить таблицу. Подробнее о [функциях форматирования](http://www.postgresql.org/docs/9.3/static/functions-formatting.html) и о [функциях над данными](http://www.postgresql.org/docs/9.3/static/functions-string.html) в PostgreSQL.

#### preprocess

Теперь необходимо в файле `module/preprocess` в секции `preprocess` указать, что необходимо преобразовывать VSA:

    with_cisco_vsa_hack = yes

### PostgreSQL

```sql
CREATE TABLE cdr (
  id serial NOT NULL,
  src character varying(30) DEFAULT ''::character varying,
  dst character varying(30) DEFAULT ''::character varying,
  start timestamp without time zone DEFAULT '1970-01-01 00:00:00'::timestamp without time zone,
  "connect" timestamp without time zone DEFAULT '1970-01-01 00:00:00'::timestamp without time zone,
  stop timestamp without time zone DEFAULT '1970-01-01 00:00:00'::timestamp without time zone,
  duration bigint DEFAULT 0,
  cause integer,
  localip inet DEFAULT '0.0.0.0'::inet,
  remoteip inet DEFAULT '0.0.0.0'::inet,
  mediaip inet DEFAULT '0.0.0.0'::inet,
  CONSTRAINT cdr_pkey PRIMARY KEY (id),
  CONSTRAINT cdr_src_dst_start_key UNIQUE (src, dst, start)
);
ALTER TABLE cdr OWNER TO radius;
COMMENT ON TABLE cdr IS 'CDR через Radius';

CREATE INDEX cdr_cause ON cdr USING btree (cause);
CREATE INDEX cdr_dst ON cdr USING btree (dst);
CREATE INDEX cdr_dst_like ON cdr USING btree (dst varchar_pattern_ops);
CREATE INDEX cdr_duration ON cdr USING btree (duration);
CREATE INDEX cdr_localip ON cdr USING btree (localip);
CREATE INDEX cdr_mediaip ON cdr USING btree (mediaip);
CREATE INDEX cdr_remoteip ON cdr USING btree (remoteip);
CREATE INDEX cdr_src ON cdr USING btree (src);
CREATE INDEX cdr_src_like ON cdr USING btree (src varchar_pattern_ops);
```

Теперь у нас все готово. Можно сказать `freeradius` перезапуститься и наблюдать как увеличивается количество записей в таблице `cdr`.

