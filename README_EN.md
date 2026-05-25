# AI Management System

> An AI-powered smart shelf management system based on Canaan K230 (RT-Thread Smart). Uses camera + NPU for real-time image classification of shelf products, 4×4 matrix keypad for inventory management, OLED for count display, and LEDs for classification indication. Won **Provincial 3rd Prize** in the National Embedded Design Competition.

**[中文 →](README.md)**

---

## Features

- **AI Image Recognition** — Real-time camera capture + K230 NPU (KPU) inference to classify Type A / Type B shelf products
- **LED Status Indicator** — Classification result drives onboard LEDs: Type A lights LED1, Type B lights LED2
- **4×4 Matrix Keypad** — Keys 1-5 increment count, Key 6 resets to zero, with 50ms debounce algorithm
- **OLED Display** — SSD1306 OLED shows "A: [count]" and "B: [count]" in real-time
- **LCD OSD Overlay** — Displays classification result, confidence score, and system status

## Hardware

| Component | Model | Interface |
|------|------|------|
| MCU / AI Chip | Canaan K230 (Dual RISC-V) | — |
| OS | RT-Thread Smart | — |
| Runtime | CanMV-K230 MicroPython | — |
| Camera | Onboard OV5647 | MIPI CSI |
| Display (LCD) | Onboard LCD | — |
| OLED | SSD1306 128×64 | I2C (Software) |
| Keypad | 4×4 Matrix | GPIO |
| LED | Onboard LED ×2 | GPIO |

## Software Architecture

```
main.py
├── OLED Module          # SSD1306 driver + 8×8 font library
├── Image Recognition    # KPU model inference + AI2D preprocessing
├── Matrix Keypad        # 4×4 matrix scan + debounce
├── LED Control          # Classification result → LED states
└── Main System Loop     # Main loop scheduling + error handling
```

- **Technical Docs**: See [TECHNICAL.md](docs/TECHNICAL.md) — pin mapping, data flow, state machine, config format
- **AI Model**: Image classification model trained at [Kendryte Training Platform](https://www.kendryte.com/zh/training/start), exported as `.kmodel` for K230 KPU
- **Inference Engine**: nncase v2.9.0 runtime
- **Image Preprocessing**: AI2D hardware-accelerated scaling & format conversion

## Project Structure

```
AI-Management-System/
├── README.md                     # Project description (Chinese)
├── README_EN.md                  # Project description (English)
├── LICENSE                       # MIT License
├── .gitignore
├── .gitattributes
├── src/
│   ├── main.py                   # Main program (all modules)
│   └── i2c_ssd1306.py           # SSD1306 OLED driver
├── mp_deployment_source/         # MicroPython deployment resources
│   ├── deploy_config.json       #   Model deployment config
│   └── *.kmodel                 #   KPU model file
├── docs/
│   ├── TECHNICAL.md             #  Technical reference (pin mapping, data flow, state machine)
│   ├── project_report.pdf       #  Contest report
│   └── release-notes/           #  Release notes
└── test2.zip                     # Model training deployment package (offline)
```

## Getting Started

### Prerequisites

- **Hardware**: RT-Thread Smart AI Kit (Canaan K230 dev board)
  - [Buy on Taobao](https://e.tb.cn/h.SBGMFrRTHpvHlVG?tk=OTVafpM7qii)
  - [Board Documentation](https://www.kendryte.com/k230/zh/rtt/dev/index_rtt_only.html)
- **Firmware**: CanMV-K230 MicroPython image (nncase v2.9.0)
  - [Baidu Pan Download](https://pan.baidu.com/s/1os9wadhNvpo3ZObLgbENFQ) Password: `rtth`
  - Image file: `CanMV-K230_DONGSHANPI_micropython_local_nncase_v2.9.0.img.gz`
- **Software**: [CanMV IDE K230](https://www.kendryte.com/k230/)
- **Boot Mode**: SD card boot (BOOT0=OFF, BOOT1=OFF)

### Deployment Steps

1. Download CanMV-K230 firmware image from Baidu Pan and flash to an SD card
2. Copy `mp_deployment_source/` to the SD card root
3. Copy `src/main.py` and `src/i2c_ssd1306.py` to the SD card root
4. Insert SD card and power on the board
5. Open CanMV IDE K230, connect to device, paste `main.py` and run

### Model Training

1. Prepare an image classification dataset (photos of Type A & B products)
2. Upload to [Kendryte Training Platform](https://www.kendryte.com/zh/training/start) for online training
3. Download the deployment package (`test2.zip`), extract and replace the `mp_deployment_source/` directory

## License

MIT License — see [LICENSE](LICENSE) for details.

## Acknowledgments

- Canaan Technology K230 chip & SDK
- RT-Thread Smart OS
- CanMV MicroPython community
- Kendryte online AI training platform
