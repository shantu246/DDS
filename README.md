# DDS (Direct Digital Synthesis) Project | DDS 直接数字频率合成项目

> FPGA + MCU co-design for configurable sine-wave generation using UART frequency control.  
> 基于 FPGA + MCU 协同设计的可配置正弦波发生系统，通过 UART 实现频率控制。

## 1. Project Overview | 项目概述

This repository contains a complete DDS signal-generation solution split into two parts:
- **FPGA design** (`DDS_FPGA_5_21`): implements DDS core, UART frequency parsing, 7-segment display, and DAC clock generation.
- **MCU firmware** (`DDS_MCU_5_21`): reads keypad input, shows UI on OLED, converts user input to ASCII frequency string, and sends it to FPGA.

本仓库包含一个完整的 DDS 信号发生方案，分为两部分：
- **FPGA 工程**（`DDS_FPGA_5_21`）：实现 DDS 核心、UART 频率解析、数码管显示与 DAC 时钟生成。
- **MCU 固件**（`DDS_MCU_5_21`）：读取矩阵键盘输入、在 OLED 上显示界面、将用户输入转换为 ASCII 频率字符串并发送给 FPGA。

---

## 2. Repository Structure | 仓库结构

```text
DDS/
├── DDS_FPGA_5_21/
│   ├── DDS_simple.v           # Top module
│   ├── MCU_uart.v             # UART receiver + ASCII to frequency parser
│   ├── dds_freq_ctrlword.v    # Frequency to phase increment mapping
│   ├── seg_display.v          # 7-segment display scan and decode
│   ├── sine_12bit_256.*       # 256-point sine LUT IP
│   ├── dac904_pll.*           # PLL IP for DAC clock
│   └── DDS_simple.qsf/.qpf    # Quartus project files
└── DDS_MCU_5_21/
    ├── User/main.c            # Main control logic
    ├── User/config.syscfg     # TI SysConfig source
    ├── Project/DDS_5_16.uvprojx # Keil uVision project
    ├── BSP/                   # Keyboard/OLED/Flash drivers
    └── Source/                # Generated config and TI/3rd-party code
```

---

## 3. System Architecture | 系统架构

### Signal Path | 信号链路
1. User enters frequency on keypad (MCU side).  
   用户通过键盘输入目标频率（MCU 侧）。
2. MCU converts frequency to ASCII text (e.g. `1000\n`) and transmits via UART1 @ 115200 bps.  
   MCU 将频率转换成 ASCII 文本（如 `1000\n`），通过 UART1（115200）发送。
3. FPGA UART module parses digits and updates `freq_out` when newline (`\n`/`\r`) arrives.  
   FPGA 串口模块接收 ASCII 数字，在接收到换行（`\n`/`\r`）后更新 `freq_out`。
4. DDS core converts target frequency into phase increment (`freq_ctrl_word`).  
   DDS 核心将目标频率映射为相位步进字（`freq_ctrl_word`）。
5. Phase accumulator drives sine LUT to output waveform samples to DAC interface (`sine_out[13:0]`).  
   相位累加器驱动正弦查找表输出波形采样到 DAC 接口（`sine_out[13:0]`）。
6. Frequency summary is shown on 7-segment display when enabled.  
   在使能条件满足时，数码管显示频率摘要。

### Control Mode | 控制模式
- `switch = 1`: FPGA uses fixed **1 kHz** output and lights LED.
- `switch = 0`: FPGA uses frequency from MCU UART.

- `switch = 1`：FPGA 使用固定 **1 kHz** 输出，同时点亮 LED。  
- `switch = 0`：FPGA 使用 MCU 串口发送的频率。

---

## 4. FPGA Design Notes | FPGA 设计说明

### Top Module (`DDS_simple.v`)
- System clock input: 50 MHz.
- Integrates UART parser, DDS control-word generation, phase accumulator, sine ROM, PLL, and display module.
- Uses top 8 bits of 32-bit phase accumulator as LUT address (`phase[31:24]`).

### UART Frequency Parser (`MCU_uart.v`)
- UART: 115200 bps.
- Input format: ASCII decimal string terminated by `\n` or `\r`.
- Accepts frequencies `>= 100` before committing update.
- `mcu_active` flag enables display after first valid message.

### Frequency Control Mapping (`dds_freq_ctrlword.v`)
- Current implementation uses approximation:
  - `freq_ctrl_word = freq_target * 86`
- This is a resource-friendly approximation to DDS scaling and may introduce small frequency error.

