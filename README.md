# AI Management System / 智能货架管理系统

> 基于嘉楠 K230 (RT-Thread Smart) 的 AI 智能货架管理系统。通过摄像头 + NPU 图像分类识别货架商品，4×4 矩阵按键管理库存数量，OLED 显示 A/B 类商品计数，LED 指示分类结果。获全国大学生嵌入式竞赛**省三等奖**。

**[English →](README_EN.md)**

---

## 功能

- **AI 图像识别** — 摄像头实时采集 + K230 NPU（KPU）推理，识别货架上 A/B 两类商品
- **LED 状态指示** — 分类结果驱动板载 LED：A 类亮 LED1，B 类亮 LED2
- **4×4 矩阵按键** — 1-5 键增加库存计数，6 键清零，带 50ms 按键防抖
- **OLED 显示** — SSD1306 OLED 实时显示 "A: [值]" 和 "B: [值]"
- **LCD 屏幕 OSD** — 叠加显示分类结果、置信度和系统状态

## 硬件

| 组件 | 型号 | 接口 |
|------|------|------|
| 主控 / AI 芯片 | 嘉楠 K230 (RISC-V 双核) | — |
| 操作系统 | RT-Thread Smart | — |
| 运行环境 | CanMV-K230 MicroPython | — |
| 摄像头 | 板载 OV5647 | MIPI CSI |
| 显示屏(LCD) | 板载 LCD | — |
| OLED | SSD1306 128×64 | I2C (软件) |
| 按键 | 4×4 矩阵键盘 | GPIO |
| LED | 板载 LED ×2 | GPIO |

## 软件架构

```
main.py
├── OLED 显示模块        # SSD1306 驱动 + 8×8 字模
├── 图像识别模块          # KPU 模型推理 + AI2D 预处理
├── 矩阵按键模块          # 4×4 矩阵扫描 + 防抖
├── LED 控制模块          # 分类结果 → LED 状态
└── 主系统集成            # 主循环调度 + 异常处理
```

- **技术文档**: 详见 [TECHNICAL.md](docs/TECHNICAL.md) — 引脚映射、数据流、状态机、配置格式
- **AI 模型**: 图像分类模型在 [Kendryte 训练网站](https://www.kendryte.com/zh/training/start) 训练，导出为 `.kmodel` 格式供 K230 KPU 运行
- **推理框架**: nncase v2.9.0 runtime
- **图像预处理**: AI2D 硬件加速缩放/格式转换

## 项目结构

```
AI-Management-System/
├── README.md                     # 项目说明（中文）
├── README_EN.md                  # Project description (English)
├── LICENSE                       # MIT License
├── .gitignore
├── .gitattributes
├── src/
│   ├── main.py                   # 主程序（含所有模块）
│   └── i2c_ssd1306.py           # SSD1306 OLED 驱动
├── docs/
│   ├── TECHNICAL.md             #  技术参考文档（引脚映射、数据流、状态机）
│   ├── TECHNICAL_EN.md          #  Technical reference (English)
│   ├── project_report.pdf       #  竞赛报告
│   └── release-notes/           #  版本发布说明
```

## 运行方式

### 前置条件

- **硬件**: RT-Thread Smart AI 套件（嘉楠 K230 开发板）
  - [淘宝购买](https://e.tb.cn/h.SBGMFrRTHpvHlVG?tk=OTVafpM7qii)
  - [开发板资料](https://www.kendryte.com/k230/zh/rtt/dev/index_rtt_only.html)
- **固件**: CanMV-K230 MicroPython 镜像（nncase v2.9.0）
  - [百度网盘下载](https://pan.baidu.com/s/1os9wadhNvpo3ZObLgbENFQ) 提取码: `rtth`
  - 镜像文件: `CanMV-K230_DONGSHANPI_micropython_local_nncase_v2.9.0.img.gz`
- **软件**: [CanMV IDE K230](https://www.kendryte.com/k230/)
- **启动方式**: SD 卡启动（BOOT0=OFF, BOOT1=OFF）

### 部署步骤

1. 从百度网盘下载 CanMV-K230 镜像，烧录到 SD 卡
2. 将 `mp_deployment_source/` 文件夹复制到 SD 卡根目录
3. 将 `src/main.py` 和 `src/i2c_ssd1306.py` 复制到 SD 卡根目录
4. 插入 SD 卡，开发板上电
5. 打开 CanMV IDE K230，连接设备，粘贴 `main.py` 代码并运行

### 模型训练

1. 准备图像分类数据集（A/B 两类商品照片）
2. 上传至 [Kendryte 训练网站](https://www.kendryte.com/zh/training/start) 进行在线训练
3. 下载部署包 `test2.zip`，解压后替换 `mp_deployment_source/` 目录

## 许可证

MIT License — 详见 [LICENSE](LICENSE)。

## 致谢

- 嘉楠科技 K230 芯片及 SDK
- RT-Thread Smart 操作系统
- CanMV MicroPython 社区
- Kendryte 在线 AI 训练平台
