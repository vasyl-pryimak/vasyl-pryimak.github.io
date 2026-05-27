---
layout: post
category: fpga
title: "Початок"
date: 2026-05-09
toc: true
permalink: /posts/components/
---

Ідея: підключити п'єзо до пружини — і отримати справжній аналоговий spring reverb як джерело звуку. Пропустити через нього аудіо з телефону чи MP3, додати ефекти — бітрашер, wet/dry мікс, decay — які керуються крутилками в реальному часі. В планах ще кнопки для перемикання пресетів ефектів.

Весь DSP — на FPGA, Verilog з нуля, без IP-ядер.

**GitHub:** [vasyl-pryimak/fpga-synth](https://github.com/vasyl-pryimak/fpga-synth)

---

Перш ніж щось запрацює — треба зрозуміти з чого воно взагалі складається.

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

## ADC — захоплюємо звук

**PCM1808** — стерео ADC (аналого-цифровий перетворювач).

- Інтерфейс: **I²S**
- Частота дискретизації: до 96 кГц
- Розрядність: 24 біти
- Живлення: 3.3В

Лінійний вхід (телефон, ноут, MP3) → PCM1808 → FPGA.

---

## DAC — виводимо звук

**CS4344** — стерео DAC (цифро-аналоговий перетворювач).

- Інтерфейс: **I²S** (Inter-IC Sound)
- Частота дискретизації: до 96 кГц
- Розрядність: 24 біти
- Підключення: 4 дроти (MCLK, SCLK, LRCK, SDIN)

Саме через нього звук іде в навушники.

---

## ADC для потенціометрів

**ADS1115** — 16-бітний I²C ADC для читання крутилок. Підключається до FPGA по I²C — чисто і просто.

Альтернатива: **Arduino UNO**, який читає потенціометри через аналогові входи і передає значення у FPGA через UART. Так, це милиця. Але якщо ADS1115 ще їде з AliExpress — працює.

---

## Схема зв'язків

<a href="/assets/images/spring_reverb.svg" class="glightbox" style="display: block;" data-description="Схема підключення компонентів">
  <img src="/assets/images/spring_reverb.svg" alt="Схема підключення компонентів" style="cursor: zoom-in; max-width: 380px;">
</a>

```
Навушники ←── CS4344 ←── FPGA
                           │
Аудіо вхід ─→ PCM1808 ──→ FPGA
                           │
Крутилки ──→ ADS1115 (або Arduino) ──→ FPGA
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
- Arduino UNO *(optional — альтернатива ADS1115)*
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
