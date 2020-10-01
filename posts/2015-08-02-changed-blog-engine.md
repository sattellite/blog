---
tags: nodejs
noedit: true
---

# Переезд блога на движок Ghost

Больше года мой блог работал на самописаном движке [Mojo::Twist](https://github.com/sattellite/mojo-twist), который был форкнут из движка [Twist](https://github.com/vti/twist) (он ранее обслуживал мой блог). Свой движок я как-то бурно начал, но вскоре кончилось время и переписать админку до вменяемого состояния не осталось ни времени, ни желания. Пару недель назад задумался перевести блог на [Ghost](https://ghost.org/).

<blockquote class="twitter-tweet" lang="ru"><p lang="ru" dir="ltr">Я вот тут задумываюсь перевести свой блог на <a href="https://twitter.com/hashtag/ghost?src=hash">#ghost</a>, вместо самописного движка.</p>&mdash; Angry Unicorn (@_sattellite) <a href="https://twitter.com/_sattellite/status/621970691001290752">17 июля 2015</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

Для этого мне необходимо было сделать структуру ссылок на нём крайне похожей на старый движок, во-первых, чтобы поисковики вели куда надо, а не на страницу 404 и, во-вторых, чтобы не потерять комментарии в Disqus. А по ходу дела я ещё и модифицировал одну из тем оформления для этого движка.

##### Установка на FreeBSD
Первым делом пришлось понять как на FreeBSD запускать продукт написанный на NodeJS. Была куча сомнений, но, как ни странно, это совсем не вызвало проблем. Установка `nodejs` и `npm` на FreBSD проста и хорошо описана в [документации](https://github.com/TryGhost/Ghost/wiki/HOWTO:FreeBSD):

```sh
cd /usr/ports/www/node
make install clean
cd /usr/ports/www/npm
make install clean
cd /tmp
fetch https://github.com/TryGhost/Ghost/archive/<X.Y.Z>.tar.gz
mkdir /usr/local/www/ghost
tar xzf master.tar.gz --strip-components=1 -C /usr/local/www/ghost
rm master.tar.gz
cd /usr/local/www/ghost
CXX=c++ npm install sqlite3 --sqlite=/usr/local
npm install --production
```

Далее правится конфигурация `config.js` и запускается блог `npm start --production`. Вроде всё работает.


##### Автоматизация запуска на FreeBSD
Для автоматизации запуска есть удобная утилита [forever](https://github.com/foreverjs/forever).

```
npm install -g forever
```

А как с её помощью создать rc-скрипт хорошо описано в [статье на Хабре](http://habrahabr.ru/post/137857/).

```shell
#!/bin/sh

# PROVIDE: blog
# REQUIRE: NETWORKING SERVERS DAEMON
# BEFORE:  LOGIN
# KEYWORD: shutdown

. /etc/rc.subr

name="blog"
forever="/usr/local/bin/node /usr/local/bin/forever"
workdir="/usr/local/www/ghost"
script="index.js"

rcvar=`set_rcvar`

start_cmd="start"
stop_cmd="stop"
restart_cmd="restart"

load_rc_config $name
eval "${rcvar}=\${${rcvar}:-'NO'}"

start()
{
  USER=root
  PATH=/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/sbin:/usr/local/bin:/root/bin
  PWD=/root
  HOME=/root
  export NODE_ENV=production
  ${forever} start -a -l /var/log/forever.log -o /dev/null -e ${workdir}/node_err.log --workingDir ${workdir} ${workdir}/${script}
}

stop()
{
  ${forever} stop ${workdir}/${script}
}

restart()
{
  ${forever} restart ${workdir}/${script}
}

run_rc_command "$1"
```

Не забыть в `/etc/rc.conf` добавить строку

```
blog_enable="YES"
```

##### Конфигурация Nginx
Привожу пример конфига `nginx` без объяснений:

```
upstream blog {
	server	127.0.0.1:2368;
}

server {
	listen		80;
	server_name	b.sattellite.me;

	location / {
		rewrite	^(.*)$ https://b.sattellite.me$1 permanent;
	}
}

server {
	listen		443 ssl spdy;
	server_name	b.sattellite.me;

	access_log	/var/log/nginx/$host.access_log main;

	ssl							on;
	ssl_session_cache			shared:SSL:10m;
	ssl_session_timeout			5m;
	ssl_prefer_server_ciphers	on;
	ssl_stapling				on;
	ssl_stapling_verify			on;
	ssl_trusted_certificate		/usr/local/etc/nginx/ssl/ca-certs.pem;
	ssl_certificate				/usr/local/etc/nginx/ssl/ssl.cert;
	ssl_certificate_key			/usr/local/etc/nginx/ssl/ssl.key;
	ssl_dhparam					/usr/local/etc/nginx/ssl/dhparam.pem;
	ssl_protocols				TLSv1.2 TLSv1.1 TLSv1;
	ssl_ciphers					'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';

	proxy_set_header	X-Real-IP $remote_addr;
	proxy_set_header	X-Forwarded-For $proxy_add_x_forwarded_for;
	proxy_set_header	X-Forwarded-Proto $scheme;
	proxy_set_header	Host $http_host;

	location ~ ^/(?:ghost) {
		expires		0;
		add_header	Cache-Control "no-cache, private, no-store, must-revalidate, max-stale=0, post-check=0, pre-check=0";
		proxy_pass	http://blog;
	}

	location ~ ^/(?:p/) {
		expires		0;
		add_header	Cache-Control "no-cache, private, no-store, must-revalidate, max-stale=0, post-check=0, pre-check=0";
		proxy_pass	http://blog;
	}

	location / {
		proxy_ignore_headers	"Set-Cookie";
		proxy_hide_header		"Set-Cookie";
		proxy_ignore_headers	"Cache-Control";
		proxy_hide_header		"Cache-Control";
		proxy_hide_header		"Etag";
		expires					30m;
		proxy_pass				http://blog;
	}

	location ~* \.(?:ico|css|js|gif|jpe?g|png|ttf|woff2?)$ {
		access_log	off;
		expires		30d;
		add_header	Pragma public;
		add_header	Cache-Control "public, mustrevalidate, proxy-revalidate";
		proxy_pass	http://blog;
	}

	location = /robots.txt {
		access_log		off;
		log_not_found	off;
	}

	location = /favicon.ico {
		access_log		off;
		log_not_found	off;
	}

	# Для совместимости со старым движком
	location /article {
		rewrite	^/article/(.*)$ $scheme://$host/$1 permanent;
	}

	location /rss.rss {
		rewrite	^(?:.*)$ $scheme://$host/rss/ permanent;
	}
}
```

##### Замена ссылок под необходимый формат

Мне необходимо было, чтобы ссылки на статьи были в формате `/year/month/slug` и теги были по ссылке `/tags`. С помощью google нашёл достаточно информации, но в концентрированном виде оставлю её тут.

Для замены формата ссылок статей необходимо в настройках движка на главной странице включить `Include the date in your post URLs`. После из корневой директории подключиться к SQLite базе и поправить одну опцию:

```
sqlite3 ./content/data/ghost.db
SQLite version 3.8.10.2 2015-05-20 18:17:19
Enter ".help" for usage hints.
sqlite> update settings set value = '/:year/:month/:slug/' where key = 'permalinks';
```
Для оценки результата перезапустить блог.

Давно уже открыт тред, в котором предлагается внести изменения в движок, которые помогли бы пользователям самостоятельно менять формат ссылок и даже уже внедрён функционал для этого, но это не работает. Я внёс в БД изменения, но они эффекта не возымели.

*uuid был сгенерирован утилитой, которая находится внутри сорцов движка*
```sql
insert into settings(uuid,key,value,type,created_at,created_by,updated_at,updated_by) values('7348788e-b932-4d38-bf06-e4ed28591044', 'routeKeywords', '{"tag":"tags", "author":"author", "page":"page", "preview":"p", "private":"private"}','blog', 1438524432000, 1, 1438524432000, 1);
```

Так делать не стоит, на данный момент (версия 0.6.4) это совсем бесполезно. Так что исправим исходники. В файле `core/server/config/index.js` найти функцию-прототип `ConfigManager.prototype.set` и в ней объект `routeKeywords`, а в нём уже необходимый путь `tag` заменить с `tag` на `tags`. Можно перезапускать блог и проверять все свои изменения.

##### Тема оформления
Родная оформление красивое, но оно у всех, а хотелось своего и чуточку удобнее. Была найдена достаточно популярная тема оформления - [Magnum](https://github.com/durgesh-priyaranjan/magnum/commits/master). Она не обновляется уже 2 года, а те редкие коммиты, которые моложе этого срока - простое изменение опечаток. Решил тему немного подогнать под ту тему, что была на старом движке. Подробностей не будет, но в ходе процесса были исправлены шрифты, цвета, вид на мобильных устройствах. Тему решил переименовать и назвал [Blognum](https://github.com/sattellite/blognum). А на последок нарисована новая картинка с перловым кодом для этого блога. Как всё выглядит можно оценить на этом сайте.

