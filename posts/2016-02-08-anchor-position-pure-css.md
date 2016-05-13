# Решение проблемы с "якорем" прячущимся за "шапкой"

Я порой удивляюсь тому, чем я занимаюсь, о чем пишу. Это не профильная для меня тематика, но решил поделиться, т.к. проблема возникла уже не в первый раз, но в первый раз нашел для нее красивое решение.

**Проблема**: При оформлении веб-страницы с фиксированной "шапкой", которая всегда(!) отображается в самом верху страницы и использовании ссылок-якорей получается так, что место, на которое ссылается "якорь", прижимается к самому верху страницы и скрывается под "шапкой". Описал путано, но надеюсь понятно.

Ранее подобные вещи решал с помощью JavaScript, это выглядит более качественно и надежно. Но вот сейчас столкнулся с тем, что на странице нет JS и подключать его только для этой цели нет смысла.

**Решение на чистом CSS**: Есть псевдо-элемент `:target`, который отвечает за оформление элемента, на который указывает "якорь". И решение сводится к тому, чтобы переместить заголовок ниже "шапки" с помощью `margin` или `padding`. И на последок можно добавить анимации :-)

Надо всего лишь добавить подобный блок в свой файл стилей:

```
:target {
  margin-top: 90px!important;
  color: #F44336;
  background-color: rgba(255, 235, 59, 0.35);
  transition: margin 0.5s ease-out, color 2s ease, background-color 1s ease;
}
```
