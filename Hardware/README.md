# 🔧 xiaoMing 硬件设计 · v1.2

> 语音灯箱主控板的原理图与 PCB 工程（立创EDA 专业版）
>
> _Author: @Hugokkl_

本目录存放 xiaoMing 的**硬件设计源文件**。板子以 **ASRPRO-CORE** 为核心，集成 USB-C 供电与下载、CH340 串口、3.3V LDO、麦克风、扬声器与指示灯，并把 ASRPRO 的串口 / IO 全部引出到连接器。

固件见 [`../Firmware/xiaoming.hd`](../Firmware/xiaoming.hd)，软件侧功能与串口协议见 [根目录 README](../README.md)。

---

## 📷 原理图

![xiaoMing v1.2 原理图](../Pics/51e14cbc650800e626c43d49330e3588.png)

> 原图：`Pics/51e14cbc650800e626c43d49330e3588.png`，1359 × 559 px。

---

## 📦 工程文件

| 文件 | 说明 |
| --- | --- |
| `ProPrj_xiaoMing-v1.2_2026-09-18.epro2` | 立创EDA 专业版工程包（原理图 + PCB + 器件库） |

**打开方式**：立创EDA 专业版（编辑器版本 **3.2.149** 及以上）→ 打开工程 → 选择 `.epro2` 文件。
该扩展名实为 ZIP 容器，内含 `project2.json`（工程元信息）、`xiaoMing.epru`（设计数据）与器件图片资源。

工程规模：**1 页原理图（SCH）** + **1 块 PCB**，含 49 个 DEVICE / 48 个 SYMBOL / 37 个 FOOTPRINT 定义。

---

## 🧩 板载功能分区

以下分区名称取自原理图上的框注：

| 分区 | 说明 |
| --- | --- |
| USB-C | USB Type-C 接口（`KH-TYPE-C-16P`），供电 + 数据 |
| CH340串口通信 | `CH340K` 完成 USB ↔ 串口转换 |
| Auto-Download | 由 `CH340K` 的 DTR / RTS 配合三极管实现 ASRPRO 免按键自动下载 |
| 3V3-LDO | `AP2112K-3.3` 产生 3.3V 轨（网络 `3.3V` / `VCC` / `5V` / `VBUS`） |
| Core | `ASRPRO-CORE` 主控核心板（`U1`） |
| 指示灯 | 电源红灯 + 用户灯（`PWR REDLIGHT` / `USER_LED`，均 0603） |
| 麦克风 | 驻极体麦克风 `GMI4015P-2C-50DB`，引出 `MIC+` / `MIC-` |
| 扬声器 | 引出 `SPK+` / `SPK-` |
| Interfaces | 对外连接器与测试点 |

板上的 ASRPRO IO 网络：`PA0`-`PA6`、`PB5`、`PB6`、`PC4`；串口网络：`USART0_TX/RX`、`USART1_TX/RX`、`USART2_TX/RX`。

---

## 🔌 对外接口

| 位号 | 封装 | 引出网络 | 用途 |
| --- | --- | --- | --- |
| CN1 | SH1.0 1×4P 卧贴 | `USART1_TX` / `USART1_RX` / `GND` | 串口一 → 固件 `Serial1`，接 [HOPE-Remote](https://github.com/ZhangKeLiang0627/HOPE-Remote) |
| CN3 | SH1.0 1×4P 卧贴 | `USART2_TX` / `USART2_RX` / `GND` | 串口二（原 433 模块口，协议合并后已不单独使用） |
| CN7 | SH1.0 1×4P 卧贴 | `RELAY0` / `RELAY1` | 继电器 / 外设控制 |
| CN8 | 1×3P 贴片 | `DATA` / `VCC` / `GND` | 数字外设（灯带类，`DATA` 走 `PA4`） |
| CN2 | 1.25mm 1×2P | `SPK+` / `SPK-` | 扬声器 |
| TP1-TP5 | 测试点 | — | 调试测量 |

> 上表网络归属据工程内网络标签整理，**引脚序号请以原理图为准**。部分连接器的个别引脚为隐藏网络标签，未在上表列出。

---

## 📋 主要器件

| 位号 | 型号 | 说明 |
| --- | --- | --- |
| U1 | ASRPRO-CORE | 语音识别主控核心板 |
| U13 | CH340K | USB 转串口，支持自动下载 |
| U6 | AP2112K-3.3TRG1 | 3.3V LDO |
| USB1 | KH-TYPE-C-16P | USB Type-C 16P 母座 |
| D1 | B5819WS | 肖特基二极管（SOD-323），**见下方注意事项** |
| D2 | USBLC6-2SC6 | USB 接口 ESD 保护 |
| Q1 | LP2301ELT1G | P 沟道 MOSFET |
| Q2 | SS8550 | PNP 三极管（自动下载电路） |
| LED1 | LED_G | 用户指示灯（绿，0603） |
| LED2 | LED_R | 电源指示灯（红，0603） |
| MIC2 | GMI4015P-2C-50DB | 驻极体麦克风 |
| SW1 | TS-1185-B-A-B-A | 轻触按键（复位 / 下载） |
| CN1 / CN3 / CN7 | A1002WR-S-4P | 长江连接器 SH 系列 1.0mm 1×4P 卧贴（LCSC `C239522`） |

### 阻容

| 位号 | 型号 | 备注（据型号解码） |
| --- | --- | --- |
| C37, C38, C43, C44 | CL05B104KB54PNC | 0402，0.1µF |
| C39, C40, C41, C42 | CL10A226MQ8NRNC | 0805，22µF |
| R5, R6, R26 | 0402WGF5101TCE | 0402，5.1kΩ |
| R20, R21 | 0402WGF2001TCE | 0402，2kΩ |
| R22 | 0402WGF1003TCE | 0402，100kΩ |
| R23 | 0402WGF4702TCE | 0402，47kΩ |
| R24 | 0402WGF3300TCE | 0402，330Ω |
| R25 | 0402WGF3301TCE | 0402，3.3kΩ |

---

## ⚠️ 设计注意事项

1. **D1 发热问题**（原理图上的原始批注）：

   > 此处二极管应该使用 0 欧电阻替代，否则发热严重

   即当前 `B5819WS` 位置在后续版本建议改用 0Ω 电阻。

2. **CN3 的用途已变化**：该口原本接独立的 433 学习/发射模块。红外与 433 合并到同一个串口后（见 [根目录 README · 串口协议](../README.md#串口协议)），固件只使用 `Serial1`（CN1），CN3 目前空闲。

---

## 🔗 与固件的对应关系

| 板载资源 | 固件中的使用 |
| --- | --- |
| CN1 / `USART1`（PA2 / PA3） | `Serial1.begin(115200)`，向 HOPE-Remote 发 `xxNNN` / `fsNNN` 指令 |
| `PA4` | `WS2812 ASR_WS2812(4)`，驱动灯箱灯带 |
| `SW1` | 配合 Auto-Download 电路进入下载模式 |

> 红外与 433 现已统一走串口一（CN1），槽号区间区分介质：红外 `000`-`095`、射频 `100`-`611`。

---

## 📝 版本

| 版本 | 日期 | 说明 |
| --- | --- | --- |
| v1.2 | 2026-09-18 | 当前工程版本（版本号与日期取自工程文件名） |

---

## 📄 License

[MIT](../LICENSE) © 2025 kkl
