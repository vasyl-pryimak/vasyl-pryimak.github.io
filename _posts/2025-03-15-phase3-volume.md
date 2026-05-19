---
layout: post
category: fpga
title: "Фаза 3: ADS1115, Arduino і Найдовший Квест"
date: 2025-03-15
toc: true
permalink: /posts/phase3-volume/
---

> *Один чіп за $2, якого я двічі вважав мертвим. Перший раз — він жив, але breadboard коротив живлення. Другий раз — чіп жив, але код читав дані не в той момент.*

**Ціль:** потенціометр → ADC → I²C → FPGA → множення семплів → гучність.

Два компоненти для цієї частини:

**ADS1115** — 16-бітний ADC, читає напругу з крутилок по I²C:

![ADS1115 ADC модуль](/assets/images/ads1115.png)

**Потенціометр 10kΩ** — власне, та сама крутилка:

![Потенціометр 10kΩ](/assets/images/potentiometer.png)

---

## Акт I: Логічний аналізатор і каруселя помилок

Написав I²C master на Verilog. Підключив. Взяв PulseView (open-source логічний аналізатор).

**Помилка #1 — перевернутий декодер:**
```
Address write: 21  ← що за 21?
```
В PulseView декодері SCL і SDA були переставлені місцями. Поміняв.

**Помилка #2 — замалий sample rate:**
I²C @ 100kHz, один біт = 10 мкс. Sample rate стояв **20kHz** — більшість бітів просто не захоплювалися. Виставив 1 MHz.

**Помилка #3 — вимірював не те:**
Виміряв напругу між VDD і ADDR — 3.3V. Вирішив: ADDR = 3.3V → адреса 0x49. Змінив код. NACK.
Потім дійшло: я вимірював **різницю** VDD(3.3V) − ADDR(GND) = 3.3V. ADDR = GND = 0V → адреса 0x48. Повернув.

NACK залишився. Вирішив що чіп мертвий — і пішов іншим шляхом.

> 😤 Спойлер: чіп був живий. Але це з'ясується пізніше.

---

## Акт II: Arduino в ролі самозванця

**Інсайт:** Arduino Uno має 10-бітний АЦП і може бути I²C slave. Якщо посадити на адресу 0x48 — FPGA питає ADS1115, Arduino відповідає замість нього. FPGA-код міняти не треба.

```cpp
Wire.begin(0x48);  // "Доброго дня, я — ADS1115"
```

**Але спочатку — залити скетч.**
Arduino не підключалась. **Причина:** VirtualHere перехоплювала всі USB пристрої. Вимкнув → Arduino завантажилась з першого разу.

> 😅 Пам'ятай вимикати VirtualHere коли не потрібен.

---

## Акт III: 5V vs 3.3V — фізика не пробачає

Arduino — 5V логіка. FPGA — 3.3V. Підключити I²C напряму не можна.

I²C — **open-drain** протокол: пристрої ніколи не "штовхають" лінію вгору, тільки тягнуть до GND або відпускають. Напруга HIGH на шині = напруга pull-up резистора.

```
З pull-up до 5V:   Arduino відпускає → 5V на вході FPGA → 💥
З pull-up до 3.3V: Arduino відпускає → 3.3V на вході FPGA → ✅
                   Arduino тягне GND → 0V на вході FPGA  → ✅
```

**Рішення:** ADS1115 модуль має вбудовані pull-up резистори 10kΩ до 3.3V. Підключив SDA/SCL модуля до шини — вони тягнуть лінії до 3.3V.

---

## Акт IV: Serial в ISR — бомба

`Wire.onRequest` і `Wire.onReceive` — це **ISR** (Interrupt Service Routine). Коли FPGA надсилає запит по I²C — Arduino отримує переривання і одразу викликає callback, навіть посеред `loop()`. Як `@EventListener` в Java — тільки без черги і без окремого потоку: все інше зупиняється і виконується ISR.

Тому в ISR — тільки швидкі операції. `Serial.println()` займає мілісекунди, FPGA чекає мікросекунди:

