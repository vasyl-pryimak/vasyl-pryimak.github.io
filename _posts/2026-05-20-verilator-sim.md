---
layout: post
category: fpga
title: "Verilator: крутилки в браузері замість перезаливки FPGA"
date: 2026-05-20
toc: true
permalink: /posts/verilator-sim/
---

> *"Підкрутив decay — тепер збери все на столі, запусти віртуалку, дочекайся Quartus..."*

**Проблема:** щоб почути як звучить змінений параметр — потрібно:

1. Підключити FPGA до Mac через USB Blaster
2. Підключити навушники до DAC (CS4344)
3. Підключити мобільний як аудіо-джерело через ADC (PCM1808)
4. Запустити Ubuntu у UTM з Rosetta 2 — бо Quartus тільки під Linux/Windows
5. Пробрасити USB Blaster через VirtualHere по SSH
6. Запустити Quartus через SSH, дочекатись компіляції бітстріму (~2 хвилини)
7. Залити у FPGA через USB Blaster
8. Послухати 5 секунд
9. Повернутись до кроку 6

<a href="/assets/images/fpga_4_knobs.JPG" class="glightbox" style="display: block;" data-description="Поточний hardware setup: FPGA, 4 крутилки, DAC, ADC">
  <img src="/assets/images/fpga_4_knobs.JPG" alt="Hardware setup: FPGA з 4 крутилками" style="cursor: zoom-in; max-width: 480px;">
</a>

*Ось що потрібно зібрати щоб просто почути зміну параметра.*

Це ОК для фінального тесту. Але для підбору параметрів на слух — нестерпно.

**Рішення:** скомпілювати Verilog у C++ і гоняти звук через нього прямо на ноутбуці. Нічого не підключати. Жодних віртуалок.

<a href="/assets/images/VerilatorUI.png" class="glightbox" style="display: block;" data-description="Браузерний UI: 4 крутилки і осцилоскоп">
  <img src="/assets/images/VerilatorUI.png" alt="Verilator браузерний UI" style="cursor: zoom-in;">
</a>

---

## Verilator: що це таке

