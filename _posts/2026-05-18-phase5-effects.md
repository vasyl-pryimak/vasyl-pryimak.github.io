---
layout: post
category: fpga
title: "Фаза 5: Чотири крутилки і одна яка не чутно."
date: 2026-05-18
toc: true
permalink: /posts/phase5-effects/
---

> *"Додав pre-delay. Технічно — правильно. На слух — нічого."*

**Ціль:** розширити Phase 4 — decay через потенціометр, pre-delay, і щось нове на A2.

---

## Що було в Phase 4

Два потенціометри: A0 = гучність, A1 = wet/dry. Decay (хвіст реверба) — захардкодований на 0.75. Все.

---

## Частина 1: Decay-крутилка і Pre-delay

### Нові крутилки

| Pot | ADS1115 | Функція |
|-----|---------|---------|
| A0 | AIN0 | Гучність |
| A1 | AIN1 | Wet/Dry crossfade |
| A2 | AIN2 | Decay (хвіст реверба) |
| A3 | AIN3 | Pre-delay (0..43ms) |

### i2c_master.v — тепер читає 4 канали

В Phase 4 читав тільки A0 і A1. Розширили `case` на 4 значення:

```verilog
output reg [15:0] decay_raw    = 16'h0000,  // A2
output reg [15:0] predelay_raw = 16'h0000,  // A3

always @(*) case (ch)
    2'd0: cfg_msb = 8'hC3;  // AIN0
    2'd1: cfg_msb = 8'hD3;  // AIN1
    2'd2: cfg_msb = 8'hE3;  // AIN2
    2'd3: cfg_msb = 8'hF3;  // AIN3
endcase

case (ch)
    2'd0: volume_raw   <= adc_data;
    2'd1: wet_raw      <= adc_data;
    2'd2: decay_raw    <= adc_data;
    2'd3: predelay_raw <= adc_data;
endcase
ch <= ch + 2'd1;  // 0→1→2→3→0, автоматичний wrap
```

`ch` — 2-бітний лічильник, переповнення само дає цикл. Повний цикл: 4 канали × ~40ms = **~160ms**. Для потенціометрів — достатньо.

### reverb_core.v — decay тепер порт, не константа

```verilog
// БУЛО (Phase 4):
wire signed [31:0] fb1_mul = $signed(tap1_raw) * $signed(16'h6000); // 0.75, назавжди

// СТАЛО (Phase 5):
module reverb_core (
    ...
    input  wire [15:0] decay_gain,  // ← крутилка A2
    ...
);
wire signed [31:0] fb1_mul = $signed(tap1_raw) * $signed({1'b0, decay_gain[14:0]});
```

**Масштабування decay:** cap на 0.75 — при decay ≥ 1.0 кожне відбиття гучніше за попереднє, накопичення → клікання → хаос:

```verilog
wire [15:0] decay_uncapped = decay_raw + {3'b000, decay_raw[14:3]};  // ×1.125
wire [15:0] decay_gain = (decay_uncapped > 16'h6000) ? 16'h6000 : decay_uncapped;
```

### Pre-delay

Pre-delay — затримка перед тим як звук потрапляє в реверб. Імітує паузу між прямим звуком і першим відбиттям від стіни.

```verilog
wire [10:0] predelay_samples = predelay_raw[13:3];

delay_line #(.DEPTH(2048), .ADDR_BITS(11)) u_predelay (
    .clk(clk), .we(sample_rdy),
    .audio_in(left_cap),
    .delay_samples(predelay_samples),
    .audio_out(predelay_out)
);

wire signed [15:0] predelayed = (|predelay_samples) ? predelay_out : left_cap;
```

### Проблема: pre-delay не чутно

Закомітив. Перевірив на залізі. Decay (A2) — чутно. A3 — **нічого**.

Причина не в коді — у фізиці сприйняття. Pre-delay змінює *коли* приходить луна, але не *яка*. Коли reverb увімкнений — звук перетворюється на суцільну "кашу" відбиттів, і пауза перед нею непомітна на слух.

> Технічно правильно. Практично — марна трата крутилки.

---

## Частина 2: Bitcrusher

A3 некорисний при увімкненому reverb. Потрібен ефект який **завжди чутно**.

### Що таке Bitcrusher

Зменшення бітрозрядності аудіо. 16-бітний сигнал — 65536 значень, звук плавний. Залишити тільки 8 старших біт — 256 значень, між ними чутні "ступеньки":

```
16-bit:  ────────────── плавна хвиля
 8-bit:  ──┐ ┌──┐ ┌───  lo-fi хрипіння
 4-bit:  ──┘ └──┘ └───  відверта "пікселізація" звуку
```

В цифровому домені — просто AND-маска: обнулити молодші N біт:

```verilog
signal & 16'hFF00  // залишити тільки старші 8 біт
```

### Код

```verilog
wire [3:0] crush_bits = decay_raw[14:11];  // A2 → 0..12 обнулених біт

wire [15:0] crush_mask =
    (crush_bits ==  0) ? 16'hFFFF :  // чисто
    (crush_bits ==  4) ? 16'hFFF0 :  // 12-bit
    (crush_bits ==  8) ? 16'hFF00 :  //  8-bit ← помітне lo-fi
    (crush_bits == 11) ? 16'hF800 :  //  5-bit
                         16'hF000;   //  4-bit ← extreme

wire signed [15:0] reverb_out = mix_out & $signed(crush_mask);
```

Bitcrusher стоїть **після** wet/dry mix. Якщо поставити до reverb — ефект вимивається feedback-ом і майже не чутно.

---

## Ланцюжок сигналу (фінальний)

```
PCM1808 → left_cap
  → reverb_core (wet_gain=A1, decay_gain=A3)
  → wet/dry mix
  → & crush_mask (A2, bitcrusher)
  → × volume (A0)
  → CS4344
```

---

[← Фаза 4: Spring Reverb DSP](/posts/phase4-reverb/) | [Verilator: симуляція →](/posts/verilator-sim/)