```cpp
void sendReading() {          // ← ISR
    Wire.write(smoothed);
    Serial.println(smoothed); // ← блокує на мілісекунди → Wire не встигає
}
```

**Рішення:** флаг + логування в `loop()`.

```cpp
volatile bool requested = false;

void sendReading() {
    Wire.write(smoothed);
    requested = true;
}

void loop() {
    smoothed = (smoothed * 7 + analogRead(POT_PIN)) / 8;
    if (requested) { Serial.println(smoothed); requested = false; }
    delay(5);
}
```

> ⚡ Ніколи не використовуй `Serial` в Wire callbacks.

---

## Акт V: `volatile` — невидимий баг

```cpp
volatile int smoothed = 512;
```

Без `volatile` компілятор кешує `smoothed` в регістрі. ISR читає застаріле значення.

Для Java-розробника: той самий `volatile` що між потоками — тільки тут між `loop()` і ISR.

---

## Акт VI: Фінальний бос — Wire.write() і C++ overloads

Serial показує `Sent: 42`, `Sent: 87`. Але гучність — не змінюється.

```cpp
int16_t val = map(smoothed, 0, 1023, 0, 99);
Wire.write(val & 0xFF);  // ← НЕ ПРАЦЮЄ
```

**Рішення:**
```cpp
Wire.write(smoothed);  // ← ПРАЦЮЄ!!!
```

**Чому?** `val & 0xFF` де `val` — `int16_t` → результат `int` → інший C++ overload → інша поведінка.
`smoothed` — `int` 0..1023, Wire бере молодший байт → 0..255. Просто і працює.

> 🤦 C++ overload resolution — окрема наука. Спробуй найпростіший варіант першим.

---

## FPGA: математика гучності

```verilog
wire [15:0] vol = {1'b0, volume_raw[14:0]};

// лівий канал
wire signed [31:0] left_vol  = $signed(left_cap)  * $signed(vol);
wire signed [15:0] left_out  = left_vol[30:15];

// правий канал — паралельно
wire signed [31:0] right_vol = $signed(right_cap) * $signed(vol);
wire signed [15:0] right_out = right_vol[30:15];
```

```
vol = 0       → out = 0        → тиша
vol = 16384   → out = cap / 2  → половина гучності
vol = 32767   → out ≈ cap      → повна гучність
```

У Verilog — це **реальна схема множника**. Лівий і правий канал обчислюються **одночасно**, кожен такт.

---

## ADS1115 живий. Breadboard вбивав живлення.

Після того як Arduino запрацювала — вирішив перевірити ADS1115 ще раз. Підключив джамперами напряму до Arduino, обходячи breadboard:

```
raw: 4651    voltage: 0.581V
raw: 25939   voltage: 3.242V
```

**ADS1115 живий.** Breadboard коротив VDD до GND — чіп просто ніколи не отримував нормального живлення.

> 🪛 Спочатку перевіряй breadboard. Потім — чіп.

---

## Таблиця болю

| Що | Проблема | Рішення |
|----|----------|---------|
| ADS1115 NACK | SCL/SDA swap в PulseView | Поміняти канали в декодері |
| ADS1115 NACK | Sample rate 20kHz | Sample rate 1MHz |
| ADS1115 NACK | Адреса 0x49 vs 0x48 | Вимірювати абсолютну напругу |
| ADS1115 NACK | КЗ в breadboard | Підключити джамперами напряму |
| Arduino не заливається | VirtualHere перехоплює USB | Вимкнути VirtualHere |
| Serial в ISR | Wire.write не встигає | Serial тільки в loop() |
| Гучність не змінюється | `volatile` відсутній | `volatile int smoothed` |
| Wire.write(val & 0xFF) | Неправильний C++ overload | `Wire.write(smoothed)` |

---

[← Фаза 2: PCM1808](/posts/phase2-pcm1808/) \| [Фаза 4: Reverb →](/posts/phase4-reverb/)
