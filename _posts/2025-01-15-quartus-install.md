---
layout: post
category: fpga
title: "Quartus на Mac M4. Епос у чотирьох актах."
date: 2025-01-15
toc: true
permalink: /posts/quartus-install/
---

> *Загальний час: кілька тижнів. Кількість VM: 4. Кількість разів коли хотілося все кинути: не рахував.*
> *Спойлер: в кінці все запрацювало і на Windows, і на Mac. Але шлях був... цікавий.*

---

## Акт I: "Просто встановлю IDE"

**Quartus Prime** — це офіційний інструмент Intel для програмування FPGA. Завантажив. Запустив.

Він просто не запускається на Apple Silicon. Альтернатив нема.
USB Blaster JTAG драйвер — не існує для macOS ARM.

Добре. UTM є — безкоштовний інструмент для запуску віртуальних машин на Mac. Поставлю Ubuntu.

---

## Акт II: Ubuntu в UTM. Три кола пекла.

**Спроба 1: Windows ARM**

Є Windows — думав все просто. Але то була ARM архітектура. Windows ARM може запускати x86 програми через емуляцію — але Quartus із його USB Blaster драйверами цю емуляцію не переварив. Встановилось, але не запустилось.

**Спроба 2: Ubuntu Desktop в UTM (emulate x86_64)**

Качаю Ubuntu 22.04. Повільно. Дуже повільно.

```
Я: "у мене швидкість 650mbps чому так повільно?"
Реальність: Ubuntu сервери в Європі
```

Поставив. Quartus встановив. Запускаю:

```
The Quartus Prime software is optimized for the Intel Nehalem processor
and newer processors. The required extensions were not found on:
'QEMU Virtual CPU version 2.5+'
Terminating...
```

Quartus відмовляється працювати на QEMU процесорі. Він хоче реальний Intel Nehalem або новіший.

Добре, може GUI відкриється хоча б? Почав запускати через SSH з X forwarding. Результат:

```
Authorization required, but no authorization protocol specified
Aborted (core dumped)
```

Години йдуть. xhost, DISPLAY, XAUTHORITY — крутимо всі ці змінні. Quartus або падає з core dump, або зависає на відкритті першого діалогу.

Диск переповнився — Ubuntu зжерла більше ніж очікувалось, LVM не розтягнувся автоматично. Розтягнув вручну через `lvextend` + `resize2fs`.

І тут — одного разу, після чергової комбінації команд — **Quartus відкрився**. GUI запустився. Я відчув полегшення.

```
USB-Blaster [5-3]: Unable to read device chain - Hardware not attached
```

USB Blaster — в системі, Quartus його бачить. Але FPGA до Blaster'а не підключена.

**VirtualHere** — утиліта для передачі USB пристроїв через мережу в VM. Налаштував. Підключив. Quartus бачить USB Blaster... але:

```
Error (209037): JTAG Server can't access selected programming hardware
```

Спробував openOCD. Спробував різні драйвери. Спробував USBIP — зависає намертво (bulk IN/OUT deadlock на JTAG). Спробував помолитись.

**Спроба 3: Ubuntu ARM64 + Rosetta 2**

Нова ідея: Ubuntu 24.04 ARM64 (рідна для M4), Rosetta 2 перекладає x86_64 бінарники на ARM. Теоретично — Quartus (x86_64) має запуститись.

Та сама помилка про Nehalem. Rosetta не емулює SSE4 розширення які Quartus 25.1 перевіряє. Обхідний шлях з `QEMU_CPU=Nehalem` — падав при першій серйозній операції.

Здався.

---

## Акт III: Windows. І мертвий Blaster.

Пішов налаштовувати все на Windows x86. Нативна архітектура, має просто працювати.

Встановив Quartus. Підключив USB Blaster. Quartus Programmer → Hardware Setup → **порожньо**.

```
jtagconfig: No JTAG hardware available
```

