---
tags: git
noedit: true
---

# Скачивание определенной директории из git-репозитория

Нашёл решение для скачивания определенной директории из git-репозитория. Это перевод [ответа на StackOverflow](http://stackoverflow.com/questions/600079/is-there-any-way-to-clone-a-git-repositorys-sub-directory-only/13738951#13738951), который в свою очередь является хорошим примером к документации по [Sparse Checkout](https://git-scm.com/docs/git-read-tree#_sparse_checkout) в Git.

Начиная с версии 1.7.0 в Git появилась возможность указания какие пути в локальной копии репозитория должны синхронизироваться - sparse checkout. Для скачивания отдельной директории из удаленного репозитория необходимо проделать следующие шаги:

```shell
mkdir <reponame>
cd <reponame>
git init
git remote add -f origin <repoURL>
```
Создается пустой репозиторий и в него скачиваются все объекты из удаленного репозитория, но не применяются.

```shell
git config core.sparseCheckout true
```
Включается `sparse checkout`. B далее надо в файле `.git/info/sparse-checkout` определить все пути, которые необходимо синхронизировать:

```shell
echo "directory/one/" >> .git/info/sparse-checkout
echo "directory/two/" >> .git/info/sparse-checkout
```
И выполнить синхронизацию с удаленным репозиторием:

```shell
git pull origin master
```

Более подробно описано [здесь](http://jasonkarns.com/blog/subdirectory-checkouts-with-git-sparse-checkout/)



