# Базовая настройка Ruby Gem в Fedora 22

*Эта запись максимум претендует на мини-заметку.*

Всё происходит на примере пакета [`tmuxinator`](https://github.com/tmuxinator/tmuxinator). При попытке его установить получил следующую ошибку.

```
$ gem install tmuxinator
Ignoring json-1.8.2 because its extensions are not built.  Try: gem pristine json --version 1.8.2
ERROR:  Loading command: install (LoadError)
	no such file to load -- jopenssl/load
ERROR:  While executing gem ... (NoMethodError)
    undefined method `invoke_with_build_args' for nil:NilClass
```

Для того, чтобы в Fedora 22 что-либо установить с помощью утилиты `gem` необходимо в систему доустановить несколько пакетов:

```
sudo dnf install rubygems rubygem-bundler ruby-devel
```

После этого `gem` заработал и не ругается ошибками:

```
$ gem install tmuxinator
Fetching: tmuxinator-0.6.11.gem (100%)
...
Successfully installed tmuxinator-0.6.11
Parsing documentation for tmuxinator-0.6.11
Installing ri documentation for tmuxinator-0.6.11
Done installing documentation for tmuxinator after 0 seconds
1 gem installed
```

