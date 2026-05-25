# Technical Reference

> Hardware pin mapping, data flow, state machine, and configuration format for the AI Management System.

---

## Hardware Pin Mapping

### K230 FPIOA Pin Assignments

| GPIO | Function | Connected To | Direction | Notes |
|------|----------|-------------|-----------|-------|
| 44 | I2C SCL (Software) | SSD1306 OLED | Output | Software I2C via GPIO bit-bang |
| 45 | I2C SDA (Software) | SSD1306 OLED | Bidir | Software I2C via GPIO bit-bang |
| 28 | ROW1 | Matrix Keypad | Input (Pull-Down) | — |
| 29 | ROW2 | Matrix Keypad | Input (Pull-Down) | — |
| 30 | ROW3 | Matrix Keypad | Input (Pull-Down) | — |
| 31 | ROW4 | Matrix Keypad | Input (Pull-Down) | — |
| 18 | COL1 | Matrix Keypad | Output | Column scan driver |
| 19 | COL2 | Matrix Keypad | Output | Column scan driver |
| 33 | COL3 | Matrix Keypad | Output | Column scan driver |
| 35 | COL4 | Matrix Keypad | Output | Column scan driver |
| 42 | LED1 (Red) | Onboard LED | Output | Active HIGH, class1 indicator |
| 6 | LED2 (Green) | Onboard LED | Output | Active HIGH, class2 indicator |

### I2C Bus

| Parameter | Value |
|-----------|-------|
| Mode | Software (bit-bang) |
| Bus ID | `I2C(5)` |
| SCL | GPIO 44 |
| SDA | GPIO 45 |
| Frequency | 400 kHz |
| Slave Device | SSD1306 @ `0x3C` |

### Camera & Display

| Peripheral | Model | Interface | Channel | Resolution |
|------------|-------|-----------|---------|------------|
| Camera | OV5647 | MIPI CSI | CHN 0 (Display), CHN 2 (AI) | 640×360 (AI), 800×480 (LCD) |
| LCD | ST7701 | RGB | — | 800×480 (default) |
| LCD (alt) | LT9611 | HDMI | — | 1920×1080 |

---

## System Architecture

### Data Flow

```
OV5647 Camera
    │
    ├─ CHN 0 (YUV) ──────────────────▶ Display Layer (VIDEO1) ──▶ LCD
    │
    └─ CHN 2 (RGB888P) ──▶ AI2D Preprocess ──▶ KPU Inference
                                                      │
                                                      ▼
                                              Post-process
                                              (softmax / sigmoid)
                                                      │
                                              ┌───────┴───────┐
                                              │               │
                                              ▼               ▼
                                         LED Control     LCD OSD (LAYER_OSD3)
                                         (LED1 / LED2)   (result + confidence)
                                              │
                                              ▼
                                         4×4 Keypad
                                    (increment / reset)
                                              │
                                              ▼
                                         SSD1306 OLED
                                    ("A: [n]  B: [n]")
```

### Module Interaction

```
main() Main Loop
    │
    ├──▶ detect_key()           ──▶ update_oled_display()
    │        │
    │        └── GPIO scan (COL output → ROW input)
    │             50ms debounce via ticks_ms()
    │
    ├──▶ sensor.snapshot(CHN_2) ──▶ ai2d_builder.run()
    │                                    │
    │                                    └── Resize: 640×360 → 224×224
    │                                        NCHW_FMT, bilinear, half_pixel
    │
    ├──▶ kpu.run()              ──▶ softmax / sigmoid → cls_idx, score
    │
    ├──▶ control_leds(cls_idx)  ──▶ LED1.on() / LED2.on()
    │
    └──▶ Display.show_image()   ──▶ OSD text overlay on LCD
```

---

## Module Details

### 1. OLED Display (SSD1306 128×64)

**Protocol**: I2C, software bit-bang on GPIO 44/45.

The driver communicates with SSD1306 via a command/data prefix byte:
- `0x00` prefix → command byte follows
- `0x40` prefix → data byte(s) follow

**Initialization sequence** (refer to SSD1306 datasheet):

| Step | Command | Description |
|------|---------|-------------|
| 1 | `0xAE` | Display OFF |
| 2 | `0xA8, 0x3F` | MUX ratio = 64 |
| 3 | `0xD3, 0x00` | Display offset = 0 |
| 4 | `0x40` | Start line = 0 |
| 5 | `0xA1` | Segment re-map (normal) |
| 6 | `0xC0` | COM scan direction (normal) |
| 7 | `0xDA, 0x12` | COM pins config |
| 8 | `0x81, 0x7F` | Contrast = max |
| 9 | `0xA4` | Resume to RAM content |
| 10 | `0xA6` | Normal (non-inverted) display |
| 11 | `0xD5, 0x80` | Oscillator frequency |
| 12 | `0x8D, 0x14` | Charge pump enable |
| 13 | `0xAF` | Display ON |

