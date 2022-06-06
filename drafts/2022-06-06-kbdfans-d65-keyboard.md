---
tags: keyboard, qmk, via
---

# Клавиатура KBDfans D65

Несколько месяцев назад я стал обладателем клавиатуры [KBDfans D65](https://kbdfans.com/products/icd65-mechanical-keyboard-kit).
Я долго выбирал какую клавиатуру я бы хотел себе приобрести и в итоге остановился на этой. Я ее использую как основную
клавиатуру

Приобретал все на AliExpress:
- [Корпус цвета Violet Purple](https://aliexpress.com/item/1005002300724821.html)
- Свитчи Gateron Zealios V2:
  + Сначала взял на [62 грамма](https://aliexpress.com/item/1005002050402909.html?sku_id=12000018558464849)
  + Позже заменил на [67 грамм](https://aliexpress.com/item/1005002050402909.html?sku_id=12000018558471873)
- Клавиши [EnjoyPBT Milky&Purple Doubleshot](https://aliexpress.com/item/1005003504374566.html)

Взял еще улучшателей:
- Пленки для свитчей [KBDfans Transparent](https://aliexpress.com/item/1005002147892603.html?sku_id=12000018890780033)
- Масло [Keychron Klube 105](https://aliexpress.com/item/1005003226619341.html?sku_id=12000024746449222)
- Смазка [Permatex 81150](https://aliexpress.com/item/1005003318129422.html)


## Сборка

## Конфигурация

Управлять конфигурацией клавиатуры можно с помощью утилиты [VIA](https://www.caniusevia.com)

Поймал проблему, что при установленном переключении языков на Caps Lock в MacOS происходит переключение языка и состояния 
Caps Lock. То есть переключаясь с русского на английский я получу то, что все набираемые символы будут в верхнем регистре.

Оказалось, что эта проблема связана с таймерами, которые выдает родная прошивка VIA -
[описание проблемы](https://github.com/the-via/keyboards/issues/551)

Стандартная прошивка QMK не поддерживает управление с помощью VIA. Для этого надо внести изменения в конфигурацию -
включить [VIA_ENABLE](https://www.caniusevia.com/docs/configuring_qmk#create-a-via-keymap-directory-and-files-in-qmk-source).

https://config.qmk.fm/#/kbdfans/kbd67/mkiirgb/v3/LAYOUT_65_ansi_blocker)