**[Verilator](https://www.veripool.org/verilator/)** — інструмент який компілює Verilog-модулі у C++ класи. Ти пишеш testbench на C++, підключаєш до нього клас — і маєш точну цифрову симуляцію свого Verilog. Не апроксимацію, не емуляцію — буквально той самий RTL-код, тільки в C++.

```
reverb_core.v ──── Verilator ────→ Vreverb_core.h / .cpp
delay_line.v  ──/                       │
                                        ↓
                                  sim_main.cpp
                                  (читає WAV → тикає такти → пише WAV)
```

Встановлення на Mac: `brew install verilator`.

---

## Збірка

```makefile
# sim/Makefile
VERILATOR = verilator
VSRCS     = ../reverb_core.v ../delay_line.v

sim: $(VSRCS) sim_main.cpp
    $(VERILATOR) --cc --exe --build -j 0 \
        -Wall --Wno-fatal \
        -CFLAGS "-O2 -std=c++17" \
        -Mdir obj_dir \
        -o ../reverb-sim \
        $(VSRCS) sim_main.cpp
```

`--cc` — генерувати C++. `--exe --build` — зібрати бінарник за один крок. `-j 0` — всі ядра.

Бінарник назвав `reverb-sim` а не `sim` — бо поруч є папка `sim/`, і `make` бачив директорію як вже побудовану ціль і нічого не компілював. Класична пастка.

---

## sim_main.cpp: WAV → Verilog → WAV

Testbench дзеркалює ланцюжок сигналу з `spring_reverb.v`:

```
input.wav → pre-delay (C++) → reverb_core (Verilator) → × volume (C++) → output.wav
```

### Що тут симулюється — а що ні

`reverb_core.v` і `delay_line.v` — справжній Verilog, запущений через Verilator. Саме їх і тестуємо.

Pre-delay і volume — написані окремо на C++. Це не повна емуляція: в Verilog вони теж реалізовані, але там вони "живуть" всередині великого ланцюжка з I²S, I²C, тактуванням і апаратними перериваннями. Симулювати весь цей обвіс через Verilator — складно і не потрібно. Тому в testbench'і вони замінені C++ еквівалентами: той самий алгоритм, але без апаратної специфіки.

Теоретично між двома реалізаціями можуть бути розбіжності. На практиці — це свідомий компроміс: volume це одне множення, а pre-delay — ring buffer. Там важко помилитись по-різному.

**Ring buffer** — масив який "замикається в кільце". Замість того щоб зсувати весь масив кожен такт, просто рухаєш один вказівник:

```
[0][1][2][3]...[997][998][999]
                              ↑ після 999 — знову 0

такт 1: пишеш в [0],   читаєш з [500]  → затримка 500 семплів
такт 2: пишеш в [1],   читаєш з [501]
...
такт 500: пишеш в [500], читаєш з [0]  ← "кільце" замкнулось
```

Розмір буфера = довжина затримки в семплах. При 48kHz і буфері 4800 — затримка 100мс.

### Такт — це 2 виклики eval()

```cpp
void tick(Vreverb_core* top) {
    top->clk = 0; top->eval();
    top->clk = 1; top->eval();
}
```

**Один семпл — два такти (we=1, потім we=0):**

```cpp
top->audio_in = sample;
top->we = 1; tick(top);        // фронт LRCLK — записуємо семпл
int16_t out = top->audio_out;  // берем результат
top->we = 0; tick(top);        // пауза між семплами
```

Це дзеркалює поведінку реального LRCLK на 48kHz — we пульсує один такт на кожен семпл.

**Параметри:**

```
--volume=0.8        A0: гучність
--wet=0.8           A1: wet/dry
--decay=0.6         A2: хвіст реверба
--predelay=0.3      A3: затримка перед ревербом (0..43ms)
--wet=0.0:1.0       sweep: лінійно від 0 до 1 за час пісні
--sweep-dur=10      тривалість sweep у секундах
--auto=curve.csv    CSV автоматизація: time,volume,wet,decay,predelay
```

Запуск:

```bash
./reverb-sim input.wav output.wav --wet=0.8 --decay=0.6
```

Перша ж перевірка: **реально так само звучить як через плату.** Затримки, хвіст, характер пружини — ідентично.

### Sweep — почути весь діапазон за один прогін

Замість фіксованого значення можна задати діапазон: `--wet=0.0:1.0`. Симулятор плавно змінює параметр від 0 до 1 протягом всього треку. Зручно коли хочеш почути як звучить весь діапазон wet або decay — не перезапускаючи симуляцію десятки разів.

```bash
./reverb-sim input.wav output.wav --wet=0.0:1.0 --sweep-dur=20
```

За 20 секунд wet пройде від сухого сигналу до повного реверба.

### CSV — автоматизація як у DAW

Якщо sweep не достатньо гнучкий — можна передати CSV файл з точними значеннями по часу:

```
time,volume,wet,decay,predelay
0.0,0.8,0.0,0.6,0.2
5.0,0.8,0.5,0.6,0.2
10.0,0.8,1.0,0.8,0.3
```

Симулятор читає таблицю і лінійно інтерполює між точками. По суті — автоматизація параметрів як в DAW, тільки в текстовому файлі.

```bash
./reverb-sim input.wav output.wav --auto=curve.csv
```

---

## Браузерний UI: крутилки в реальному часі

Консоль — це добре для батчів. Але для підбору на слух потрібен live UI.

Зробили три шари:

### server.py — Python HTTP + subprocess

```python
# POST /render → запускає reverb-sim → output.wav
result = subprocess.run(
    [SIM_BIN, INPUT_WAV, OUTPUT_WAV,
     f"--volume={volume:.4f}", f"--wet={wet:.4f}",
     f"--decay={decay:.4f}", f"--predelay={predelay:.4f}"],
    capture_output=True, text=True
)
```

Два нюанси яких не видно одразу:

**ThreadingMixIn** — без нього сервер однопоточний. Поки рендер (1-2 секунди) — браузер висить. `ThreadingMixIn` вирішує за один рядок.

**HTTP Range requests (206 Partial Content)** — браузерний `<audio>` при seek надсилає `Range: bytes=X-Y`. Без обробки цього хедера аудіо не можна перемотувати і після перезавантаження файлу програвач скидається в позицію 0. Реалізували вручну:

```python
m = re.match(r"bytes=(\d+)-(\d*)", rng)
if m:
    start, end = int(m.group(1)), int(m.group(2)) or total-1
    self.send_response(206)
    self.send_header("Content-Range", f"bytes {start}-{end}/{total}")
    self.wfile.write(data[start:end+1])
```

### index.html — 4 крутилки + осцилоскоп

Чотири крутилки (A0-A3) з SVG arc-індикаторами, 270° sweep. Drag вертикальний — 140px = повний діапазон. Дебаунс 500ms: поки крутиш — таймер скидається, рендер запускається тільки після паузи.

**Відновлення позиції після рендеру:**

```js
audio.addEventListener('canplay', () => {
    audio.addEventListener('seeked', () => {
        if (wasPlaying) audio.play();
    }, { once: true });
    audio.currentTime = savedTime;
}, { once: true });
audio.src = '/output.wav?t=' + Date.now();  // cache-bust
```

`?t=` — щоб браузер не брав стару версію з кешу.

**Осцилоскоп** — Web Audio API:

```js
const analyser = audioCtx.createAnalyser();
audioCtx.createMediaElementSource(audio)
    .connect(analyser)
    .connect(audioCtx.destination);

analyser.getByteTimeDomainData(buffer);  // 2048 значень 0..255
// малюємо лінію по canvas
```

При паузі — хвиля завмирає, не скидається в нуль.

---

## Баги які знайшла симуляція

Симуляція точна — і показала два баги в `reverb_core.v` яких на залізі не помітив через менший рівень вхідного сигналу з телефона.

### Баг 1: dry + wet overflow

Симуляція одразу дала рипіння на нормальному WAV файлі. З телефоном на залізі (-12..-18 dBFS) — не чутно, але WAV з нормальним мастерингом — 0 dBFS:

```
dry_out = 32767 × 1.0 = 32767
wet_out = 32767 × 0.8 ≈ 26214
mix = 58981  →  переповнення 16 біт  →  hard clip  →  рипіння
```

Фікс — crossfade: `dry_gain = 1.0 - wet`. Сума dry+wet ніколи не перевищує 100%.

Перевірив Verilog — там теж вже пофіксовано: `wire [15:0] dry_gain = 16'h7FFF - wet_gain;`. Симуляція підтвердила що підхід правильний.

### Баг 2: wet_sum знаковий wrap (в Verilog)

```verilog
wire signed [17:0] wet_sum_18 = sign_ext(tap1) + sign_ext(tap2) + sign_ext(tap3);
wire signed [15:0] wet_sum = wet_sum_18[16:1];  // ÷2
```

При резонансі кожен тап може досягати насичення (32767). Сума трьох:

```
3 × 32767 = 98301  →  bits[16:1] = 0xBFFE = -16386  (знак перевертається!)
```

Позитивний wet_sum раптово стає великим від'ємним → різкий стрибок сигналу → клацання.

**Фікс:** saturating divide-by-2 через функцію `sat17`:

```verilog
function signed [15:0] sat17;
    2'b01:   sat17 = 16'sh7FFF;  // positive overflow → затискаємо
    2'b10:   sat17 = 16'sh8000;  // negative overflow → затискаємо
    default: sat17 = val[15:0];  // нормальний діапазон
endfunction

wire signed [15:0] wet_sum = sat17(wet_sum_18[17:1]);
```

Якщо біт[17] і біт[16] різні — переповнення, затискаємо до максимуму. Однакові — нормальний діапазон.

Перевірив Verilog — фікс вже є і в hardware коді. Симуляція підтвердила що він працює правильно.

---

## Підсумок

| Що | Як |
|----|----|
| Симуляція | Verilator: `reverb_core.v` + `delay_line.v` → C++ |
| Аудіо I/O | 16-bit WAV @ 48kHz, читання/запис вручну |
| Параметри | `--wet`, `--decay`, `--volume`, `--predelay`, sweep, CSV |
| UI | Python HTTP сервер + HTML/JS, 4 крутилки |
| Осцилоскоп | Web Audio API `AnalyserNode` + Canvas |
| Знайдені баги | dry+wet overflow, wet_sum sign wrap |

Тепер цикл: покрутив в браузері → 1-2 секунди → чуєш результат. Без USB Blaster, без Quartus, без UTM, без трьох підключених пристроїв.

Код симулятора: [github.com/vasyl-pryimak/fpga-synth/.../sim](https://github.com/vasyl-pryimak/fpga-synth/tree/main/spring_reverb_3_knobs/sim)

---

[← Фаза 5: Ефекти](/posts/phase5-effects/) | [Pitch Shifter →](/posts/pitch-shifter/)