**Character rendering**: Uses an 8×8 bitmap font (`FONT_8x8`, 13 glyphs: `0-9`, `A`, `B`, `:`, Space). Each glyph is 8 bytes, one per row. `oled_show_text()` sets page + column address, then writes 8 data bytes per character. Display is organized as 8 pages × 128 columns.

**API**:

| Function | Parameters | Description |
|----------|-----------|-------------|
| `oled_init()` | — | Send init sequence, turn display on |
| `oled_clear()` | — | Write `0x00` to all 8×128 pixels |
| `oled_show_text(text, page, start_col)` | `text: str`, `page: int (0-7)`, `start_col: int (0-127)` | Render text at page/column |
| `update_oled_display(value_A, value_B)` | `value_A: int`, `value_B: int` | Clear screen, show "A: n" on page 0 and "B: n" on page 1 |

### 2. Image Recognition (KPU + AI2D)

**Pipeline**:

1. **Capture**: `sensor.snapshot(chn=CAM_CHN_ID_2)` returns RGB888P image at 640×360
2. **Preprocess** (AI2D hardware): Resize 640×360 → 224×224, NCHW format, bilinear interpolation
3. **Inference** (KPU): Load `.kmodel`, set input tensor, run, get output tensors
4. **Post-process**: Apply softmax (multi-class) or sigmoid (binary) activation, threshold by `confidence_threshold`

**Model format**: `.kmodel` (nncase v2.9.0 compiled), trained externally at Kendryte training platform.

**Post-processing logic**:

```
if num_classes > 2:
    softmax → argmax → check > confidence_threshold
else:
    sigmoid → if > threshold: cls=1, else: cls=0
```

If confidence is below threshold, `cls_idx = -1` (unrecognized).

**Key parameters** (from `deploy_config.json`):

| Parameter | Typical Value | Description |
|-----------|--------------|-------------|
| `img_size` | `[224, 224]` | Model input resolution |
| `confidence_threshold` | `0.5` | Minimum confidence for positive classification |
| `num_classes` | `2` | Number of categories |
| `categories` | `["class1", "class2"]` | Label names |

### 3. Matrix Keypad (4×4)

**Scanning algorithm** (column strobe):

1. Pull all 4 COL pins LOW
2. Assert one COL HIGH at a time
3. For each COL, check all 4 ROW pins
4. If a ROW reads HIGH → key at `[row][col]` is pressed

**Debounce**: Record timestamp via `time.ticks_ms()`. Only register a keypress when the same key is detected twice with >50ms interval, and the key was not already in the "pressed" state.

**Key mapping**:

| | COL1 | COL2 | COL3 | COL4 |
|---|------|------|------|------|
| **ROW1** | 1 | 2 | 3 | 4 |
| **ROW2** | 5 | 6 | 7 | 8 |
| **ROW3** | 9 | 10 | 11 | 12 |
| **ROW4** | 13 | 14 | 15 | 16 |

**Business logic**: Keys 1-5 increment the current category's count by their value (1-5), Key 6 resets to 0. Which category (A or B) is updated depends on the most recent classification result (`cls_idx`).

### 4. LED Control

Simple GPIO output control driven by classification result:

| `cls_idx` | LED1 (GPIO 42) | LED2 (GPIO 6) | Meaning |
|-----------|----------------|---------------|---------|
| 0 (class1) | ON | OFF | Category A detected |
| 1 (class2) | OFF | ON | Category B detected |
| -1 (unknown) | OFF | OFF | No confident detection |

LEDs only update when `cls_idx` changes (`cls_idx != last_class`) to avoid redundant GPIO writes.

---

## Main System Loop

### State Machine

