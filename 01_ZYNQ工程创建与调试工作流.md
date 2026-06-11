# ZYNQ 工程创建与调试工作流

## 环境
- Vivado / Vitis 2025.2
- 板卡：微相 Z7-Lite (Zynq-7020, xc7z020clg400-2)
- JTAG：板载 FT232HL (VID 0403:PID 6014)
- UART：板载 CH340E (/dev/ttyUSB0, 115200 baud)

---

## 一、Vivado 硬件设计

### 1.1 创建工程
```tcl
create_project Hello_World ./Hello_World -part xc7z020clg400-2
```

### 1.2 创建 Block Design
1. **Create Block Design** → 添加 **ZYNQ7 Processing System**
2. 双击 ZYNQ IP 核心配置：
   - **PS-PL Configuration** → 关闭不需要的外设
   - **MIO Configuration** → 使能 **UART0** (MIO 14..15) → 115200 baud
   - 使能 **QSPI**、**SD**、**ENET** 等按需配置
3. **Run Block Automation**
4. **Create HDL Wrapper**

### 1.3 生成 bitstream
```tcl
launch_runs impl_1 -to_step write_bitstream
```

### 1.4 导出 XSA
```tcl
write_hw_platform -fixed -file ./Hello_World_wrapper.xsa
```

---

## 二、Vitis 裸机开发

### 2.1 创建 Platform (Python API)
```bash
source /home/kl/Xilinx_2025/2025.2/Vitis/settings64.sh
export VITIS_EMBEDDED_INSTALL=true
mkdir -p vitis_workspace && vitis -s - << 'EOF'
import vitis
client = vitis.create_client(workspace='/path/to/vitis_workspace')
platform = client.create_platform_component(
    name='hello_platform',
    hw_design='/path/to/Hello_World_wrapper.xsa',
    cpu='ps7_cortexa9_0',
    os='standalone'
)
app = client.create_app_component(
    name='hello_world',
    platform='/path/to/vitis_workspace/hello_platform/export/hello_platform/hello_platform.xpfm',
    template='hello_world',
    domain='standalone_ps7_cortexa9_0'
)
app.build()
EOF
```

### 2.2 关键注意事项
- **MIO0-15 不会自动初始化** — 需要在 C 代码中手动配置 MIO14/15：
```c
#define SLCR_UNLOCK  0xF8000008
#define SLCR_LOCK    0xF8000004
#define MIO_PIN_14   0xF8000B38
#define MIO_PIN_15   0xF8000B3C

void uart_init(void) {
    Xil_Out32(SLCR_UNLOCK, 0xDF0D);
    Xil_Out32(MIO_PIN_14, 0x00000603);  // L0_SEL=3 (UART0)
    Xil_Out32(MIO_PIN_15, 0x00000603);
    Xil_Out32(0xE0000034, 0x06);         // Baud divisor
    Xil_Out32(0xE0000018, 0x7C);         // Baud generator -> ~115200
    Xil_Out32(0xE0000004, 0x20);         // Mode: 8N1
    Xil_Out32(0xE0000000, 0x114);        // Control: TX+RX enable
    Xil_Out32(SLCR_LOCK, 0x767B);
}
```

---

## 三、调试 (xsct)

### 3.1 下载运行程序
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

### 3.2 检查硬件连接
```tcl
# Vivado TCL
open_hw_manager
connect_hw_server
current_hw_target [lindex [get_hw_targets] 0]
open_hw_target
get_hw_devices
# 应看到: arm_dap_0 xc7z020_1
```

---

## 四、常见问题

### 4.1 Vitis 只有 HLS 选项
根因：缺少 `.vitis_embedded` 标记文件。
解决：创建该文件后重启 Vitis：
```bash
touch /home/kl/Xilinx_2025/2025.2/Vitis/.vitis_embedded
source /home/kl/Xilinx_2025/2025.2/Vitis/settings64.sh
vitis -w /path/to/workspace
```

### 4.2 "Could not find ARM device on the board"
- 确认跳线 J1 短接 1-2 (JTAG 模式)
- 确认 FT232HL USB 已连接 (lsusb 应看到 0403:6014)
- Linux 下安装 cable 驱动：
```bash
sudo /home/kl/Xilinx_2025/2025.2/Vivado/data/xicom/cable_drivers/lin64/install_script/install_drivers/install_drivers
```

### 4.3 UART 无输出
- 检查 CH340 USB (第二个 Type-C 口) 是否连接
- 确认波特率 115200
- 确认 MIO14/15 已在 C 代码中初始化 (见 2.2)
- 测试：`stty -F /dev/ttyUSB0 115200 raw -echo && timeout 3 cat /dev/ttyUSB0`

### 4.4 Classic IDE workspace 无法在 Unified IDE 使用
旧版 Eclipse workspace 不兼容新版。创建新目录作为 workspace 即可。

### 4.5 安装程序找不到 Java
```bash
ln -sf /home/kl/Xilinx_2025/2025.2/Vivado/tps/lnx64/jre21.0.5_11 \
       /home/kl/Xilinx_2025/.xinstall/2025.2/tps/lnx64/jre21.0.5_11
```

---

## 五、跳线设置 (J1)

| 模式 | 跳线帽 |
|------|--------|
| JTAG | 短接 1-2 |
| QSPI | 短接 2-3 |
| SD   | 短接 3-4 |

调试时需 JTAG 模式，否则 ARM DAP 被禁用。

---

## 六、板卡信息
- 型号：微相 Z7-Lite (Zynq-7020)
- 芯片：xc7z020clg400-2
- JTAG：FT232HL (0403:6014)
- UART：CH340E (1a86:7523)
- DDR3：2Gb
- QSPI Flash：W25Q128JV
