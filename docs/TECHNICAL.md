# 技术参考文档

> AI 智能货架管理系统的硬件引脚映射、数据流、状态机和配置格式参考。

**[English →](https://github.com/w4ysonch/AI-Management-System/blob/master/docs/TECHNICAL_EN.md)**

---

## 硬件引脚映射

### K230 FPIOA 引脚分配

| GPIO | 功能 | 连接外设 | 方向 | 备注 |
|------|------|---------|------|------|
| 44 | I2C SCL（软件） | SSD1306 OLED | 输出 | 软件 I2C GPIO 模拟 |
| 45 | I2C SDA（软件） | SSD1306 OLED | 双向 | 软件 I2C GPIO 模拟 |
| 28 | ROW1 | 矩阵按键 | 输入（下拉） | — |
| 29 | ROW2 | 矩阵按键 | 输入（下拉） | — |
| 30 | ROW3 | 矩阵按键 | 输入（下拉） | — |
| 31 | ROW4 | 矩阵按键 | 输入（下拉） | — |
| 18 | COL1 | 矩阵按键 | 输出 | 列扫描驱动 |
| 19 | COL2 | 矩阵按键 | 输出 | 列扫描驱动 |
| 33 | COL3 | 矩阵按键 | 输出 | 列扫描驱动 |
| 35 | COL4 | 矩阵按键 | 输出 | 列扫描驱动 |
| 42 | LED1（红） | 板载 LED | 输出 | 高电平有效，A 类指示 |
| 6 | LED2（绿） | 板载 LED | 输出 | 高电平有效，B 类指示 |

### I2C 总线

| 参数 | 值 |
|------|-----|
| 模式 | 软件模拟（bit-bang） |
| 总线 ID | `I2C(5)` |
| SCL | GPIO 44 |
| SDA | GPIO 45 |
| 频率 | 400 kHz |
| 从设备 | SSD1306 @ `0x3C` |

### 摄像头与显示器

| 外设 | 型号 | 接口 | 通道 | 分辨率 |
|------|------|------|------|--------|
| 摄像头 | OV5647 | MIPI CSI | CHN 0（显示）, CHN 2（AI） | 640×360（AI）, 800×480（LCD） |
| LCD | ST7701 | RGB | — | 800×480（默认） |
| LCD（备选） | LT9611 | HDMI | — | 1920×1080 |

---

## 系统架构

### 数据流

```
OV5647 摄像头
    │
    ├─ CHN 0 (YUV) ──────────────────▶ 显示层 (VIDEO1) ──▶ LCD
    │
    └─ CHN 2 (RGB888P) ──▶ AI2D 预处理 ──▶ KPU 推理
                                                │
                                                ▼
                                          后处理
                                        (softmax / sigmoid)
                                                │
                                         ┌──────┴──────┐
                                         │              │
                                         ▼              ▼
                                     LED 控制      LCD OSD (LAYER_OSD3)
                                   (LED1 / LED2)   (结果 + 置信度)
                                         │
                                         ▼
                                    4×4 矩阵按键
                                    (增加 / 清零)
                                         │
                                         ▼
                                   SSD1306 OLED
                                  ("A: [n]  B: [n]")
```

### 模块交互

```
main() 主循环
    │
    ├──▶ detect_key()           ──▶ update_oled_display()
    │        │
    │        └── GPIO 扫描（列输出 → 行输入）
    │             50ms 防抖 (ticks_ms())
    │
    ├──▶ sensor.snapshot(CHN_2) ──▶ ai2d_builder.run()
    │        │
    │        └── 缩放: 640×360 → 224×224
    │            NCHW_FMT, 双线性插值, half_pixel
    │
    ├──▶ kpu.run()              ──▶ softmax / sigmoid → cls_idx, score
    │
    ├──▶ control_leds(cls_idx)  ──▶ LED1.on() / LED2.on()
    │
    └──▶ Display.show_image()   ──▶ LCD OSD 文字叠加
```

---

## 模块详解

### 1. OLED 显示（SSD1306 128×64）

**协议**: I2C，GPIO 44/45 软件模拟。

驱动通过命令/数据前缀字节与 SSD1306 通信：
- `0x00` 前缀 → 后续为命令字节
- `0x40` 前缀 → 后续为数据字节

**初始化序列**（参考 SSD1306 数据手册）：

| 步骤 | 命令 | 说明 |
|------|------|------|
| 1 | `0xAE` | 关闭显示 |
| 2 | `0xA8, 0x3F` | MUX 比例 = 64 |
| 3 | `0xD3, 0x00` | 显示偏移 = 0 |
| 4 | `0x40` | 起始行 = 0 |
| 5 | `0xA1` | 段重映射（正常方向） |
| 6 | `0xC0` | COM 扫描方向（正常） |
| 7 | `0xDA, 0x12` | COM 引脚配置 |
| 8 | `0x81, 0x7F` | 对比度 = 最大 |
| 9 | `0xA4` | 恢复 RAM 内容显示 |
| 10 | `0xA6` | 正常显示（非反色） |
| 11 | `0xD5, 0x80` | 振荡器频率 |
| 12 | `0x8D, 0x14` | 使能电荷泵 |
| 13 | `0xAF` | 开启显示 |

**字符渲染**: 使用 8×8 点阵字库（`FONT_8x8`，13 个字形：`0-9`、`A`、`B`、`:`、空格）。每个字形 8 字节，每字节对应一行像素。`oled_show_text()` 先设置页地址和列地址，再逐字符写入 8 字节数据。屏幕分为 8 页 × 128 列。

**API**:

| 函数 | 参数 | 说明 |
|------|------|------|
| `oled_init()` | — | 发送初始化序列，打开显示 |
| `oled_clear()` | — | 向全部 8×128 像素写入 `0x00` |
| `oled_show_text(text, page, start_col)` | `text: str`，`page: int (0-7)`，`start_col: int (0-127)` | 在指定页/列位置渲染文本 |
| `update_oled_display(value_A, value_B)` | `value_A: int`，`value_B: int` | 清屏，在第 0 页显示 "A: n"，第 1 页显示 "B: n" |

### 2. 图像识别（KPU + AI2D）

**处理流水线**:

1. **采集**: `sensor.snapshot(chn=CAM_CHN_ID_2)` 获取 640×360 的 RGB888P 图像
2. **预处理**（AI2D 硬件）: 缩放 640×360 → 224×224，NCHW 格式，双线性插值
3. **推理**（KPU）: 加载 `.kmodel`，设置输入张量，运行，获取输出张量
4. **后处理**: 应用 softmax（多分类）或 sigmoid（二分类）激活函数，按 `confidence_threshold` 阈值过滤

**模型格式**: `.kmodel`（nncase v2.9.0 编译），在 Kendryte 训练平台在线训练导出。

**后处理逻辑**:

```
if num_classes > 2:
    softmax → argmax → 检查 > confidence_threshold
else:
    sigmoid → 若 > threshold: cls=1, 否则: cls=0
```

若置信度低于阈值，`cls_idx = -1`（未识别）。

**关键参数**（来自 `deploy_config.json`）：

| 参数 | 典型值 | 说明 |
|------|--------|------|
| `img_size` | `[224, 224]` | 模型输入分辨率 |
| `confidence_threshold` | `0.5` | 判定为有效识别的最低置信度 |
| `num_classes` | `2` | 分类类别数量 |
| `categories` | `["class1", "class2"]` | 类别标签名称 |

### 3. 矩阵按键（4×4）

**扫描算法**（列选通法）:

1. 将全部 4 个列引脚拉低
2. 依次将一列拉高
3. 每拉高一列，检测全部 4 个行引脚
4. 若某行读到高电平 → `[row][col]` 位置的按键被按下

**防抖**: 通过 `time.ticks_ms()` 记录时间戳。仅当同一按键间隔 >50ms 被两次检测到，且该按键未处于 "已按下" 状态时，才确认一次有效按键。

**按键映射**:

| | COL1 | COL2 | COL3 | COL4 |
|---|------|------|------|------|
| **ROW1** | 1 | 2 | 3 | 4 |
| **ROW2** | 5 | 6 | 7 | 8 |
| **ROW3** | 9 | 10 | 11 | 12 |
| **ROW4** | 13 | 14 | 15 | 16 |

**业务逻辑**: 按键 1-5 按对应数值增加当前类别的计数，按键 6 清零。更新 A 还是 B 取决于最近一次分类结果（`cls_idx`）。

### 4. LED 控制

由分类结果驱动的简单 GPIO 输出控制：

| `cls_idx` | LED1（GPIO 42） | LED2（GPIO 6） | 含义 |
|-----------|-----------------|----------------|------|
| 0（class1） | 亮 | 灭 | 检测到 A 类 |
| 1（class2） | 灭 | 亮 | 检测到 B 类 |
| -1（未知） | 灭 | 灭 | 无有效识别 |

LED 仅在 `cls_idx` 变化时更新（`cls_idx != last_class`），避免冗余 GPIO 写入。

---

## 主系统循环

### 状态机

```
                    ┌──────────────┐
                    │   启动初始化   │
                    │ OLED / LED   │
                    │ KPU / 摄像头  │
                    │ 按键初始化    │
                    └──────┬───────┘
                           │
                           ▼
                 ┌──────────────────┐
          ┌─────│   空闲循环        │◀──────────────┐
          │     └────────┬─────────┘               │
          │              │                          │
          │     ┌───────┴───────┐                  │
          │     │               │                  │
          │     ▼               ▼                  │
          │ ┌────────┐  ┌──────────────┐          │
          │ │ 按键   │  │ 摄像头       │          │
          │ │ 扫描   │  │ 图像采集     │          │
          │ └───┬────┘  └──────┬───────┘          │
          │     │              │                   │
          │     ▼              ▼                   │
          │ ┌────────┐  ┌──────────────┐          │
          │ │ 防抖   │  │ AI2D + KPU   │          │
          │ │(50ms)  │  │ 推理         │          │
          │ └───┬────┘  └──────┬───────┘          │
          │     │              │                   │
          │     ▼              ▼                   │
          │ ┌────────┐  ┌──────────────┐          │
          │ │ 更新   │  │ 后处理       │          │
          │ │ A / B  │  │ (softmax/    │          │
          │ │ 数值   │  │  sigmoid)    │          │
          │ └───┬────┘  └──────┬───────┘          │
          │     │              │                   │
          │     ▼              ▼                   │
          │ ┌────────┐  ┌──────────────┐          │
          │ │ OLED   │  │ LED 控制     │          │
          │ │ 刷新   │  │ (仅变化时)    │          │
          │ └───┬────┘  └──────┬───────┘          │
          │     │              │                   │
          │     ▼              ▼                   │
          │ ┌─────────────────────────────┐       │
          │ │   LCD OSD 更新              │       │
          │ │   (结果 / 置信度 / A B)      │       │
          │ └──────────────┬──────────────┘       │
          │                │                      │
          │                ▼                      │
          │ ┌─────────────────────────────┐       │
          │ │   gc.collect()              │       │
          │ │   (内存回收)                 │───────┘
          │ └─────────────────────────────┘
          │
          ▼
    ┌──────────┐
    │ 安全退出  │  (Ctrl+C / 异常)
    │ 关闭摄像头 │
    │ 释放显示器 │
    │ 关闭 LED  │
    │ 清空 OLED │
    └──────────┘
```

### 异常处理

| 异常 | 处理方式 |
|------|----------|
| `deploy_config.json` 不存在或 JSON 格式错误 | 打印错误，`return` 退出 `main()` |
| `KeyboardInterrupt`（Ctrl+C） | 打印提示，进入 `finally` 清理 |
| 运行时异常（任意） | 打印错误信息，进入 `finally` 清理 |
| 图像采集返回 `None` 或格式不符 | 跳过本轮推理 |

### 清理流程（finally 块）

1. 删除 AI2D 输入/输出张量（如已分配）
2. `sensor.stop()` — 停止摄像头流
3. `Display.deinit()` — 释放 LCD 显示器
4. `MediaManager.deinit()` — 释放媒体子系统
5. `control_leds(-1)` — 关闭两颗 LED
6. `oled_clear()` — 清空 OLED 屏幕

---

## 配置参考

### deploy_config.json

位于 `/sdcard/mp_deployment_source/deploy_config.json`，由 Kendryte 训练平台生成。

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

| 字段 | 类型 | 说明 |
|------|------|------|
| `chip_type` | string | 目标芯片，固定为 `"k230"` |
| `inference_width` | int | 模型输入宽度（像素） |
| `inference_height` | int | 模型输入高度（像素） |
| `confidence_threshold` | float | 0.0–1.0，判定为有效识别的最低置信度 |
| `nncase_version` | string | 使用的 nncase 编译器版本 |
| `model_type` | string | 模型架构类型 |
| `img_size` | `[int, int]` | 模型输入的 `[width, height]` |
| `mean` | `[float, float, float]` | 逐通道归一化均值（RGB） |
| `std` | `[float, float, float]` | 逐通道归一化标准差（RGB） |
| `categories` | `[string, ...]` | 类别标签名称，索引对应模型输出 |
| `kmodel_path` | string | `.kmodel` 文件名（相对 config 所在目录） |
| `num_classes` | int | 输出类别数量 |

---

## 内存管理

K230  RAM 有限，主循环采取以下策略：

- **张量复用**: `ai2d_input_tensor` 和 `ai2d_output_tensor` 在循环外分配一次，循环内复用
- **显式释放**: 读取 KPU 输出后立即 `del output_data` 释放张量引用
- **引用置空**: 每轮循环末尾 `rgb888p_img = None`，便于 GC 回收
- **定期 GC**: 每轮循环调用 `gc.collect()` 回收 MicroPython 短生命周期对象
- **单例 OSD 图像**: `osd_img` 仅分配一次（800×480×4 ≈ 1.5MB ARGB8888），通过 `.clear()` 刷新而非重新分配