### Display (`seg_display.v`)
- Dynamically scans 4-digit 7-segment display.
- Displays compact frequency format based on magnitude.
- Display enable controlled by: `switch || mcu_active`.

### Target FPGA and Constraints
- Device: **Cyclone IV E EP4CE6F17C8** (from `DDS_simple.qsf`).
- Quartus project includes pin assignments for clock, UART RX, DAC outputs, switch, LED, and display pins.

---

## 5. MCU Firmware Notes | MCU 固件说明

### Main Logic (`User/main.c`)
- Initializes system config, W25Q64, OLED, and input UI.
- Scans keypad and builds user frequency input.
- Sends frequency to FPGA over UART as ASCII line.

### Key Mapping Behavior | 按键行为
- `0~9`: append digits to temporary value.
- `A`: commit as Hz.
- `B`: commit as kHz.
- `C`: commit as MHz.
- `*`: increment current frequency by 100 Hz.
- `#`: decrement current frequency by 100 Hz (floor at 100 Hz).
- `D`: clear/reset entry state.

- `0~9`：追加数字输入。
- `A`：按 Hz 提交。
- `B`：按 kHz 提交。
- `C`：按 MHz 提交。
- `*`：当前频率 +100 Hz。
- `#`：当前频率 -100 Hz（最低 100 Hz）。
- `D`：清空/复位输入状态。

### MCU Platform
- Device: **TI MSPM0G3519**.
- Tooling: TI MSPM0 SDK + SysConfig + Keil uVision project (`DDS_5_16.uvprojx`).

---

## 6. Build & Program Guide | 构建与下载指南

### FPGA (Quartus)
1. Open `DDS_FPGA_5_21/DDS_simple.qpf` in Quartus Prime (18.0-compatible project).
2. Compile project (`Processing -> Start Compilation`).
3. Program target FPGA with generated `.sof`.

### MCU (Keil uVision)
1. Open `DDS_MCU_5_21/Project/DDS_5_16.uvprojx`.
2. Ensure MSPM0 SDK path and SysConfig pre-build step are valid on your machine.
3. Build target and flash MCU via debugger/J-Link.

### Environment Note | 环境说明
- This repository does not provide a cross-platform CI pipeline for FPGA/Keil builds.
- FPGA/MCU build requires local vendor toolchains.

- 本仓库未提供可直接在通用 CI 环境运行的 FPGA/Keil 构建流水线。  
- FPGA/MCU 构建需依赖本地厂商工具链。

---

## 7. Runtime Usage | 运行使用说明

1. Power up MCU + FPGA board.
2. Select mode by hardware switch:
   - Fixed mode (`switch=1`): output 1 kHz.
   - UART mode (`switch=0`): controlled by MCU keypad input.
3. On MCU keypad, input value and unit (`A/B/C`) to transmit frequency.
4. Observe:
   - OLED (MCU UI)
   - 7-segment display (FPGA side)
   - DAC analog output waveform

1. 上电 MCU 与 FPGA 板卡。  
2. 用硬件拨码开关选择模式：
   - 固定模式（`switch=1`）：输出 1 kHz。
   - UART 模式（`switch=0`）：由 MCU 键盘输入控制。
3. 在 MCU 键盘输入数值，并通过 `A/B/C` 选择单位后发送。
4. 观察：
   - OLED（MCU 显示界面）
   - 数码管（FPGA 显示）
   - DAC 输出模拟波形

---

## 8. Known Limitations | 已知限制

- DDS control-word scaling currently uses an approximation (`*86`), not full-precision formula.
- UART protocol is plain ASCII newline-terminated text (no checksum/packet framing).
- Effective frequency range should be validated against your DAC, clocking, and analog front-end constraints.

- DDS 控制字当前采用近似换算（`*86`），并非高精度完整公式。  
- UART 协议为简单 ASCII + 换行结束，不带校验与帧封装。  
- 实际可用频率范围需结合 DAC、时钟和模拟前端能力验证。

---

## 9. Suggested Future Improvements | 后续优化建议

- Replace approximate frequency mapping with higher-precision scaling.
- Add explicit protocol framing and timeout handling for UART.
- Add reproducible build notes/scripts for both FPGA and MCU flows.
- Add calibration and waveform-quality measurement documentation.

- 将频率映射改为更高精度实现。  
- 为 UART 增加帧协议与超时保护。  
- 补充 FPGA 与 MCU 的可复现构建脚本/文档。  
- 增加输出波形质量与标定流程文档。

