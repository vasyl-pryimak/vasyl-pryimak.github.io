---
layout: post
category: fpga
title: "Nios II/e — коли Verilog стає занадто складним"
date: 2026-06-06
toc: true
permalink: /posts/nios2-i2c-master/
---

> *"Він каже: на вашу плату можна всунути програмний процесор, поміститься. І що вони якраз через це і працюють."*

---

## Як все почалось — розмова з FPGA Developer-ом

На днях познайомився з одним FPGA Developer-ом — почали говорити про залізо, і я дізнався дуже багато цікавого.

Чим займаються такі розробники: фізика сигналів — дБ, Гц, спектр, перетворення Фур'є, DSP фільтри. І ще відкрилась для мене значно ширший і глибший океан FPGA.

Два конкретних відкриття:
1. **На мою плату EP4CE6 можна всунути Nios II/e** — програмний процесор прямо в логічних клітинках FPGA
2. **MATLAB/Simulink** — так тестують і проектують алгоритми обробки сигналів перед тим як писати Verilog

Вирішив почати з Nios II.

---

## Що таке Nios II

**Nios II** — soft-core процесор від Altera/Intel. Не фізичний чіп — а **опис схеми на Verilog**, який синтезується в логічні клітинки FPGA.

Три варіанти:

| Варіант | LE (логічні елементи) | Швидкість |
|---|---|---|
| Nios II/f (fast) | ~1800 | максимальна |
| Nios II/s (standard) | ~1400 | середня |
| **Nios II/e (economy)** | **~700** | мінімальна, але достатня |

EP4CE6 має 6272 LE. Nios II/e займає ~700 — **влізає**.

---

## Навіщо процесор в FPGA

До цього у мене був `i2c_master.v` — Verilog FSM (кінцевий автомат) на ~200 рядків:

```verilog
localparam S_IDLE    = 5'd0,
           S_C_START = 5'd1,
           S_C_ADDR  = 5'd2,
           // ... ще 14 станів
           S_WAIT    = 5'd16;

always @(posedge clk) begin
    case (state)
    S_IDLE: begin ... end
    S_C_START: begin ... end
    // ...
    endcase
end
```

Той самий код на C з Nios II:

```c
i2c_start();
i2c_write_byte(0x90);    // адреса ADS1115
i2c_write_byte(0x01);    // config register
i2c_write_byte(cfg[ch]); // канал + PGA
i2c_write_byte(0xE3);    // 860 SPS
i2c_stop();
```

200 рядків Verilog → 5 рядків C. Ось навіщо процесор.

---

## Крок 1 — Platform Designer

**Platform Designer** — інструмент всередині Quartus де в GUI збираєш систему з IP-блоків і з'єднуєш їх мишкою. На виході — `.qsys` файл і згенерований Verilog який Quartus синтезує в FPGA.

```
Platform Designer (GUI)
    ↓ Generate HDL
nios_ctrl.v  ← синтезується в FPGA як звичайний Verilog
```

Наша система `nios_ctrl`:

```
nios_ctrl:
  ├── Nios II/e (CPU)
  ├── On-chip RAM 8KB
  ├── JTAG UART
  ├── PIO: volume, wet, decay, pitch, piezo (16-bit output)
  └── PIO: scl, sda_out (1-bit output), sda_in (1-bit input)
```

Кожен блок — окремий IP-модуль:

| Блок | Що це |
|---|---|
| **Nios II/e** | Сам процесор (32-bit RISC) — синтезується в ~700 LE |
| **On-chip RAM 8KB** | Пам'ять для C програми — з M9K блоків FPGA |
| **JTAG UART** | Зв'язок з комп'ютером для дебагу (printf) |
| **PIO 16-bit output** | Регістр куди Nios II пише значення потенціометра |
| **PIO 1-bit output/input** | Один пін — SCL або SDA для I²C |

Всі блоки з'єднані через **Avalon шину** — внутрішню шину Nios II. Як PCIe у комп'ютері — через неї процесор звертається до будь-якого компонента за адресою:

```
Nios II пише: адреса 0x5040 = значення 12345
              ↓
         PIO_VOLUME → Verilog DSP читає volume_raw = 12345
```

В C коді це виглядає просто:
```c
IOWR_ALTERA_AVALON_PIO_DATA(PIO_VOLUME_BASE, val);
// розгортається в: *(volatile int*)0x5040 = val;
```

### Підводний камінь: Java Swing і чорний інтерфейс

Platform Designer написаний на Java. При запуску через SSH+XQuartz — інтерфейс повністю чорний.

Фікс — додати в `/etc/environment`:
```
JAVA_TOOL_OPTIONS="-Dswing.defaultlaf=javax.swing.plaf.metal.MetalLookAndFeel -Dsun.java2d.xrender=false"
```

> ⚠️ `.bashrc` і `.profile` не підходять — SSH не підхоплює їх. Тільки `/etc/environment`.

---

## Крок 2 — Компіляція без Eclipse

