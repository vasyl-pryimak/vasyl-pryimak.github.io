---
layout: post
category: fpga
title: "Список компонентів"
date: 2025-01-01
toc: true
permalink: /posts/components/
---

> *Перш ніж щось запрацює — треба зрозуміти з чого воно взагалі складається.*

---

## FPGA

**Cyclone IV EP4CE6E22C8N** — серце всього проєкту. Це FPGA від Intel/Altera, одна з найдоступніших для хоббі.

- 6272 логічних елементи
- 270 Кбіт вбудованої пам'яті
- 15 множників 18×18
- Інструмент розробки: **Quartus Prime Lite** (безкоштовний)
- Прошивка через **USB Blaster**

Плата: **EP4CE6 FPGA Development Board** — дешева китайська плата з AliExpress, але робоча.

<a href="/assets/images/fpga_board.png" class="glightbox" style="display: block;" data-description="EP4CE6 FPGA Development Board">
  <img src="/assets/images/fpga_board.png" alt="EP4CE6 FPGA Development Board" style="cursor: zoom-in; max-width: 380px;">
</a>

*Ось вона. Скромно виглядає — але всередині 6272 логічних елементів.*

---

## DAC — виводимо звук

**Cirrus Logic CS4344** — стерео DAC (цифро-аналоговий перетворювач).

- Інтерфейс: **I²S** (Inter-IC Sound)
- Частота дискретизації: до 96 кГц
- Розрядність: 24 біти
- Підключення: 4 дроти (MCLK, SCLK, LRCK, SDIN)

Саме через нього звук іде в навушники.

---

## ADC — захоплюємо звук

**Texas Instruments PCM1808** — стерео ADC (аналого-цифровий перетворювач).

- Інтерфейс: **I²S**
- Частота дискретизації: до 96 кГц
- Розрядність: 24 біти
- Живлення: 3.3В

Мікрофон або лінійний вхід → PCM1808 → FPGA.

---

## ADC для потенціометрів

**Texas Instruments ADS1115** — 16-бітний ADC для читання крутилок.

- Інтерфейс: **I²C**
- 4 канали (мультиплексор)
- Напруга: 0–3.3В або 0–5В
- Точність: 16 біт (реально ~15 корисних)
- Адреса I²C: визначається пінами ADDR

Проблема: FPGA добре вміє паралельну логіку, але I²C — це послідовний протокол із тайммінгами. Тому між FPGA і ADS1115 стоїть Arduino.

---

## Arduino як I²C міст *(optional)*

**Arduino Nano** — посередник між FPGA і ADS1115.

- Читає 4 канали ADS1115 по I²C
- Передає значення у FPGA через **UART**
- Частота оновлення: ~100 Гц на канал

Так, це виглядає як милиця. Але працює.

---

## Схема зв'язків

<a href="/assets/images/spring_reverb.svg" class="glightbox" style="display: block;" data-description="Схема підключення компонентів">
  <img src="/assets/images/spring_reverb.svg" alt="Схема підключення компонентів" style="cursor: zoom-in; max-width: 380px;">
</a>

```
Навушники ←── CS4344 ←── FPGA
                           │
Мікрофон ──→ PCM1808 ──→ FPGA
                           │
Крутилки ──→ ADS1115 ──→ Arduino ──→ FPGA
```

---

## Що далі

Далі — встановлення Quartus і перший проєкт: змусити FPGA блимати світлодіодом.

---

## Список покупок

**Компоненти:**
- EP4CE6 FPGA Development Board (Cyclone IV EP4CE6E22C8N)
- USB Blaster (програматор для FPGA)
- CS4344 DAC модуль
- PCM1808 ADC модуль (HW-935)
- ADS1115 ADC модуль
- Arduino Nano *(optional — для зв'язку з ADS1115)*
- 4× потенціометр 10kΩ
- Breadboard + дроти

**Інструменти для пайки (всі компоненти зазвичай приходять з ніжками окремо):**
- Паяльник
- Третя рука
- Плоскогубці
- Пінцет
- Целюлозна губка без добавок
- Мідна губка для очищення жала
- Припій
- Каніфоль
- Спирт (для очищення)
- Вентилятор
- Мультиметр

---

**[Далі: Quartus на Mac M4. Епос у чотирьох актах. →](/posts/quartus-install/)**
