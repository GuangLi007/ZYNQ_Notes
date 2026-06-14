# Hello World 与 MIO Key 实验笔记

## 环境
- Vivado / Vitis 2025.2
- 板卡：微相 Z7-Lite (Zynq-7020, xc7z020clg400-2)
- UART：CH340E (/dev/ttyUSB0, 115200 baud)
- GPIO：MIO0 (LED1), MIO9 (KEY1)

---

## 一、Hello World

### 1.1 标准 Vitis 模板版本

**源码**: `Hello_Vitis/hello_world/src/helloworld.c`

```c
#include <stdio.h>
#include "platform.h"
#include "xil_printf.h"

int main()
{
    init_platform();
    print("Hello World\n\r");
    print("Successfully ran Hello World application");
    cleanup_platform();
    return 0;
}
```

- 由 Vitis 的 `hello_world` 模板自动生成
- UART 由 bootrom/BSP 初始化 (115200 baud)
- 调用 `print()` 已是重定向到 UART 的版本 (`xil_printf`)

### 1.2 手动寄存器版本
git
**源码**: `Hello_World/vitis_workspace/hello_world/src/helloworld.c`

```c
#define SLCR_UNLOCK  0xF8000008
#define SLCR_LOCK    0xF8000004
#define SLCR_UNLOCK_VAL 0xDF0D
#define SLCR_LOCK_VAL   0x767B

#define MIO_PIN_14  0xF8000B38
#define MIO_PIN_15  0xF8000B3C
#define MIO_PIN_UART_VAL 0x00000603  // L0_SEL=3 (UART0)

#define UART0_CR         0xE0000000
#define UART0_MR         0xE0000004
#define UART0_BAUD_GEN   0xE0000018
#define UART0_TX_RX_FIFO 0xE0000020
#define UART0_BAUD_DIV   0xE0000034
#define UART0_SR         0xE000002C

void uart_init(void)
{
    Xil_Out32(SLCR_UNLOCK, SLCR_UNLOCK_VAL);

    Xil_Out32(MIO_PIN_14, MIO_PIN_UART_VAL);
    Xil_Out32(MIO_PIN_15, MIO_PIN_UART_VAL);

    Xil_Out32(UART0_CR, 0x00000028);       // Reset TX/RX
    Xil_Out32(UART0_MR, 0x00000020);       // 8N1
    Xil_Out32(UART0_BAUD_DIV, 0x00000006); // Baud divisor = 6
    Xil_Out32(UART0_BAUD_GEN, 0x0000007C); // Baud gen = 124 -> ~115200
    Xil_Out32(UART0_CR, 0x00000114);       // Enable TX+RX

    Xil_Out32(SLCR_LOCK, SLCR_LOCK_VAL);
}
```

#### 为什么要手动初始化 UART？

MIO0-15 属于 Bank0，其 I/O 配置由 **SLCR (System Level Control Registers)** 控制，BootROM **不会自动配置** MIO0-15 的外设复用功能。因此必须：
1. 解锁 SLCR（写入 `0xDF0D` 到 `0xF8000008`）
2. 配置 MIO14/15 的 L0_SEL 为 3（UART0 功能）
3. 初始化 UART0 控制器寄存器
4. 锁回 SLCR

#### 手动 putc/puts

```c
void uart_putc(char c)
{
    while (!(Xil_In32(UART0_SR) & 0x00000008)); // 等待 TX FIFO 空
    Xil_Out32(UART0_TX_RX_FIFO, (unsigned int)c);
}

void uart_puts(const char *s)
{
    while (*s) {
        if (*s == '\n') uart_putc('\r');
        uart_putc(*s++);
    }
}
```

#### 主循环

```c
int main()
{
    init_platform();
    uart_init();
    uart_puts("\r\nHello World\r\n");
    while (1) {
        uart_puts("Hello World Loop\r\n");
        volatile int i;
        for (i = 0; i < 50000000; i++);  // 软件延时
    }
    return 0;
}
```

- 无限循环打印，适合快速验证 UART 是否工作正常
- `volatile` 防止编译器优化掉延时循环

---

## 二、MIO Key（GPIO 按键控制 LED）

**源码**: `Hello_Vitis/Mio_Key/src/helloworld.c`

