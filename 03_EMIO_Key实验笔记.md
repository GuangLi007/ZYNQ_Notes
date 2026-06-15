# EMIO Key 实验笔记

## 环境
- Vivado / Vitis Unified 2025.2
- 板卡：微相 Z7-Lite (Zynq-7020, xc7z020clg400-2)
- UART：CH340E (/dev/ttyUSB0, 115200 baud)
- EMIO：PS GPIO 通过 EMIO 扩展到 PL 引脚

---

## 一、EMIO 与 MIO 的区别

| 特性 | MIO GPIO | EMIO GPIO |
|------|----------|-----------|
| 引脚 | PS 端固定 54 个 MIO 引脚 (0-53) | 通过 PL 引脚引出 |
| 数量 | 54 个 (固定) | 最大 64 个 (可配置) |
| 配置 | BootROM 默认配置为 GPIO | 需在 Vivado 中使能并配置宽度 |
| 引脚分配 | 无需 xdc | **需要 xdc 约束** |
| 驱动 | `XGpioPs` | `XGpioPs` (统一接口) |
| 引脚号 | 0-53 | 54-117 (EMIO 起始于 54) |

### 核心图解：为什么代码不用变

```text
【软件层】                    【硬件层】
                           ┌─────────────────────────────────────────┐
你的代码：                  │  ZYNQ PS端 (ARM Cortex-A9)              │
XGpioPs_WritePin(54, 1);   │                                          │
           ↓               │   GPIO 控制器 (相同的软件接口)            │
        引脚号 54           │      ↓                               ↓   │
                           │    MIO引脚 (0-53)        EMIO引脚 (54-117)│
                           │      ↓                     ↓            │
                           │   直接连接PS外设        连接到PL (FPGA)   │
                           │      ↓                     ↓            │
                           │   板上LED/MIO          通过PL再连接到引脚 │
                           └─────────────────────────────────────────┘
```

**根本原因**：MIO 和 EMIO 由 PS 端**同一个 GPIO 控制器**管理，软件驱动模型完全相同。Xilinx 通过这种抽象设计，让开发者用同一套 API 操作两种不同的物理路径。

- **相同的是**：驱动 API（`XGpioPs_*`）、初始化流程、方向/输出使能/读写操作
- **不同的是**：引脚编号范围（MIO=0-53，EMIO=54-117）和硬件连线方式（MIO 直连 PS，EMIO 需通过 PL 布线）

---

## 二、Vivado Block Design 配置

### 2.1 PS 配置要点

在 Vivado IP Integrator 中双击 Zynq PS IP：

1. 进入 **MIO Configuration** → **I/O Peripherals** → **GPIO**
2. 勾选 **EMIO GPIO (Width)**，设置宽度（本例为 `2`，对应 KEY1 + LED1）
3. 确认 **MIO** 下的 GPIO 也勾选（否则 GPIO 控制器会被禁用）

### 2.2 Block Design 连线

- PS 的 `GPIO_0` 端口（EMIO）右键 → **Make External** 或 **Create Port**
- 端口名可自定义（如 `emio_key_led_tri_io`）

### 2.3 xdc 引脚约束

参考 `Z7_LITE.xdc`，板载按键和 LED 的 PL 引脚：

```tcl
# PL 时钟 (50MHz)
set_property PACKAGE_PIN N18     [get_ports PL_CLK_50M]
set_property IOSTANDARD LVCMOS33 [get_ports PL_CLK_50M]

# PL 按键
set_property PACKAGE_PIN P16 [get_ports PL_KEY1]
set_property PACKAGE_PIN T12 [get_ports PL_KEY2]
set_property IOSTANDARD LVCMOS33 [get_ports PL_KEY1]
set_property IOSTANDARD LVCMOS33 [get_ports PL_KEY2]

# PL LED
set_property PACKAGE_PIN P15 [get_ports PL_LED1]
set_property PACKAGE_PIN U12 [get_ports PL_LED2]
set_property IOSTANDARD LVCMOS33 [get_ports PL_LED1]
set_property IOSTANDARD LVCMOS33 [get_ports PL_LED2]
```

端口名需要与 Block Design 中 Make External 时导出的端口名一致，或者直接在顶层 HDL 中命名。

---

## 三、软件代码

### 3.1 完整源码

