# Создание функций INET_ATON и INET_NTOA для SQLite3 в Perl

В SQLite нет функций, которые были бы аналогичны функциям MySQL `INET_ATON` и `INET_NTOA`. Суть этих функций превращать IP-адрес в число и обратно. Такие функции крайне полезны для того, чтобы хранить адреса в колонках с типом `INT` вместо `VARCHAR`.

Для этого можно добавить необходимые функции в SQLite. Так как я пользуюсь в основном Perl'ом, то с его помощью и будем добавлять их.

В Perl драйвер для SQLite3 имеет функцию `sqlite_create_function`, функционал которой подробно описан [тут](https://metacpan.org/pod/DBD::SQLite#dbh-sqlite_create_function-name-argc-code_ref).

Функции `inet_ntoa` и `inet_aton` есть в модуле [Socket](https://metacpan.org/pod/Socket). Их и будем использовать.

Пример кода для добавления этих функций:

```perl
use DBI;
use Socket qw(inet_aton inet_ntoa);

my $db = DBI->connect('dbi:SQLite:dbname=sqlite_database_file.db');
$db->sqlite_create_function( 'inet_aton', 1, sub{ return unpack('N', inet_aton(shift)) } );
$db->sqlite_create_function( 'inet_ntoa', 1, sub{ return inet_ntoa( pack('N', shift) ) } );

my ($num, $ip) = $db->selectrow_array("SELECT inet_aton('192.168.0.1'), inet_ntoa(3232235521)");
# $num = 3232235521
# $ip  = '192.168.0.1'
```