Blaster взагалі не бачиться. Device Manager показує "Altera USB-Blaster" з правильним драйвером — але Quartus нічого не бачить.

Вже думав купувати новий Blaster — вирішив заповнити вішліст на ДР, пішов шукати. І натрапив на коментар від доброї людини в інтернеті:

> *"бластер не працює, але якщо прошити — то працює. думайте якщо хочете з цим копатись"*

Дешеві клони USB Blaster використовують мікроконтролер **CH552G** замість оригінального FTDI. Заводська прошивка CH552G **не реалізує протокол Altera JTAG** — тому Quartus не може з ним спілкуватись, незважаючи на правильний драйвер. Серійний номер `FFFFFFFF` в jtagconfig — вірна ознака клона.

**Рішення:** перепрошити CH552G community firmware.

На задній стороні плати є пади **USBISP**. Щоб увійти в bootloader:
1. Відключити Blaster від USB
2. Підключити — і одразу замкнути пінцетом пади D+ і VCC (~1 секунду)
3. Windows покаже "Unknown USB Device" — це bootloader CH552G

Далі WCHISPStudio → вибрати `.bin` з GitHub → Download:

```
Erasing...    complete
Programming.. complete
Verifying...  complete
Succeed!
```

Підключив Blaster звичайно. Відкрив Quartus Programmer:

```
1) USB-Blaster [USB-0]
  020F10DD   EP4CE6/EP4CE10
```

**Blaster живий. FPGA видно. Windows працює!!!**

> 🔧 Якщо дешевий клон Blaster не бачиться — він не зламаний, просто з неправильною прошивкою. CH552G перепрош'ється за 5 хвилин.

---

## Акт IV: Повернення на Mac. Цього разу успішне.

Після того як Windows запрацювала — захотів повернути все на Mac. Але вже з новими знаннями.

Ключові висновки з попередніх спроб:
- Quartus 25.1 на ARM64 — не працює (AVX2 інструкції крашать Rosetta)
- **Quartus 20.1** через Rosetta 2 — **працює** ✅
- USBIP бібліотека для USB passthrough — deadlock, не використовувати
- **VirtualHere** — стабільно ✅

Нова конфігурація VM в UTM:
- **Apple Virtualization Framework** (AVF), не QEMU emulate
- Ubuntu **24.04 ARM64** (рідна для M4 — швидко!)
- Rosetta 2 для запуску x86_64 бінарників Quartus

Налаштував Rosetta в binfmt, встановив Quartus 20.1, підключив VirtualHere.

```bash
~/intelFPGA_lite/20.1/quartus/bin/jtagconfig
# 1) USB-Blaster [...]
#   020F10DD   EP4CE6/EP4CE10
```

**Quartus на Mac. Через Rosetta. Без QEMU. Блискавично.**

Фінальний workflow:

```
Verilog у VSCode (Mac)
    ↓
icarus-verilog → симуляція → WaveTrace (Mac)
    ↓
Quartus 20.1 GUI в UTM ARM64 VM → компіляція → .sof
    ↓
Програматор в Quartus (через VirtualHere) → FPGA
```

---

## Підсумок для тих, хто буде гуглити

| Варіант | Результат |
|---------|-----------|
| Quartus на macOS нативно | ❌ не існує |
| UTM x86_64 emulate + Quartus 25.1 | ❌ QEMU CPU помилка |
| UTM ARM64 + Rosetta + Quartus 25.1 | ❌ AVX2 краш |
| UTM ARM64 + Rosetta + Quartus 20.1 | ✅ працює |
| Windows x86 + оригінальний Blaster | ✅ працює |
| Windows x86 + клон Blaster CH552G | ❌ → перепрошити → ✅ |
| USBIP для USB passthrough | ❌ deadlock |
| VirtualHere для USB passthrough | ✅ стабільно |

---

[← Список компонентів](/posts/components/) \| [LED Blink →](/posts/led-blink/)
