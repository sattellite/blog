---
tags: perl
noedit: true
---

# Использование Unicode в Perl

У любого, кто с Perl работал не так мало времени не один раз возникала ошбика упоминающая в себе `Wide character`.
Это все из-за того, что Perl с unicode плохо дружит и реализация поддержки самого unicode прилеплена с боку. Все это
оставляет желать только лучшего. Вот небольшой сборник рецептов для помощи, если вдруг возникла проблема.

    use utf8;

Часто решает проблему, но если не решило, то стоит отключить сразу же для предотвращения ошибок в следующих способах.

Переменная [среды выполенения](http://perldoc.perl.org/perlrun.html#PERL_UNICODE):

    export PERL_UNICODE=SDL

[Командная строка](http://perldoc.perl.org/perlrun.html):

    perl -CSDL -le 'print "\x{1815}"';

[binmode](http://perldoc.perl.org/functions/binmode.html):

    binmode(STDOUT, ":utf8");
    binmode(STDIN, ":encoding(utf8)");

[PerlIO](http://perldoc.perl.org/PerlIO.html):

    open my $fh, ">:utf8", $filename
        or die "could not open $filename: $!\n";

    open my $fh, "<:encoding(utf-8)", $filename
        or die "could not open $filename: $!\n";

 [open pragma](http://perldoc.perl.org/open.html):

    use open ":encoding(utf8)";
    use open IN => ":encoding(utf8)", OUT => ":utf8";
    # or
    use open qw/:std :utf8/;

Примеры приведены [отсюда](http://stackoverflow.com/questions/627661/how-can-i-output-utf-8-from-perl).

А еще полезно почитать [тут](http://habrahabr.ru/post/53578/).