```
                    ┌─────────────┐
                    │   STARTUP   │
                    │ OLED / LED  │
                    │ KPU / Cam   │
                    │ Keypad init │
                    └──────┬──────┘
                           │
                           ▼
                 ┌──────────────────┐
          ┌─────│   IDLE (loop)    │◀──────────────┐
          │     └────────┬─────────┘               │
          │              │                          │
          │     ┌───────┴───────┐                  │
          │     │               │                  │
          │     ▼               ▼                  │
          │ ┌────────┐  ┌──────────────┐          │
          │ │ KEY    │  │ CAMERA       │          │
          │ │ SCAN   │  │ CAPTURE      │          │
          │ └───┬────┘  └──────┬───────┘          │
          │     │              │                   │
          │     ▼              ▼                   │
          │ ┌────────┐  ┌──────────────┐          │
          │ │ DEBOUNCE│  │ AI2D + KPU   │          │
          │ │ (50ms)  │  │ INFERENCE    │          │
          │ └───┬────┘  └──────┬───────┘          │
          │     │              │                   │
          │     ▼              ▼                   │
          │ ┌────────┐  ┌──────────────┐          │
          │ │ UPDATE │  │ POST-PROCESS │          │
          │ │ A / B  │  │ (softmax/    │          │
          │ │ VALUE  │  │  sigmoid)    │          │
          │ └───┬────┘  └──────┬───────┘          │
          │     │              │                   │
          │     ▼              ▼                   │
          │ ┌────────┐  ┌──────────────┐          │
          │ │ OLED   │  │ LED CONTROL  │          │
          │ │ REFRESH│  │ (if changed) │          │
          │ └───┬────┘  └──────┬───────┘          │
          │     │              │                   │
          │     ▼              ▼                   │
          │ ┌─────────────────────────────┐       │
          │ │   LCD OSD UPDATE            │       │
          │ │   (result / conf / A B)     │       │
          │ └──────────────┬──────────────┘       │
          │                │                      │
          │                ▼                      │
          │ ┌─────────────────────────────┐       │
          │ │   gc.collect()              │       │
          │ │   (memory cleanup)          │───────┘
          │ └─────────────────────────────┘
          │
          ▼
    ┌──────────┐
    │ SHUTDOWN │  (Ctrl+C / exception)
    │ camera   │
    │ display  │
    │ LED off  │
    │ OLED off │
    └──────────┘
```

### Error Handling

| Exception | Behavior |
|-----------|----------|
| `deploy_config.json` not found or invalid JSON | Print error, `return` from `main()` |
| `KeyboardInterrupt` (Ctrl+C) | Print message, enter `finally` cleanup |
| Runtime exception (any) | Print traceback, enter `finally` cleanup |
| Image capture returns `None` or wrong format | Skip inference for this cycle |

### Cleanup Sequence (finally block)

1. Delete AI2D input/output tensors (if allocated)
2. `sensor.stop()` — stop camera streaming
3. `Display.deinit()` — release LCD display
4. `MediaManager.deinit()` — release media subsystem
5. `control_leds(-1)` — turn both LEDs off
6. `oled_clear()` — clear OLED screen

---

## Configuration Reference

### deploy_config.json

Located at `/sdcard/mp_deployment_source/deploy_config.json`. Generated by the Kendryte training platform.

```json
{
    "chip_type": "k230",
    "inference_width": 224,
    "inference_height": 224,
    "confidence_threshold": 0.5,
    "nncase_version": "2.9.0",
    "model_type": "can2",
    "img_size": [224, 224],
    "mean": [0.485, 0.456, 0.406],
    "std": [0.229, 0.224, 0.225],
    "categories": ["class1", "class2"],
    "kmodel_path": "can2_10.0l_20250706152359.kmodel",
    "num_classes": 2
}
```

| Field | Type | Description |
|-------|------|-------------|
| `chip_type` | string | Target chip, always `"k230"` |
| `inference_width` | int | Model input width (px) |
| `inference_height` | int | Model input height (px) |
| `confidence_threshold` | float | 0.0–1.0, minimum confidence for positive result |
| `nncase_version` | string | nncase compiler version used |
| `model_type` | string | Model architecture type |
| `img_size` | `[int, int]` | `[width, height]` of model input |
| `mean` | `[float, float, float]` | Per-channel normalization mean (RGB) |
| `std` | `[float, float, float]` | Per-channel normalization std (RGB) |
| `categories` | `[string, ...]` | Class label names, index matches model output |
| `kmodel_path` | string | `.kmodel` filename (relative to config dir) |
| `num_classes` | int | Number of output classes |

---

## Memory Management

The K230 has limited RAM. The main loop takes these measures:

- **Reuse tensors**: `ai2d_input_tensor` and `ai2d_output_tensor` are allocated once before the loop and reused
- **Explicit delete**: After reading KPU output, `del output_data` immediately frees the tensor reference
- **Nullify references**: `rgb888p_img = None` at end of each loop iteration to allow GC
- **Periodic GC**: `gc.collect()` called every loop iteration to reclaim short-lived MicroPython objects
- **Single OSD image**: `osd_img` is allocated once (800×480×4 = ~1.5MB ARGB8888) and cleared with `.clear()` rather than reallocated