```c
#include <stdio.h>
#include "xil_printf.h"
#include "xgpiops.h"
#include "xgpiops_hw.h"

#define EMIOLED1 54   // EMIO 起始 pin 号
#define EMIOKEY1 55

#define input  0
#define output 1

XGpioPs Gpios;

int main()
{
    int Status;
    XGpioPs_Config *ConfigPtr;

    init_platform();
    printf("EMIO Test! \n\r");

    ConfigPtr = XGpioPs_LookupConfig(XPAR_XGPIOPS_0_DEVICE_ID);
    Status = XGpioPs_CfgInitialize(&Gpios, ConfigPtr, ConfigPtr->BaseAddr);
    if (Status != XST_SUCCESS) {
        return XST_FAILURE;
    }

    XGpioPs_SetDirectionPin(&Gpios, EMIOLED1, output);
    XGpioPs_SetDirectionPin(&Gpios, EMIOKEY1, input);
    XGpioPs_SetOutputEnablePin(&Gpios, EMIOLED1, 1);

    while (1) {
        if (XGpioPs_ReadPin(&Gpios, EMIOKEY1))
            XGpioPs_WritePin(&Gpios, EMIOLED1, 1);
        else
            XGpioPs_WritePin(&Gpios, EMIOLED1, 0);
    }
    return XST_SUCCESS;
}
```

### 3.2 EMIO Pin 号计算规则

PS GPIO 控制器的引脚编号分布：

| 范围 | 类型 | 说明 |
|------|------|------|
| 0-53 | MIO | PS 端固定引脚 (Bank0/1/2) |
| 54-117 | EMIO | 通过 PL 扩展，最多 64 个 |

EMIO 引脚按照 Block Design 中配置的 **EMIO GPIO Width** 顺序映射：
- EMIO[0] → GPIO pin 54
- EMIO[1] → GPIO pin 55
- EMIO[N] → GPIO pin 54 + N

---

## 四、踩坑记录

### 4.1 BSP 未编译导致 xparameters.h 缺少定义

**现象**：编译时报错找不到 `XPAR_XGPIOPS_0_DEVICE_ID`（或其他 GPIOPS 相关符号）

**原因**：Platform 工程的 BSP 虽然配置了 `gpiops` 驱动，但 **没有实际编译生成**，导致 `xparameters.h` 中缺少 GPIOPS 寄存器基址、设备 ID 等定义。

**解决**（Vitis Unified IDE）：

1. **先编译 Platform 工程**（右键 Platform → **Build**），这会：
   - 生成完整的 BSP 库文件
   - 生成正确的 `xparameters.h`（含 GPIOPS 定义）
2. **再编译 Application 工程**

如果问题依旧，右键 Platform → **Regenerate BSP Sources** 强制重新生成 BSP。

### 4.2 确认 BSP 配置是否正确

检查以下文件（无需 IDE，可直接查看）：

| 文件 | 作用 | 正确配置 |
|------|------|----------|
| `bsp/include/DRVLISTConfig.cmake` | 驱动列表 | 应包含 `gpiops` |
| `bsp/include/ip_drv_map.yaml` | IP-驱动映射 | `ps7_gpio_0` → `gpiops` |

### 4.3 EMIO 未在 Vivado 中使能

如果 BSP 编译后 `xparameters.h` 仍无 GPIOPS，回 Vivado 检查：
- Zynq PS 配置中 **MIO Configuration → GPIO → EMIO GPIO** 是否勾选
- Block Design 中 GPIO EMIO 端口是否已 Make External
- 重新生成 XSA 并更新 Platform

---

## 五、项目文件结构

```
Hello_Vitis/
├── EMIO_Key/              ← Vitis 硬件平台 (由 XSA 创建)
│   ├── hw/                - XSA 硬件描述
│   ├── ps7_cortexa9_0/    - 域 (standalone) + BSP
│   └── export/            - 导出的平台文件 (含 xparameters.h)
├── EMio_Key/              ← Vitis 应用 (EMIO GPIO)
│   └── src/helloworld.c   - EMIO KEY1 控制 LED1
├── Mio_Key/               ← Vitis 应用 (MIO GPIO 对比)
│   └── src/helloworld.c
└── platform/              ← 其他平台 (用于 Hello World)
```

---

## 六、对比：MIO vs EMIO Key

```
MIO Key:   PS GPIO pin 9 (MIO9)  → 板载按键 (直连 PS)
EMIO Key:  PS GPIO pin 55 (EMIO1) → PL 引脚 P16 → 板载按键 (通过 PL 布线)
```

两者使用完全相同的 `XGpioPs` API，驱动层无差别。差异仅在于：
- **引脚号不同**（MIO=0-53, EMIO=54+）
- **MIO 无需 xdc，EMIO 需要 xdc**
- **EMIO 需先在 Vivado 中配置并使能**

---

## 七、调试命令 (xsct)

```tcl
setws /path/to/Hello_Vitis
connect
targets -set -filter {name =~ "ARM*#0"}
rst -system
source /path/to/EMIO_Key/hw/sdt/ps7_init.tcl
ps7_init
ps7_post_config
dow /path/to/EMio_Key/build/EMio_Key.elf
con
```

UART 输出观察：
```bash
stty -F /dev/ttyUSB0 115200 raw -echo
cat /dev/ttyUSB0
```