**Eclipse** — це IDE для C/C++ розробки під Nios II. Як IntelliJ для Java — автодоповнення, дебагер, кнопка Run. Але C код — це звичайний текстовий файл. Можна писати в блокноті, компілювати командним рядком. Eclipse в цьому проекті не вдалось встановити — і це не завадило.

```bash
# BSP — знає де що знаходиться в системі
nios2-bsp hal bsp nios_ctrl.sopcinfo --cpu-name nios2_gen2_0

# Makefile для проекту
nios2-app-generate-makefile --bsp-dir bsp --app-dir app \
  --elf-name reverb.elf --src-files app/main.c

# Компіляція і завантаження
cd app && make && nios2-download -g reverb.elf
```

**BSP (Board Support Package)** — автоматично згенерований C код який знає про залізо: де знаходиться кожен PIO в адресному просторі, скільки RAM, яка тактова частота. Без BSP твій код не знає нічого про конкретне залізо.

---

## Крок 3 — C код для ADS1115

```c
#define SCL_HI()   IOWR_ALTERA_AVALON_PIO_DATA(PIO_SCL_BASE, 1)
#define SDA_HI()   IOWR_ALTERA_AVALON_PIO_DATA(PIO_SDA_OUT_BASE, 1)
#define SDA_READ() IORD_ALTERA_AVALON_PIO_DATA(PIO_SDA_IN_BASE)

// Головний цикл
while (1) {
    unsigned short val = ads1115_read(ch);
    IOWR_ALTERA_AVALON_PIO_DATA(pio_base[ch], val);
    ch = (ch + 1) % 5;
}
```

Nios II читає ADS1115 і пише значення в PIO. Verilog DSP читає з PIO і не знає нічого про I²C.

---

## Результат

```
Total logic elements: 3602 / 6272  (57%)
Program size:         5100 bytes / 8192 bytes
```

Nios II/e влізає в EP4CE6 разом з повним spring reverb + pitch shifter + piezo vibrato. Крутилки працюють.

---

## Нюанси і налаштування

### Затримка при крутінні потенціометра

Після першого запуску — відчутна затримка 0.5-1 секунда. Три параметри в C коді відповідають за швидкість:

**1. SPS (швидкість конверсії ADS1115)** — в байті конфігу:
```c
i2c_write_byte(0xE3);  // LSB конфігу ADS1115
//             ^^^^
// 0x83 = 128 SPS → конверсія 7.8ms
// 0xE3 = 860 SPS → конверсія 1.2ms  ← наш вибір
```
128 SPS × 5 каналів = 39ms мінімум. 860 SPS × 5 = 6ms.

**2. Wait loop** — очікування поки ADS1115 завершить конверсію:
```c
volatile int w;
for (w = 0; w < 200; w++);  // ~2ms при 860 SPS
```
На Nios II/e з `-O0` кожна ітерація займає ~10мкс. Тому `200 × 10мкс = 2ms` — достатньо для 860 SPS.

**3. i2c_delay** — час між бітами I²C:
```c
static void i2c_delay(void) {
    volatile int i;
    for (i = 0; i < 5; i++);  // ~50мкс → ~20kHz I²C
}
```

Всі три підбирались експериментально — зменшували поки не стало швидко і стабільно.

### Тихо при старті

PIO регістри при reset = 0 → volume = 0 → тиша поки Nios II не прочитає перше значення (~100ms).

Рішення — ініціалізувати дефолтні значення перед головним циклом:
```c
IOWR_ALTERA_AVALON_PIO_DATA(PIO_VOLUME_BASE, 0x7FFF);
IOWR_ALTERA_AVALON_PIO_DATA(PIO_WET_BASE,    0x7FFF);
// ...
```

---

## Де межа між C і Verilog

```
Nios II при 50MHz:
  тактів між семплами = 50,000,000 / 48,000 = ~1041
  ÷ ~5 тактів на інструкцію = ~200 інструкцій на семпл
```

За 20мкс між семплами Nios II встигає виконати ~200 інструкцій. Для I²C і управління — вистачає. Для реверба з трьома delay lines — ні.

В Verilog те саме множення виконується за **1 такт паралельно** з усіма іншими операціями.

| Завдання | де краще |
|---|---|
| I²C, SPI, UART протоколи | **C (Nios II)** |
| Складна логіка управління | **C (Nios II)** |
| Real-time DSP (reverb, filter) | **Verilog (hardware)** |
| Множення на кожен семпл | **Verilog (hardware)** |

**Ідеальний розподіл:**
```
Nios II → читає сенсори, керує параметрами
Verilog → виконує DSP в реальному часі
```

---

## Наступний крок

MATLAB/Simulink — те що використовують на виробництві. Спроектувати алгоритм в MATLAB, протестувати на аудіо файлах, потім генерувати Verilog або C код.

Океан відкривається далі 🌊

---

[← Pitch Shifter](/posts/pitch-shifter/)
