---
tags: perl, mojolicious
noedit: true
---

# Подключение стилей и скриптов для специфичных страниц в Mojolicious

Иногда при оформлении определенных частей приложения требуется создать специфичный стиль. Конечно же, можно описать его в общем стиле если это пару строчек. Но если это огромное количество стилей, необходимых только тут, то таскать за зря на 99% страниц всего приложения за собой ненужных 2-4кб становится, наверное, не правильным. Даже не смотря на возможности современного интернета и web-серверов.

Я тут прикинул возможности шаблонизатора в `Mojolicious` и сделал следующую конструкцию, которая включается в главный *слой* приложения, например `main`, в заголовок `head`:

    % if ( my $css = stash 'css' ) {
    %   foreach my $element ( @$css ) {
    %     if ($element =~ m#^http|^//|\.css$#) {
    %=      stylesheet $element
    %     } else {
    %=      stylesheet begin
    %==       $element
    %       end
    %     }
    %   }
    % }

Суть этой конструкции проста: если вызывается родительский *слой* и при вызове определена опция `css`, то встроить этот стиль в страницу. Точно так же можно подключать `javascript`.

### Пример
В общей структуре приложения, выглядещей таким образом:

    app
    |--templates
      |--layouts
      | |--main.html.ep
      |--page1.html.ep
      |--page2.html.ep

В файл `main.html.ep` прописываем:

    <!DOCTYPE html>
    <html>
    <head>
      <title><%= title %></title>
        %= javascript "//code.jquery.com/jquery-2.1.0.min.js"

        % if ( my $js = stash 'js' ) {
        %   foreach my $element ( @$js ) {
        %     if ($element =~ m#^http|^//|\.js$#) {
        %=      javascript $element
        %     } else {
        %=      javascript begin
        %==       $element
        %       end
        %     }
        %   }
        % }

        % if ( my $css = stash 'css' ) {
        %   foreach my $element ( @$css ) {
        %     if ($element =~ m#^http|^//|\.css$#) {
        %=      stylesheet $element
        %     } else {
        %=      stylesheet begin
        %==       $element
        %       end
        %     }
        %   }
        % }
    </head>
    <body>
      %= include 'menu'
      <%= content %>
    </body>
    </html>

В `page1.html.ep`:

    % layout 'main',

    %= form_for url_for("user_show", user => $name) => (method => 'POST') => begin
    ...

В `page2.html.ep`:

    % layout 'main',
    % js => ['//code.jquery.com/ui/1.10.4/jquery-ui.min.js',
    % '$(function() {
    %    $( "#sortable" ).sortable();
    %    $( "#sortable" ).disableSelection();
    %  });'],
    % css => ['//code.jquery.com/ui/1.10.4/themes/smoothness/jquery-ui.css',
    %'#sortable { list-style-type: none; margin: 0; padding: 0; width: 40%; }
    % #sortable li { m  argin: 0 3px 3px 3px; padding: 4px; font-size: 13px; font-family: monospace;  height: 18px; }'];

    %= form_for url_for("dplan_show", dplan => $name) => (method => 'POST') => begin
    ...

В итоге для `page2`  помимо **jquery**, дополнительно будет подключен **jquery-ui** и стиль к нему.