```c
#include "xgpiops.h"     // PS GPIO 驱动
#include "xgpiops_hw.h"  // GPIO 寄存器定义

#define MIO_LED1 0   // LED1 连接到 MIO0
#define MIO_KEY1 9   // KEY1 连接到 MIO9

#define input  0
#define output 1

XGpioPs Gpios;

int main()
{
    init_platform();

    XGpioPs_Config *ConfigPtr;
    ConfigPtr = XGpioPs_LookupConfig(0);
    XGpioPs_CfgInitialize(&Gpios, ConfigPtr, ConfigPtr->BaseAddr);

    // 设置方向
    XGpioPs_SetDirectionPin(&Gpios, MIO_LED1, output);
    XGpioPs_SetDirectionPin(&Gpios, MIO_KEY1, input);
    XGpioPs_SetOutputEnablePin(&Gpios, MIO_LED1, 1);

    while (1)
    {
        if (XGpioPs_ReadPin(&Gpios, MIO_KEY1))
            XGpioPs_WritePin(&Gpios, MIO_LED1, 1);  // 松开 -> 灭
        else
            XGpioPs_WritePin(&Gpios, MIO_LED1, 0);  // 按下 -> 亮
    }
    return XST_SUCCESS;
}
```

### MIO GPIO 特点

| MIO 信号 | GPIO 引脚 | 方向 | 板载对应 |
|----------|-----------|------|----------|
| MIO0     | GPIO 0    | 输出 | LED1     |
| MIO9     | GPIO 9    | 输入 | KEY1     |

- **Zynq PS GPIO (MIO)** 由 `XGpioPs` 驱动操作，非 PL 端的 `XGpio`
- MIO GPIO 无需额外配置 SLCR，BootROM 已将其设置为 GPIO 功能
- `XGpioPs_LookupConfig(0)` 查找设备 ID 0 的配置
- `XGpioPs_CfgInitialize` 初始化 GPIO 实例

### 关键 API

| API | 功能 |
|-----|------|
| `XGpioPs_LookupConfig(DeviceId)` | 查找 GPIO 配置 |
| `XGpioPs_CfgInitialize(InstPtr, ConfigPtr, EffectiveAddr)` | 初始化 GPIO 实例 |
| `XGpioPs_SetDirectionPin(InstPtr, Pin, Direction)` | 设置单引脚方向 |
| `XGpioPs_SetOutputEnablePin(InstPtr, Pin, OpEnable)` | 设置输出使能 |
| `XGpioPs_ReadPin(InstPtr, Pin)` | 读取引脚电平 |
| `XGpioPs_WritePin(InstPtr, Pin, Value)` | 写入引脚电平 |

---

## 三、MIO Bank 说明

Zynq-7000 的 MIO 分为 3 个 Bank：

| Bank | 引脚范围 | 电压域     | 需手动配置 |
|------|----------|------------|------------|
| Bank0 | MIO0-15  | VCCPIO0    | **是**（SLCR） |
| Bank1 | MIO16-31 | VCCPIO1    | 否（BootROM 配置） |
| Bank2 | MIO32-53 | VCCPIO2    | 否（BootROM 配置） |

- Bank0 的引脚复用由 SLCR 寄存器控制，**必须软件初始化**
- 通过 `XGpioPs` 驱动访问时，Bank0/1/2 统一为 0-53 的连续引脚号

---

## 四、项目文件结构

```
Hello_Vitis/
├── hello_world/           ← Vitis 应用 (标准模板)
│   └── src/helloworld.c   - print("Hello World")
├── Mio_Key/               ← Vitis 应用 (MIO GPIO)
│   └── src/helloworld.c   - KEY1 控制 LED1
└── platform/              ← Vitis 硬件平台

Hello_World/               ← Vivado 工程
└── vitis_workspace/hello_world/src/helloworld.c  - 手动 UART 初始化版

MIO_Key/                   ← Vivado 工程 (MIO Key)
```

---

## 五、调试命令 (xsct)

```tcl
setws /path/to/vitis_workspace
connect
targets -set -filter {name =~ "ARM*#0"}
rst -system
source /path/to/ps7_init.tcl
ps7_init
ps7_post_config
dow /path/to/hello_world.elf
con
```

观察 UART 输出：
```bash
stty -F /dev/ttyUSB0 115200 raw -echo
cat /dev/ttyUSB0
```
