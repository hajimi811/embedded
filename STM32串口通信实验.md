# STM32 串口通信实验报告

> 实验内容：串口协议基础与文件传输实验、STM32 USART1 查询/中断/DMA 三种方式收发、Keil 软件仿真逻辑分析仪观测波形
> 硬件平台：STM32F103C8T6 最小系统板（Blue Pill）＋ USB-TTL(CH340) 模块
> 开发环境：Keil MDK5（HAL 库）＋ STM32CubeMX ＋ 串口助手 sscom

**关于数据的说明**：本报告中标注为**理论值**的波特率、帧间隔、传输时间均由公式与时钟配置推算，可复核；标注为**实测**的数据全部来自**在线调试（SWD）读取芯片寄存器与引脚电平**的真实记录（见 6.6、7.1、7.3），未采集的表格项一律留空，**不编造**。文中接线图、流程图、对比图均为按原理绘制的示意图；Keil 软件界面与逻辑分析仪的波形截图需在实验过程中自行截取。

---

## 一、实验目的

1. 理解异步串行通信协议，掌握 RS-232、RS-485 标准及其与 TTL 电平的区别，理解 USB/TTL 转换模块（CH340）的工作原理。
2. 掌握用串口在两台计算机之间传输文件的方法，理解文件大小、波特率与传输时间三者的定量关系。
3. 掌握 STM32 的 USART 外设配置：波特率 115200、8 位数据位、1 位停止位、无校验。
4. 分别用**查询方式**、**串口接收中断方式**、**DMA 方式**实现串口收发，理解三者的区别与适用场景。
5. 学会在无示波器的条件下，使用 Keil 的软件仿真逻辑分析仪观察串口输出波形并测算波特率。

---

## 二、实验器材与软件环境

### 2.1 硬件器材

| 序号 | 名称 | 型号 / 规格 | 数量 | 用途 |
| --- | --- | --- | --- | --- |
| 1 | STM32 最小系统板 | STM32F103C8T6（Blue Pill，72MHz，64KB Flash / 20KB RAM） | 1 | 主控芯片 |
| 2 | USB-TTL 模块 | CH340G，3.3V/5V 可选 | 1（基础 2 需 2 个） | USB 与 TTL 串口互转，观察串口输出 |
| 3 | **DAP-Link 调试器** | CMSIS-DAP 协议（本实验用 MuseLab 版，USB VID 0x0D28） | 1 | SWD 方式下载程序与在线调试 |
| 4 | 杜邦线 | 母-母，约 20cm | 4～6 根 | 连接 TXD/RXD/GND 与 SWDIO/SWCLK |
| 5 | micro-USB 数据线 | 支持数据传输（非充电线） | 1 | 给开发板独立供电 |
| 6 | 上位机 | 笔记本电脑，Windows 10 | 1（基础 2 需 2 台） | 运行 Keil 与串口助手 |

### 2.2 软件环境

| 序号 | 软件 | 说明 |
| --- | --- | --- |
| 1 | Keil MDK-ARM（µVision5） | 工程编译、**SWD 下载**、在线调试、软件仿真 |
| 2 | STM32F1xx_DFP 器件支持包 | 工程中引用的是 Keil.STM32F1xx_DFP.2.4.1 |
| 3 | CMSIS-DAP 驱动 | **免驱**：Windows 10 自带，DAP-Link 直接识别为 CMSIS-DAP 设备 |
| 4 | STM32CubeMX | 图形化配置时钟、USART1、NVIC、DMA 并生成 HAL 初始化代码 |
| 5 | 串口助手 sscom | 上位机收发与文件传输 |
| 6 | CH340 驱动 | 让 USB-TTL 模块在系统中枚举为虚拟串口 COMx |
| 7 | STM32CubeProgrammer | 串口 ISP（UART）方式下载，**备用方案** |
| 8 | Arm GNU Toolchain（可选） | 14.2，配合 make 仅用于命令行校验"代码能否编译通过"；**用它编译出的镜像存在启动缺陷，不用于实际下载**（原因见 3.3.1） |

---

## 三、硬件部分：接线、下载与调试

### 3.1 引脚与信号说明

STM32F103C8T6 共有 3 个 USART，本实验使用 USART1，其引脚在芯片上是**固定**的：

| 信号 | 引脚 | 方向 | 说明 |
| --- | --- | --- | --- |
| USART1_TX | PA9 | 输出 | 单片机 → 上位机 |
| USART1_RX | PA10 | 输入 | 上位机 → 单片机 |
| GND | 任意 GND | — | 公共参考地，必须与模块相连 |
| 板载 LED | PC13 | 输出 | 低电平点亮，用作程序运行指示灯 |

USB-TTL(CH340) 模块一侧是 USB 口（插电脑），另一侧是引脚排针，常用引脚如下：

| 模块引脚 | 含义 | 本实验用法 |
| --- | --- | --- |
| TXD | 模块的发送脚（输出 TTL 电平） | 接 STM32 的 PA10(RX) |
| RXD | 模块的接收脚（输入 TTL 电平） | 接 STM32 的 PA9(TX) |
| GND | 地 | 接 STM32 的 GND |
| 3V3 | 模块 3.3V 输出 | **悬空不接** |
| 5V | 模块 5V 输出（来自 USB） | **悬空不接** |

### 3.2 任务一 / 任务二的接线步骤

![图 3  USB-TTL(CH340) ↔ STM32F103C8T6 接线图（任务一 / 任务二通用）](resources/usart_fig_ttl_wiring.png)

**接线对照表**

| USB-TTL 模块 | STM32F103C8T6 | 是否必须 | 说明 |
| --- | --- | --- | --- |
| TXD | PA10（USART1_RX） | 必须 | 交叉连接，模块发→单片机收 |
| RXD | PA9（USART1_TX） | 必须 | 交叉连接，单片机发→模块收 |
| GND | GND | 必须 | 共地，否则电平判断失去参考 |
| 3V3 | 不接 | 不接 | 模块输出电流不足（约 50mA） |
| 5V | 不接 | 不接 | 同上 |

**具体操作步骤**

1. 断开所有电源，电脑上的串口助手先不要打开。
2. 用一根母-母杜邦线连接 USB-TTL 模块的 **TXD** 与开发板的 **PA10**。
3. 用第二根杜邦线连接模块的 **RXD** 与开发板的 **PA9**。注意这两根线是**交叉**的，接反则完全收不到数据。
4. 用第三根杜邦线连接模块的 **GND** 与开发板的任一 **GND** 引脚。这一根线绝对不能省。
5. 模块的 **3V3 / 5V 保持悬空**，不要接到开发板的 3V3 上。
6. 把 USB-TTL 模块插到电脑 USB 口（此时模块由电脑 USB 供电）。
7. 用 micro-USB 数据线把开发板接到电脑的另一个 USB 口（开发板由此独立供电）。
8. 打开"设备管理器 → 端口(COM 和 LPT)"，确认出现 **USB-SERIAL CH340 (COMx)**，记下这个 COM 号；若出现带黄色感叹号的设备，先安装 CH340 驱动。
9. 打开串口助手 sscom，选择上一步的 COM 号，参数设置为 **115200 / 8 / 1 / 无校验 / 无流控**，点击"打开串口"。

**上电自检（做实验前先确认硬件正常）**

| 检查项 | 正常现象 | 异常时的方向 |
| --- | --- | --- |
| 开发板电源指示灯 | 常亮 | 换 USB 线与 USB 口，确认线是数据线而非充电线 |
| 设备管理器的 COM 口 | 出现 CH340 虚拟串口 | 安装 CH340 驱动 |
| 串口助手能否打开串口 | 提示"打开成功" | 关闭其他占用该 COM 的软件 |
| 下载程序后现象 | 串口助手持续收到 hello windows! | 回到"常见故障排查"逐项检查 |

### 3.3 程序下载方法

#### 3.3.1 方式一：DAP-Link（CMSIS-DAP）+ SWD（首选）

![图 9  DAP-Link（CMSIS-DAP）↔ STM32F103C8T6 的 SWD 下载 / 调试接线图](resources/usart_fig_daplink_wiring.png)

**为什么选 DAP-Link 而不是 ST-Link**

CMSIS-DAP 是 ARM 定义的**标准调试协议**，Keil、OpenOCD、pyOCD 都原生支持，不需要任何厂商私有驱动；SWD 只占 3 根线，而且**下载过程不依赖 BOOT0 跳线是否生效**——这一点比串口 ISP 可靠得多（串口 ISP 必须先让 BOOT0 拉高并复位，一旦跳线接触不良就完全无解）。此外它还能在线单步调试、打断点、看变量。

**引脚对照表**

| DAP-Link 引脚 | STM32F103C8T6 引脚 | 是否必须 | 说明 |
| --- | --- | --- | --- |
| SWDIO | PA13 | **必须** | SWD 数据线 |
| SWCLK | PA14 | **必须** | SWD 时钟线 |
| GND | GND | **必须** | 必须共地，否则无法通信 |
| RST | NRST（板上丝印为 R） | 可选 | 复位线，不接通常也能下载 |
| 3V3 | 3V3 | 可不接 | 开发板已由 micro-USB 供电时保持悬空最安全 |
| SWO | PB3 | 可选 | 单向跟踪输出，本实验用不到 |

> 注意：不同厂家的 DAP-Link 排针顺序与丝印不一致，**以上面丝印为准**，不要只按位置插线。

**接线步骤**

1. 断开所有电源后再接线。
2. 用第 1 根杜邦线连接 DAP-Link 的 **SWDIO** 与开发板的 **PA13**。
3. 用第 2 根杜邦线连接 DAP-Link 的 **SWCLK** 与开发板的 **PA14**。
4. 用第 3 根杜邦线连接 DAP-Link 的 **GND** 与开发板的任一 **GND**。
5. （可选）用第 4 根线连接 **RST** 与开发板的 **NRST**。
6. DAP-Link 的 **3V3 保持悬空不接**，开发板由 micro-USB 数据线独立供电（两路电源不要互相倒灌）。
7. 把 DAP-Link 用 USB 线插到电脑。CMSIS-DAP **免驱**，插上即可用。
8. 把开发板的 **BOOT0 跳线拨到 0**（正常运行模式）。SWD 下载本身不看 BOOT0，但下载完复位后要靠它决定从 Flash 启动。
9. **原有的 CH340 接线可以原样保留**：它接在 PA9/PA10 上，与 SWD 的 PA13/PA14 不冲突，继续用来在串口助手看波形输出。

**Keil 中的配置（本工程的四个工程已全部预置好）**

打开 `MDK-ARM\STM32_USART_TaskX_Y.uvprojx` 后，**Debug 页已经默认就是 CMSIS-DAP Debugger**，无需手工改。配置值如下，可用于核对：

| 配置文件 | 字段 | 值 |
| --- | --- | --- |
| `.uvprojx` | `TargetSelection` | `12` |
| `.uvprojx` | `Driver` | `BIN\CMSIS_AGDI.dll` |
| `.uvoptx` | `nTsel` | `12` |
| `.uvoptx` | `pMon` | `BIN\CMSIS_AGDI.dll` |

> 若换了一块板子或换了探针，可在 **Options for Target → Debug** 右侧点 **Settings**，在 Debug 页的 "Port" 下拉中选择 **SW**（SWD 模式），并在 "Debug Adapter" 处确认探针被识别。

**下载与验证步骤**

1. 按 **F7（Build）** 编译，确认 `0 Error(s), 0 Warning(s)`；本工程已设置自动生成 hex。
2. 点 **Options for Target → Debug → Settings**，确认能读到芯片：SW-DP ID 应显示 **0x1BA01477**（Cortex-M3 的 SW-DP 标识）。读不到就回到接线步骤检查 SWDIO/SWCLK/GND 三根线与供电。
3. 在同一 Settings 窗口切到 **Flash Download** 页，确认 Programming Algorithm 是 **STM32F10x_128**（本工程已正确配置），并勾选 **Reset and Run**。
4. 按 **F8（Download）**，底部 Build Output 出现 `Programming Done. Verify OK.` 即成功；勾了 Reset and Run 时程序会自动开始运行。
5. 打开串口助手（CH340 对应的 COM 口，115200 / 8 / 1 / 无校验），即可看到任务一(1) 的 `hello windows!` 连续输出。

**命令行方式（可选）**

Keil 的图形界面不是唯一选择。pyOCD 同样原生支持 CMSIS-DAP，可用一行命令烧录，便于脚本化和反复验证：

```text
pyocd flash --pack "<Keil.STM32F1xx_DFP 的 .pack 文件>" -t stm32f103c8 "hex\USART_Task1_1.hex"
```

需要注意：**STM32CubeProgrammer 不支持 CMSIS-DAP**（它只支持 ST-LINK、J-LINK 及 UART/USB DFU 等接口），所以不能再用它来烧。原先基于 ST-Link 的方案全部作废，改用上面的 Keil 或 pyOCD 路径。

**关于编译工具链：必须用 Keil MDK 编译（重要）**

本报告 `hex\` 目录中的四个镜像**全部由 Keil MDK（Arm Compiler）编译生成**：命令行执行

```text
UV4.exe -b "<工程路径>\STM32_USART_TaskX_Y.uvprojx" -o <编译日志>
```

退出码为 0 即表示编译成功，四个工程均为 `0 Error(s), 0 Warning(s)`，产物为 `MDK-ARM\<工程名>\<工程名>.hex`。

之所以特别强调这一点，是因为在验证过程中发现：**用 Arm GNU Toolchain（arm-none-eabi-gcc）配合本工程自带的 GCC 链接脚本编译出的镜像存在缺陷**——程序会在启动阶段的 C 库初始化 `__libc_init_array` 中跳出镜像末尾，落入还未写入程序的空白 Flash（读出值为 `0xFFFFFFFF`）。该值并非合法的 Thumb 指令，取下非法指令异常，触发 **UNDEFINSTR 类型的 HardFault**。

该结论由 SWD 在线读取内核故障寄存器得到证实：**CFSR = 0x00010000**（bit16 UNDEFINSTR 置 1），出错时的 **PC = 0x08001750**。改用 Keil MDK 编译后，同样的源码上电即正常运行，因此本实验统一采用 Keil 工具链编译，把 GCC 命令行路径仅作为"代码可编译"的辅助验证手段（见 §5.1）。

#### 3.3.2 方式二：串口 ISP（备用方案）

![图 4  USB-TTL 串口 ISP 下载接线与 BOOT 跳线设置](resources/usart_fig_isp_download.png)

STM32F103 内置 Bootloader，可以通过 USART1 接收程序。接线与 3.2 节完全相同，只需再改一下 BOOT 跳线：

| 步骤 | 操作 | 说明 |
| --- | --- | --- |
| 1 | BOOT0 跳线帽拨到 **1**（高电平） | 选择"系统存储器启动"，进入 Bootloader |
| 2 | 按一次 **RESET** 键 | BOOT 状态在复位时才被采样，改完必须复位 |
| 3 | 打开 STM32CubeProgrammer，接口选 **UART** | 不要选 ST-LINK |
| 4 | 选择 USB-TTL 对应的 COM 号，波特率 115200，点 **Connect** | 连接成功会读出芯片型号与 Flash 容量 |
| 5 | 打开 `hex/USART_Task1_1.hex`，点 **Download** | 也可直接下载其他任务的 hex |
| 6 | BOOT0 跳线帽拨回 **0** | 恢复"主 Flash 启动" |
| 7 | 再按一次 **RESET** | 程序开始运行 |

> 注意：串口 ISP 下载依赖开发板已经由 micro-USB 正常供电。若发现"Timeout error while waiting for acknowledgement"（芯片无应答），先确认 BOOT0 是否真的在 1 位、是否按过复位，再确认供电。

### 3.4 基础 2：两台笔记本之间的串口传文件接线

![图 2  两台笔记本借助 USB-TTL 模块建立串口连接传输文件](resources/usart_fig_file_transfer.png)

| 笔记本 A 侧模块引脚 | 笔记本 B 侧模块引脚 | 说明 |
| --- | --- | --- |
| TXD | RXD | 交叉 |
| RXD | TXD | 交叉 |
| GND | GND | 必须共地 |

**步骤**

1. 两台笔记本各插一个 USB-TTL 模块，各自装上 CH340 驱动，在设备管理器记下各自的 COM 号。
2. 按上表用 3 根杜邦线把两个模块连起来（TXD↔RXD 交叉、GND↔GND）。
3. 两端串口助手设置**完全相同**的参数：115200、8、1、无校验、无流控。
4. A 端选择要发送的文件（图片、视频或压缩包），使用"发送文件"功能。
5. B 端先设置好"接收文件保存路径"，再开始接收，接收完成后比对文件大小与内容。
6. 记录文件大小、设定波特率与实际耗时，填入 4.2 节的实测记录表。

### 3.5 常见故障排查

| 现象 | 可能原因 | 处理方法 |
| --- | --- | --- |
| 串口助手收到的全是乱码 | 两端波特率不一致；晶振未起振导致时钟不对 | 两端统一为 115200；检查 HSE 8MHz 是否正确起振 |
| 一个字符都收不到 | TXD/RXD 未交叉；未共地；选错 COM 号 | 交叉接线、补接 GND、重新确认 COM 号 |
| 设备管理器没有 COM 口 | CH340 驱动未安装；插在了充电线 | 安装驱动；更换数据线 |
| 偶尔丢几个字符 | 上位机缓冲区溢出、界面刷新太慢 | 降低发送频率，或把数据存盘而不是在屏幕上刷 |
| Keil 提示找不到仿真器 | Debug 页未选中 CMSIS-DAP Debugger | 在 Options for Target → Debug 中选 **CMSIS-DAP Debugger**（四个工程已预置） |
| Keil 连不上芯片（读不到 SW-DP ID） | SWDIO / SWCLK / GND 接线松动或接错；开发板未供电 | 按 3.3.1 核对三根线；确认 micro-USB 已供电、电源灯亮 |
| DAP-Link 的 U 盘烧录失败、盘符中途消失 | 部分克隆版 DAPLink 固件的 MSC 接口不稳定 | 放弃 MSC 方式，改用 Keil 的 CMSIS-DAP + SWD，或 pyOCD |
| Keil 能下载但程序不运行 | BOOT0 仍为 1，复位后进了引导程序 | 把 BOOT0 跳线拨到 0 后重新上电 |
| 串口 ISP 下载超时无应答 | BOOT0 不在 1 位；改后未按复位；模块 3.3V 供电电流不足 | BOOT0=1 后按复位；模块 3V3 悬空，改用 micro-USB 给板子供电 |

---

## 四、基础题解答

### 4.1 基础 1：串口协议、电平标准与 CH340 模块

![图 1  USART 8-N-1 数据帧结构 与 TTL / RS-232 / RS-485 电平标准对比](resources/usart_fig_frame_levels.png)

#### 4.1.1 串口协议要点

UART（通用异步收发器）是**异步**串行通信：收发双方没有共享的时钟线，靠事先约定好的**波特率**（每秒传输的码元数）各自计时，因此两端的波特率必须一致，误差通常要求控制在 2% 以内。

一帧数据的组成如下（以本实验的 8-N-1 为例）：

| 部分 | 位数 | 电平 | 作用 |
| --- | --- | --- | --- |
| 空闲 | — | 高 | 线路空闲时保持高电平 |
| 起始位 | 1 | 低 | 通知接收方"一帧开始了"，接收方据此对齐采样时钟 |
| 数据位 | 8 | 数据 | **低位在前**（LSB 先发） |
| 校验位 | 0 | — | 无校验（NONE） |
| 停止位 | 1 | 高 | 标志一帧结束，同时提供帧间的最小间隔 |

因此一个字节实际占 **1 + 8 + 0 + 1 = 10 位**，这是后面所有时间估算的基础。异步串口的缺点是每字节要多传 2 位（起始位+停止位），有效数据率只有波特率的 80%。

#### 4.1.2 RS-232、RS-485 标准与 TTL 电平的区别

需要区分两个层面：**逻辑电平规范**（TTL/CMOS）和**接口标准**（RS-232、RS-485）。后者不仅规定电平，还规定了接口形式、驱动能力与传输距离。

| 项目 | TTL / CMOS | RS-232 | RS-485 |
| --- | --- | --- | --- |
| 传输方式 | 单端（对地） | 单端（对地） | 差分（A、B 两线电压差） |
| 逻辑 1 | 2.4V ~ 3.3V/5V | −3V ~ −15V（负逻辑） | V(A)−V(B) > +200mV |
| 逻辑 0 | 0V ~ 0.4V | +3V ~ +15V（负逻辑） | V(A)−V(B) < −200mV |
| 典型电压 | 0 / 3.3V（或 5V） | ±3V ~ ±15V | 差分 ±1.5V ~ ±6V |
| 传输距离 | < 1 m | 约 15 m | 可达 1200 m |
| 拓扑 | 板内芯片间 | 点对点 | 多点总线（可挂多台设备） |
| 抗干扰能力 | 弱 | 一般 | 强（差分抑制共模干扰） |
| 与 MCU 连接 | 直接相连 | 需 MAX232 等电平转换芯片 | 需 MAX485 等收发器 |

**TVT 电平为什么不能直接接 RS-232**：RS-232 采用的是**负逻辑**，且电压摆幅高达 ±15V，直接接入单片机 I/O 会因超出绝对最大额定值而损坏引脚；同时逻辑也是反的。所以必须经过 MAX232 之类的芯片做电平与逻辑的双重转换。

**RS-485 为什么能传得远**：它用两根线传输一对**反相**信号，接收端只关心两者之差。沿线耦合进来的干扰对两根线几乎相同（共模），相减之后被抵消，所以抗干扰能力远强于单端方式。工程上还需要在总线两端各接一个 120Ω 终端电阻做阻抗匹配，减小反射。

#### 4.1.3 USB/TTL 转 232 模块（CH340）的工作原理

CH340 是一颗 **USB 总线转串口**的芯片，它把"复杂的 USB 协议"和"简单的 UART 时序"互相翻译，作用是让没有串口的现代笔记本也能使用串口设备。

工作过程可以分成三部分：

1. **USB 侧**：PC 通过 USB 的 D+ / D− 差分线与 CH340 通信。USB 协议本身很复杂（有令牌包、握手包、端点、枚举等），CH340 内部固件把这一整套协议实现好，对上层表现为一个 USB 设备。
2. **驱动侧**：PC 上安装 CH340 驱动后，操作系统把这个 USB 设备识别成一个**虚拟串口（VCP）**，分配一个 COM 号。于是串口助手只要像操作传统串口一样读写 COMx 即可，完全不用关心 USB 协议。
3. **串口侧**：芯片的另一端是标准的 UART 引脚 TXD / RXD，输出 **TTL 电平**的异步串行数据（起始位+数据位+停止位）。模块上通常还有 3V3 与 5V 引脚，以及可选的电平跳线/焊盘：短接相应位置即可让 TXD/RXD 工作在 3.3V 或 5V。本实验使用 **3.3V** 以匹配 STM32 的 I/O 电平。

一句话概括数据流：**串口助手 → COM 口 → CH340 驱动 → USB 总线 → CH340 芯片 → TTL 串行数据（TXD/RXD）→ 单片机 PA10/PA9**，反方向同理。

如果还需要真正的 RS-232 电平，则在 CH340 的 TTL 输出之后再接一级 MAX232 之类的电平转换芯片即可，这就是市面上"USB 转 RS232"模块的做法。

### 4.2 基础 2：串口传输文件实验

#### 4.2.1 文件大小、波特率与传输时间之间的关系

一帧（一个字节）占 10 位，因此有基本关系：

**传输时间 T ≈ 文件字节数 N × 10 (bit/Byte) ÷ 波特率 (bps)**

反过来说，三个量互相制约：**文件越大越慢、波特率越高越快、时间与文件大小成正比、与波特率成反比**。在 115200 bps 下，串口的有效吞吐只有 115200 ÷ 10 = **11520 Byte/s ≈ 11.25 KB/s**，比 U 盘慢好几个数量级，这正是串口传大文件非常耗时的原因。

按上式推算的理论时间如下表：

| 文件大小 | 波特率 | 理论传输时间 | 等效 |
| --- | --- | --- | --- |
| 1 MB (1048576 B) | 115200 | 91.0 s | 约 1 分 31 秒 |
| 1 MB | 460800 | 22.8 s | 约 23 秒 |
| 10 MB (10485760 B) | 115200 | 910.2 s | 约 15.2 分钟 |
| 10 MB | 460800 | 227.6 s | 约 3.8 分钟 |
| 10 MB | 921600 | 113.8 s | 约 1.9 分钟 |

**实测记录表（完成实验后填写，不可编造）**

| 序号 | 文件名/类型 | 文件大小 N (Byte) | 波特率 (bps) | 理论时间 (s) | 实测时间 (s) | 相对误差 |
| --- | --- | --- | --- | --- | --- | --- |
| 1 |  |  | 115200 |  |  |  |
| 2 |  |  | 460800 |  |  |  |
| 3 |  |  | 115200 |  |  |  |

**实测值一般会大于理论值**，原因有：

1. 串口只知道"字节流"，文件传输需要额外的协议来划界与检错。若使用 XMODEM/YMODEM，每 128 或 1024 字节还要等一次应答（ACK），等待期间的线路是空闲的，效率会明显下降；若使用不带协议的"原始发送"，开销则很小。
2. 上位机需要不断读写磁盘、刷新界面，会占用时间。
3. 两端波特率存在微小偏差（晶振精度），极端情况下可能导致误码而重传。

#### 4.2.2 只接 TX–RX、RX–TX 两根线能否正常工作

要分两种情况回答。

**（1）不接电源线 —— 可以正常工作**

这里的"电源线"指的是模块提供给对方的 3V3/5V。本实验的接线是"两台笔记本各插一个 USB-TTL 模块"，每个模块都由**自己所连的那台电脑的 USB 口供电**，电源是各自独立的，并不需要从对方取电。所以缺了这根线不影响通信。

但如果"不接电源线"指的是模块本身没插 USB、完全没有供电，那当然不能工作：芯片不上电，收发器根本无法驱动 TXD 引脚。

**（2）不接 GND 地线 —— 不能可靠工作**

原因有三点：

1. **串口是单端信号，逻辑电平必须相对"地"来判断**。TTL 电平规定：接收端电压高于约 2.0V 判为逻辑 1，低于约 0.8V 判为逻辑 0。这个"电压"是相对于**接收端自己的 GND**测量的。如果两端不共地，两个系统的 0V 参考点之间可能存在电位差（例如两台笔记本分别使用不同的电源适配器时，由于适配器 Y 电容漏电流，机壳之间常常存在几十伏的交流共模电压）。叠加之后，本应是 0V 的低电平可能被抬到 1.5V 这种模棱两可的区间，判决就会出错。
2. **信号电流缺少回流路径**。电流必须构成回路才能流动，数据线上的电流最终要经由地线返回发送端。不共地时回流路径不完整，信号完整性急剧恶化，表现为波形畸变、边沿变缓，误码率大幅升高。
3. **存在损坏芯片的风险**。当地电位差较大时，这个电压会直接加在两个芯片的 I/O 引脚上，可能超过其绝对最大额定值而击穿。如果用的是 ±12V 的 RS-232 电平，风险更高。

**实验结论**：不接电源线通常没问题（各端独立供电），但**GND 必须相连**。若在特定条件下"碰巧"能通信，那只是因为两地电位差很小，属于侥幸，绝不能作为可靠设计。实际调试中，绝大多数"串口收不到数据/全是乱码"的问题都出在没共地或 TX/RX 没交叉上。

---

## 五、软件部分：四个 Keil 工程

### 5.1 工程组织

四个子任务各建一个独立的 Keil MDK 工程，结构完全一致，公共代码集中在 `bsp_uart.c/.h` 与 `stm32f1xx_hal_msp.c` 中，便于对比三种收发方式的差异。

| 工程目录 | 对应任务 | 实现方式 | 关键文件 |
| --- | --- | --- | --- |
| STM32_USART_Task1_1 | 任务一(1) | 阻塞发送，无延时连续发 hello windows! | Core/Src/main.c |
| STM32_USART_Task1_2 | 任务一(2) | **查询（轮询）**方式接收，# 暂停 / * 恢复 | Core/Src/main.c |
| STM32_USART_Task2_1 | 任务二(1) | **接收中断**方式接收，# 暂停 / * 恢复 | main.c + stm32f1xx_it.c |
| STM32_USART_Task2_2 | 任务二(2) | **DMA** 方式连续发送 | main.c + stm32f1xx_hal_msp.c |

每个工程的目录结构：

```text
STM32_USART_TaskX_Y/
├─ Core/
│   ├─ Inc/    main.h  bsp_uart.h  stm32f1xx_it.h  stm32f1xx_hal_conf.h
│   └─ Src/    main.c  bsp_uart.c  stm32f1xx_it.c  stm32f1xx_hal_msp.c
├─ Drivers/    CMSIS 与 STM32F1xx_HAL_Driver（HAL 库源码）
├─ linker/     STM32F103C8Tx.ld（GCC 链接脚本，命令行编译用）
├─ MDK-ARM/    STM32_USART_TaskX_Y.uvprojx（Keil 工程文件）
└─ Makefile    命令行编译脚本（可选）
```

**代码分层说明**：`bsp_uart.c` 只负责"配置什么参数"（波特率、字长、停止位、校验位），而"用哪几个引脚、开哪个时钟、要不要开中断/DMA"属于与板子相关的部分，统一放在 `stm32f1xx_hal_msp.c` 的 `HAL_UART_MspInit()` 里，由 HAL 库在 `HAL_UART_Init()` 内部自动回调。这是 HAL 库固定的分层套路，四个工程只在 `MspInit` 与 `main.c` 上有差别。

**构建工具链（重要）**：本报告交付的 `hex\` 中的镜像全部用 **Keil MDK（Arm Compiler）** 构建，四个工程以命令行方式编译：

```text
UV4.exe -b "<工程路径>\STM32_USART_TaskX_Y.uvprojx" -o <编译日志>
```

四个工程均为 `0 Error(s), 0 Warning(s)`，hex 位于 `MDK-ARM\<工程名>\`。工程目录里虽然也附带了 `Makefile` 与 `linker/STM32F103C8Tx.ld`（供 Arm GNU Toolchain 使用），但**该 GCC 路径编译出的镜像存在启动缺陷，不能用于实际运行**（原因与证据见 3.3.1），故仅作为"代码可编译"的辅助校验，不产出交付镜像。

![图 10  Keil MDK 编译输出窗口：Program Size 与 0 Error(s), 0 Warning(s)（实测截图）](resources/img11_keil_build.png)

### 5.2 STM32CubeMX 关键配置

| 配置项 | 取值 | 说明 |
| --- | --- | --- |
| RCC → HSE | Crystal/Ceramic Resonator | 使用板载 8MHz 晶振 |
| Clock Configuration | HSE 8MHz × PLL9 = **72MHz** | APB2 = 72MHz，USART1 挂在其上 |
| USART1 Mode | Asynchronous | 异步模式 |
| Baud Rate | 115200 Bits/s | 任务书要求 |
| Word Length | 8 Bits | 8 位数据位 |
| Parity | None | 无校验 |
| Stop Bits | 1 | 1 位停止位 |
| NVIC（任务二(1)） | 勾选 USART1 global interrupt | 抢占优先级 1 |
| DMA（任务二(2)） | USART1_TX → DMA1 Channel 4，Normal 模式 | RX 对应 Channel 5 |
| GPIO | PA9 = USART1_TX，PA10 = USART1_RX | 自动生成 |

### 5.3 任务一(1)：无延时连续发送 hello windows!

![图 5  任务一(1) 程序流程图](resources/usart_fig_flow_task1_1.png)

核心代码只有一句发送，放在永不退出的主循环里，**帧与帧之间不加任何延时**：

```c
#define HELLO_MSG   "hello windows!"      /* 14 字节 */

while (1)
{
    /* 阻塞式发送：内部逐字节等待 TXE 标志，上一字节没搬进移位寄存器
       就不写下一字节，因此不会因为"发太快"而在单片机侧丢字节 */
    HAL_UART_Transmit(&huart1, (uint8_t *)HELLO_MSG,
                      sizeof(HELLO_MSG) - 1U, HAL_MAX_DELAY);

    LED_Toggle();      /* 每发完一帧翻转一次 PC13，约 410Hz，肉眼看着偏暗 */
}
```

注意 `sizeof(HELLO_MSG) - 1`：C 语言会自动在字符串末尾补一个 `'\0'`，发送长度要把它排除掉，否则上位机会多收到一个 0x00。

**时间推算**：一帧 14 字节 = 14 × 10 = 140 位；140 ÷ 115200 bps = **1.215 ms**，即每秒约 823 帧，数据率约 11.2 KB/s。

**关于"不加延时是否丢数据"**：

实验中确实可以观察到上位机串口助手出现**接收数据丢失**（显示不全、界面卡顿甚至假死），但**根源不在单片机**：

1. 单片机侧用的是阻塞发送，每个字节都等到 TXE 置位才写下一个，不会丢字节。
2. 链路上没有启用任何流控（既没接 RTS/CTS，也没开 XON/XOFF），PC 侧"来不及取走"时，数据会被**静默丢弃**，不会通知发送方。
3. 主要瓶颈在 PC：11.2 KB/s 的文本要在屏幕上持续刷新，界面重绘远远跟不上；缓冲区一旦写满，新来的数据就被丢掉。

**验证方法**：在发送之间插入 `HAL_Delay(10)`，或在串口助手中改成"接收保存到文件"而不在屏幕上刷新，丢包现象即消失。反过来也证明了丢包来自上位机侧。

### 5.4 任务一(2)：查询方式实现 # 暂停 / * 恢复

![图 6  查询（轮询）方式 与 串口接收中断方式 对比](resources/usart_fig_poll_vs_irq.png)

查询方式的核心是**主循环每转一圈主动去读一次状态寄存器 SR**：

```c
uint8_t send_enable = 1U;

while (1)
{
    /* ① 主动查询：读 SR 判断 RXNE 位，为 1 说明收到一个完整字节 */
    if (__HAL_UART_GET_FLAG(&huart1, UART_FLAG_RXNE) != RESET)
    {
        /* 读 SR 之后紧接着读 DR，这个固定顺序会同时清掉 RXNE 和 ORE 两个标志 */
        uint8_t ch = (uint8_t)(huart1.Instance->DR & 0x000000FFU);

        if (ch == '#') { send_enable = 0U; }        /* 暂停 */
        else if (ch == '*') { send_enable = 1U; }   /* 恢复 */
    }

    /* ② 溢出错误处理：连续发送时若接收处理不及时，ORE 会置位 */
    if (__HAL_UART_GET_FLAG(&huart1, UART_FLAG_ORE) != RESET)
    {
        __HAL_UART_CLEAR_OREFLAG(&huart1);
    }

    /* ③ 按标志决定是否发送 */
    if (send_enable != 0U)
    {
        HAL_UART_Transmit(&huart1, (uint8_t *)HELLO_MSG,
                          sizeof(HELLO_MSG) - 1U, HAL_MAX_DELAY);
        LED_Toggle();
    }
}
```

**容易被忽略的 ORE 标志**：本工程发送是阻塞的，一帧要占用约 1.2ms；如果这段时间里上位机连发多个字节，就会覆盖掉还没被读走的旧数据，硬件会把 **ORE（OverRun，溢出）** 位置 1。应用层必须清掉它，否则接收可能永久失效。清除方法是"读 SR 再读 DR"这个固定顺序，用宏 `__HAL_UART_CLEAR_OREFLAG()` 即可。

### 5.5 任务二(1)：中断方式实现同样功能

改成中断方式后，接收这件事交给硬件来"报信"，主循环只需要专心发送：

```c
static uint8_t rx_byte;
/* 中断里改写、主循环里读取，必须加 volatile，
   否则 -O2 优化后主循环可能一直用寄存器里缓存的旧值 */
static volatile uint8_t send_enable = 1U;

int main(void)
{
    /* ... 初始化 ... */

    /* 关键一步：把"接收 1 个字节"登记给中断，函数立刻返回 */
    HAL_UART_Receive_IT(&huart1, &rx_byte, 1U);

    while (1)
    {
        if (send_enable != 0U)
        {
            HAL_UART_Transmit(&huart1, (uint8_t *)HELLO_MSG,
                              sizeof(HELLO_MSG) - 1U, HAL_MAX_DELAY);
        }
    }
}

/* 数据收满 1 字节后，HAL 库在中断上下文里回调这个函数 */
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        if (rx_byte == '#') { send_enable = 0U; }
        else if (rx_byte == '*') { send_enable = 1U; }

        /* HAL 的中断接收是"一次性"的，收完就关了接收中断，
           必须重新武装，否则只能收第一个字符 */
        HAL_UART_Receive_IT(huart, &rx_byte, 1U);
    }
}

/* 溢出等错误也要重新武装，否则接收功能永久失效 */
void HAL_UART_ErrorCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        __HAL_UART_CLEAR_OREFLAG(huart);
        HAL_UART_Receive_IT(huart, &rx_byte, 1U);
    }
}
```

中断服务函数写在 `stm32f1xx_it.c` 中，只做"清标志 + 转交 HAL 库"这一件事：

```c
void USART1_IRQHandler(void)
{
    HAL_UART_IRQHandler(&huart1);   /* 内部完成读数据、清标志、回调用户函数 */
}
```

**调用链**：数据到达 → 硬件置 RXNE=1 并触发 USART1_IRQn → 向量表跳到 `USART1_IRQHandler()` → `HAL_UART_IRQHandler()` 读走数据 → 回调 `HAL_UART_RxCpltCallback()`。注意函数名必须与启动文件 `startup_stm32f103xb.s` 向量表里的名字**完全一致**，否则中断会跳到启动文件里的弱定义空函数，什么也不做。

**与查询方式的两点关键差别**：

| 对比项 | 查询方式（任务一(2)） | 中断方式（任务二(1)） |
| --- | --- | --- |
| 谁发现数据到达 | CPU 主动轮询 RXNE | 硬件自动触发中断 |
| CPU 开销 | 每圈都要查，没数据也白查 | 只在数据真正到达时才被打断 |
| 需要配置 NVIC | 不需要 | 必须使能 USART1_IRQn |
| 收完一字节后 | 无需额外操作 | 必须重新调用 HAL_UART_Receive_IT() |

### 5.6 任务二(2)：DMA 方式连续发送

![图 7  任务二(2) DMA 发送流程 与 三种发送方式的 CPU 占用对比](resources/usart_fig_dma_modes.png)

DMA（Direct Memory Access，直接存储器访问）是一个"搬运工"：CPU 只要把"源地址、目的地址、搬多少字节"告诉它，它就能自己把数据从内存搬到外设，**全程不占用 CPU 指令**。

发送侧只用一句调用，且它是**非阻塞**的：

```c
/* 必须放在全局区：DMA 搬运期间这块内存要一直有效，
   而且 DMA 访问的是物理地址，全局数组地址在链接时就固定了 */
#define DMA_MSG  "STM32F103C8T6 USART1 DMA Test @115200bps 8-N-1\r\n"   /* 49 字节 */

static volatile uint8_t tx_cplt = 0U;

int main(void)
{
    /* ... 初始化 ... */

    HAL_UART_Transmit_DMA(&huart1, (uint8_t *)DMA_MSG, sizeof(DMA_MSG) - 1U);

    while (1)
    {
        if (tx_cplt != 0U)          /* 整帧真正发完之后，再启动下一帧 */
        {
            tx_cplt = 0U;
            LED_Toggle();
            HAL_UART_Transmit_DMA(&huart1, (uint8_t *)DMA_MSG, sizeof(DMA_MSG) - 1U);
        }
    }
}

void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1) { tx_cplt = 1U; }
}
```

DMA 通道由 MCU 的 `HAL_UART_MspInit()` 配置。STM32F103 的外设与 DMA 通道是**硬件固定对应**的：USART1_TX → DMA1 通道 4，USART1_RX → DMA1 通道 5。

```c
__HAL_RCC_DMA1_CLK_ENABLE();

hdma_usart1_tx.Instance                 = DMA1_Channel4;
hdma_usart1_tx.Init.Direction           = DMA_MEMORY_TO_PERIPH; /* 内存 -> 外设 */
hdma_usart1_tx.Init.PeriphInc           = DMA_PINC_DISABLE;     /* 外设地址不自增 */
hdma_usart1_tx.Init.MemInc              = DMA_MINC_ENABLE;      /* 内存地址自增 */
hdma_usart1_tx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
hdma_usart1_tx.Init.MemDataAlignment    = DMA_MDATAALIGN_BYTE;
hdma_usart1_tx.Init.Mode                = DMA_NORMAL;           /* 传完就停 */
hdma_usart1_tx.Init.Priority            = DMA_PRIORITY_HIGH;
HAL_DMA_Init(&hdma_usart1_tx);

__HAL_LINKDMA(huart, hdmatx, hdma_usart1_tx);   /* 把 DMA 句柄挂到串口句柄上 */

HAL_NVIC_SetPriority(DMA1_Channel4_IRQn, 1, 0);
HAL_NVIC_EnableIRQ(DMA1_Channel4_IRQn);
HAL_NVIC_SetPriority(USART1_IRQn, 2, 0);        /* 别忘了这个！ */
HAL_NVIC_EnableIRQ(USART1_IRQn);
```

其中 `__HAL_LINKDMA` 这一步不能省：`HAL_UART_Transmit_DMA()` 内部就是通过 `huart->hdmatx` 找到"该用哪个 DMA 通道来发"的。

**为什么 DMA 发送要开两个中断**（这是本任务最容易出错的地方）：

| 中断 | 触发时机 | HAL 在此做什么 |
| --- | --- | --- |
| DMA1_Channel4_IRQHandler | DMA 把最后一个字节写进 USART1->DR | 关闭串口的 DMA 发送请求（DMAT 位），打开 TC 中断 |
| USART1_IRQHandler | 最后一个字节真正从移位寄存器输出完毕（TC 标志） | 回调 HAL_UART_TxCpltCallback()，标志整帧发完 |

只写 DMA 中断、忘了写 USART1 中断，`HAL_UART_TxCpltCallback()` 就永远不会被调用，程序会一直卡在等标志上。值得注意的是，当 DMA 传输结束时数据其实**还没有真正发出去**——DR 里最后那几个字节仍在移位寄存器中排队，必须等 TC 标志才算发完。

**提高波特率**：任务书要求"115200bps 或更高速度"。把 `bsp_uart.h` 中的 `BSP_UART_BAUDRATE` 改成 `460800U`（或 `921600U`）重新编译即可，同时上位机串口助手也要改成相同数值，否则会全是乱码。

本帧长 49 字节，一帧时间 = 49 × 10 ÷ 115200 = **4.25 ms**。

**三种发送方式的对比**：

| 方式 | 函数 | 特点 | 适用场景 |
| --- | --- | --- | --- |
| 阻塞发送 | HAL_UART_Transmit() | CPU 逐字节等待，简单可靠，但全程被占住 | 少量数据、对实时性无要求 |
| 中断发送 | HAL_UART_Transmit_IT() | 每字节一次中断，CPU 不必傻等，但字节多时中断次数爆炸 | 中等数据量 |
| DMA 发送 | HAL_UART_Transmit_DMA() | 一条命令交出去，整块搬完才产生 1 次中断，CPU 占用率最低 | 大批量、高速率传输 |

---

## 六、任务三：Keil 软件仿真逻辑分析仪观测串口波形

在没有示波器的条件下，可以用 Keil 自带的**软件仿真 + 逻辑分析仪**功能观察 USART1_TX 引脚的输出波形，并据此测算实际波特率。

![图 8  任务三  Keil 软件仿真逻辑分析仪观察 USART1_TX 波形并测算波特率](resources/usart_fig_logic_analyzer.png)

### 6.1 前置配置（不做这一步，仿真会直接报错）

Keil 默认使用通用的 Cortex-M 仿真模型（Dialog DLL = DCM3.DLL、Parameter = -pCM3），它**不模拟 STM32 的外设**：USART1 不会真的向外移位输出，逻辑分析仪上只会看到一条直线。必须先切换到 STM32 专用仿真模型，并放开外设地址空间的读写权限。

**第 1 步：切换仿真模型**

1. 打开 `STM32_USART_Task2_1` 工程，点工具栏的 **Options for Target**（魔法棒图标，快捷键 Alt+F7）。
2. 切到 **Debug** 选项卡。
3. 选中左侧的 **Use Simulator** 单选按钮（不要选右边那一组 "Use: CMSIS-DAP Debugger"）。
4. 修改左下角的两个字段：

| 字段 | 默认值 | 改为 |
| --- | --- | --- |
| Dialog DLL | DCM3.DLL | DARMSTM.DLL |
| Parameter | -pCM3 | -pSTM32F103C8 |

Parameter 里的型号必须与工程实际器件一致，本实验为 STM32F103C8。

**第 2 步：放开外设地址空间的读写权限**

若跳过这一步直接运行，Keil 会弹出错误并中止仿真：

```text
error 65: access violation at 0x40021000 : no 'read' permission
```

原因是仿真模型默认没有把外设寄存器所在的地址段映射成可读写。两种解决办法任选其一。

**办法 A（推荐，一劳永逸）**：在 `MDK-ARM` 目录下新建文本文件 `debug_map.ini`（注意扩展名是 `.ini`，不是 `.txt`），内容如下：

```text
map 0x40000000, 0x40007FFF read write   // APB1 外设
map 0x40010000, 0x40017FFF read write   // APB2 外设（USART1 挂在这里）
map 0x40018000, 0x4002FFFF read write   // AHB 外设（RCC / FLASH / EXTI 等）
map 0xE0000000, 0xE00FFFFF read write   // Cortex-M3 内核外设
```

然后在 **Options for Target → Debug → Initialization File** 中填入该文件的相对路径（如 `..\MDK-ARM\debug_map.ini`），确定保存。此后每次进入仿真都会自动执行这些映射。

**办法 B（临时，不新增文件）**：先按 Ctrl+F5 进入仿真（此时会停在报错处），再打开菜单 **Debug → Memory Map**，在 "Map Range" 中填起始地址 `0x40000000`、长度 `0x2FFFF`，勾选 **Read** 与 **Write**，点 **Map Range** 添加；再用同样方式添加 `0xE0000000`、长度 `0xFFFFF`；最后点 **Close**，按 F5 继续运行。

### 6.2 打开逻辑分析仪并添加信号

1. 先确认工程已编译通过（0 Error），否则无法进入仿真。
2. 按 **Ctrl+F5**（Start/Stop Debug Session）进入调试模式。
3. 打开菜单 **View → Analysis Windows → Logic Analyzer**（旧版 MDK 中直接是 **View → Logic Analyzer**）。
4. 在逻辑分析仪窗口左上角点 **Setup…**，弹出 "Setup Logic Analyzer" 对话框。
5. 在 "Current Logic Analyzer Signals" 列表里点 **New**（新建/插入按钮），列表中出现一个可编辑的空行。
6. 在框内输入信号名 **PORTA.9**（PA9 即 USART1_TX），按回车确认；如需对照，可再点 New 输入 **USART1_SR**（串口状态寄存器）。
7. 若列表中有 **Display** 列，把它设为 **Bit**，这样显示的是该引脚的高低电平，而不是整组端口的数值。
8. 点 **Close** 关闭 Setup 对话框。

### 6.3 调整时间轴（这一步最容易漏）

一帧 8-N-1 数据只有 10 位 ≈ 86.8 μs。若时间轴还停留在默认的秒级范围，波形会被挤成一条竖线，什么也看不出来。

1. 在逻辑分析仪窗口下方找到 **Min Time** 与 **Max Time** 两个输入框。
2. 先把 Min Time 设为 `0s`、Max Time 设为 `0.0002s`（即 200 μs），此时屏幕上大约能同时看到 2 帧。
3. 点 **Zoom In** 逐级放大，直到一个数据位在屏幕上占几十个像素；也可以把 Max Time 改成 `0.0001s`（100 μs）只保留一帧。
4. 按 **F5** 全速运行，逻辑分析仪开始采集，PA9 上立刻出现一串串方波。
5. 若画面太乱，按 **Stop**（或再按一次 F5）暂停后再放大细看。

### 6.4 用游标测量位宽并计算波特率

1. 启用两条游标：点工具栏的 **T1** 与 **T2** 按钮，或直接在波形区点击来放置游标。
2. 把 **T1** 对准某个数据位的**上升沿**，**T2** 对准相邻的**下降沿**（或对准下一个同方向的边沿），使两条游标刚好卡住**一个位**的宽度。
3. 读出两条游标的时间差 Δt（窗口中会显示各游标的时间值，两者相减即可；部分版本直接显示 Δt）。
4. 若不易精确对准单个位，也可以量**一个完整帧**的宽度（起始位下降沿到停止位上升沿，共 10 位）记作 T_frame，再按 T_bit = T_frame ÷ 10 折算。
5. 代入公式计算：

```text
波特率 = 1 / T_bit = 1 / Δt
```

6. 把结果填入 6.6 节的实测记录表。

**读不准时的两个技巧**：一是把 Max Time 缩到 100 μs，让一个位占满大半个屏幕再放游标；二是用整帧宽度折算，整帧 86.8 μs，游标对准的绝对误差被摊到 10 位上，相对误差会小很多。

### 6.5 波形判据

一个正确的 8-N-1 串口波形应当满足：

| 判据 | 正确表现 | 异常含义 |
| --- | --- | --- |
| 空闲电平 | 高电平 | 空闲为低说明接线或极性有问题 |
| 起始位 | 从高跳到低，宽度 = 1 位 | 起始位宽度不对说明波特率设错 |
| 数据位 | 8 位，**低位在前** | — |
| 停止位 | 回到高电平，宽度 = 1 位 | 停止位不足会造成帧错误 |
| 位宽均匀性 | 每一位宽度完全一致 | 忽宽忽窄说明时钟不稳或晶振配置错误 |
| 帧间隔 | 不加延时时帧与帧首尾相接 | 有间隙说明程序中加了延时或被中断打断 |

### 6.6 波特率结果与精度说明

```text
波特率 = 1 / T_bit        其中 T_bit 为一位的时间宽度
```

理论值：T_bit = 1 ÷ 115200 bps = **8.6806 μs**；一帧 10 位 ≈ 86.8 μs。

**波特率分频寄存器回读（本次实测）**

除了用逻辑分析仪量波形，还可以直接在线读回 USART1 的寄存器，从"分频值"反推实际波特率——这是一种不依赖仪器、也不受游标对准误差影响的硬证据：

| 寄存器 | 地址 | 实测值 | 含义 |
| --- | --- | --- | --- |
| USART1_BRR | 0x40013808 | 0x0271 = 625 | 分频值 625，波特率 = 72 MHz ÷ 625 = **115200 bps**，与设定值完全一致 |
| USART1_CR1 | 0x4001380C | 0x200C | bit13 UE=1（USART 使能）、bit3 TE=1（发送使能）、bit2 RE=1（接收使能） |
| RCC_APB2ENR | 0x40021018 | 0x4015 | bit14 USART1EN=1、bit2 IOPAEN=1（GPIOA 时钟）、bit0 AFIOEN=1 |

> 推算依据：STM32F1 的 USART 波特率分频公式为 BRR = f_PCLK ÷ 波特率（OVER8=0 时按 16 倍过采样）。本工程 USART1 挂在 APB2 上，PCLK2 = 72 MHz，72 000 000 ÷ 115 200 = 625 = 0x271。读回的 BRR 与理论分频值完全相同，说明**时钟树（HSE 8 MHz × PLL9 = 72 MHz）与波特率设置均正确**。

**逻辑分析仪游标记录表（需完成 6.1~6.4 的仿真操作后填写，不可编造）**

| 任务 | 设定波特率 (bps) | 理论 T_bit (μs) | 实测 Δt (μs) | 实测波特率 = 1/Δt (bps) | 相对误差 |
| --- | --- | --- | --- | --- | --- |
| 任务二(1) 中断方式 | 115200 | 8.68 |  |  |  |
| 任务二(2) DMA 方式 | 115200 | 8.68 |  |  |  |

**结论**：由寄存器回读可知，实际分频值与设定波特率 115200 bps **完全一致**；若再用游标测得 Δt ≈ 8.68 μs，即可确认起始位为低、停止位为高、每位宽度均匀，**时序状态正确**。

> 需要注意：软件仿真是由 Keil 的 ARM 指令级模型按设定的时钟频率推算时序的，它验证的是"程序逻辑与寄存器配置是否正确"，而不是真实晶振的精度。若要考察真实硬件的波特率误差，还需要用示波器或逻辑分析仪测量实际引脚，并考虑 HSE 晶振的精度与负载电容匹配情况。

---

## 七、实验现象与数据记录

### 7.1 四个任务的实验现象

| 任务 | 预期现象 | 实测记录 |
| --- | --- | --- |
| 任务一(1) | 串口助手持续刷屏显示 hello windows!，帧间隔约 1.22 ms，每秒约 823 帧 | **已完成（SWD 在线验证）**：从 Flash 地址 0x0800139C 起逐字节读出的内容与源码中的 `hello windows!` 一致；USART1_BRR = 0x0271（分频 625，对应 115200 bps）；PA9 配置为复用推挽输出；程序运行期间反复采样 GPIOA_IDR，PA9（bit9）电平持续翻转，证明**正在连续发送** |
| 任务一(1) 变体 | 把发送改为加 10ms 延时后，刷屏变得可读，丢包消失 | 本次未单独测试该变体，留空 |
| 任务一(2) | 发送 # 后停止发送，发送 * 后继续发送 | **已完成（SWD 在线验证）**：上电后 PA9 持续翻转＝正在发送；从 COM8 发送字符 `#` 后再采样，PA9 恒为高电平（UART 空闲电平）＝**已暂停**；随后发送 `*`，PA9 重新翻转＝**已恢复** |
| 任务二(1) | 现象与任务一(2) 完全一致，但接收由中断完成 | **已完成（SWD 在线验证）**：与本任务一(2) 现象完全一致——发 `#` 后 PA9 恒为高电平（暂停），发 `*` 后恢复翻转（继续发送）；回读 USART1_CR1 = 0x200C（UE/TE/RE 均置 1），说明 USART1 已使能并处于收发状态 |
| 任务二(2) | 串口助手持续收到 DMA 测试字符串，帧间隔约 4.25 ms | **已完成（SWD 在线验证）**：烧录 DMA 工程后复位运行，PA9 持续翻转＝DMA 连续发送正常；帧长 49 字节，一帧 49×10÷115200 ≈ 4.25 ms |
| 任务三 | 逻辑分析仪上看到规整的 8-N-1 方波，游标测得 T_bit ≈ 8.68 μs | 未运行 Keil 软件仿真；本次改用 **SWD 回读 USART1_BRR = 0x0271**（分频 625），即 72 MHz ÷ 625 = 115200 bps，与设定值一致（见 6.6）。波形游标读数未采集，留空 |

> **实测方法说明**：本实验中 PC 侧串口助手在"单片机→PC"方向始终收不到数据（原因见 7.3），因此改用 **DAP-Link + SWD 在线观测** 的方式完成四个任务的功能验证——直接读取芯片内部的发送寄存器与 PA9 引脚状态，其证据强度不低于串口助手截图。

### 7.2 建议在报告中附上的实测截图

1. 设备管理器中 CH340 虚拟串口的截图。
2. 串口助手收到 hello windows! 的界面截图（任务一(1)）。
3. 发送 # 后停止、发送 * 后恢复的串口助手截图（任务一(2) 与任务二(1) 各一张）。
4. Keil 逻辑分析仪的波形截图，含游标读数（任务三）。
5. Keil 编译输出窗口截图，显示 Program Size 与 0 Error(s), 0 Warning(s)。
6. 本报告 7.1 实测数据的来源记录：DAP-Link + SWD 反复采样 PA9（USART1_TX）电平、回读 USART1 寄存器（BRR / CR1）的结果截图。

### 7.3 关于"单片机→上位机"方向未收到数据的说明（如实记录）

本次实验全过程中，PC 侧串口助手在**"单片机→PC"方向始终收不到任何字节**（CH340 对应的 COM8 与 DAP-Link 虚拟串口 COM7 均收到 0 字节）。为定位问题，依次做了以下排查，并**排除了**如下可能：

| 排查项 | 结果 | 结论 |
| --- | --- | --- |
| 芯片与程序是否真的在发 | SWD 采样 PA9 持续翻转；Flash 中确实存在 `hello windows!` 字符串；BRR = 0x0271 | 发送逻辑、数据与波特率均正确，**排除** |
| 板端 PA9 排针 | 可在 PA9 排针处观测到持续翻转的电平 | 引脚本身正常，**排除** |
| USB-TTL(CH340) 模块本身 | 模块 TXD↔RXD 自环回测试收发正常 | 模块收发通路基本正常，**排除** |
| 模块与板子的连接 | 将模块 RXD 与 STM32 的 PA9 短接后，回读 USART1_SR 的 ORE 位为 1 | 串口线路在工作，说明并非"完全无信号" |
| PC → 单片机方向 | 从 COM8 发送 0xA5，回读 USART1_DR 得到 0xA5 | **反方向通信正常** |

综合以上结果，问题应出在**该 USB-TTL 模块 RXD 输入侧的接收门限或接线环节**（TTL 输入电平判据未被满足）。由于本实验的教学目的在**三种收发方式的程序实现与寄存器配置**，而非修复某一根杜邦线，最终改用 **SWD 在线观测 PA9 波形 + 回读串口寄存器** 的方式完成了全部功能验证（即 7.1 表中的实测记录），未继续排查该硬件链路。

> 这条记录本身也是一条结论：**判断"单片机有没有在发数据"，不能只看 PC 侧串口助手有没有显示**——用调试器直接读引脚电平和发送寄存器，比串口助手的显示更接近真相。

---

## 八、问题与解决过程

| 问题 | 原因分析 | 解决办法 |
| --- | --- | --- |
| 串口助手收到的全是乱码 | 单片机与上位机波特率不一致 | 两端统一设置为 115200 / 8 / 1 / 无校验 |
| 一个字符都收不到 | TXD/RXD 没有交叉，或未共地 | 按图 3 检查：模块 TXD→PA10、RXD→PA9、GND↔GND |
| 收到一次 # 后就再也不响应 | HAL 的中断接收是一次性的，收完未重新武装 | 在 HAL_UART_RxCpltCallback() 末尾重新调用 HAL_UART_Receive_IT() |
| 发送快时偶发丢字符 | ORE 溢出标志置位后未清除 | 增加 ORE 判断与清除，或在回调中处理 HAL_UART_ErrorCallback() |
| 收到 # 后主循环不停 | 标志变量未加 volatile，被编译器优化掉 | 给 send_enable 加 volatile |
| DMA 发送只发出一帧就停住 | 漏写 USART1_IRQHandler，TC 中断无人处理 | 补上 USART1_IRQHandler 与对应的 NVIC 使能 |
| Keil 提示找不到仿真器 | Debug 页默认是 ULINK2/ME，与实际使用的 DAP-Link 不符 | 在 Options for Target → Debug 中选择 CMSIS-DAP Debugger（四个工程已预置） |
| 串口 ISP 烧录始终超时无应答 | 板子 BOOT0 跳线未真正拉高，芯片没进引导程序 | 改用 DAP-Link + SWD（见 3.3.1），完全绕开 BOOT0 |
| STM32CubeProgrammer 连不上 DAP-Link | 该软件不支持 CMSIS-DAP 接口 | 改用 Keil 的 CMSIS-DAP Debugger 或 pyOCD |
| 串口 ISP 下载提示无应答 | BOOT0 未置 1、改后未复位，或模块 3V3 供电电流不足 | BOOT0=1 后按复位；模块 3V3 悬空，改用 micro-USB 独立给板子供电 |
| 仿真一运行就报 error 65: access violation，程序自动停止 | 仿真模型没有把外设地址段映射为可读写 | 按 6.1 的办法 A 添加 debug_map.ini，或用 Debug → Memory Map 手动映射 |
| 逻辑分析仪里 PORTA.9 始终是一条直线 | Dialog DLL 仍是 DCM3.DLL，通用 Cortex-M 模型不模拟 STM32 外设 | 按 6.1 第 1 步把 Dialog DLL 改为 DARMSTM.DLL、Parameter 改为 -pSTM32F103C8 |
| 逻辑分析仪上看不出方波，只有一条竖线 | 时间轴范围太大，86.8 μs 的一帧被压缩 | 按 6.3 节把 Max Time 设为 0.0002s 并 Zoom In |
| 用 arm-none-eabi-gcc 编译的镜像一上电就进 HardFault（CFSR = 0x00010000，UNDEFINSTR） | GCC 链接脚本/启动流程与 C 库初始化不匹配，程序在 `__libc_init_array` 阶段跳出镜像末尾，执行了空白 Flash（0xFFFFFFFF），取下非法指令 | 改用 **Keil MDK** 编译：`UV4 -b <工程.uvprojx> -o <日志>`，四个工程均 0 Error 0 Warning，镜像正常运行（见 3.3.1） |
| 单片机→PC 方向串口助手始终收不到数据 | 已排除芯片/程序/PA9 引脚/模块自环回，问题定位在模块 RXD 输入侧的门限或接线环节 | 反方向 PC→单片机 正常；改用 SWD 在线观测 PA9 电平 + 回读串口寄存器完成任务验证（见 7.3） |

---

## 九、实验总结

1. **协议层面**：弄清了一个字节在异步串口上要占 10 位，这是所有估算的基础；也理解了 RS-232 的负逻辑与高压摆幅、RS-485 的差分传输为什么能传得更远，以及 TTL 电平为什么不能直接接 RS-232 设备。
2. **工具层面**：CH340 模块把复杂的 USB 协议封装成简单的虚拟串口，让上层软件只需读写 COM 口，这正是"协议转换芯片"的价值所在。
3. **编程层面**：同一个功能用查询、中断、DMA 三种方式实现，代码从"CPU 死等"到"硬件主动报信"再到"CPU 只下命令"，体现了嵌入式系统中"把重复劳动交给硬件"的一贯思路。三个工程的现象完全一致，差别只在效率。
4. **工程细节**：本次实验暴露了不少易错点——中断接收的一次性武装、ORE 溢出标志的清除、volatile 的必要性、DMA 需要同时开两个中断、字符串发送长度要减去结尾的 `'\0'`。这些细节正是从"能跑"到"稳定可靠"的关键。
5. **调试方法**：在没有示波器时，Keil 的软件仿真逻辑分析仪同样能观察引脚波形、验证时序与波特率，说明调试手段不限于昂贵的仪器，善用现有工具同样能定位问题。
6. **工具链与验证手段**：本次踩到一个相当隐蔽的坑——**同一份源码用不同工具链编译，运行行为可能完全不同**（GCC 镜像上电即 HardFault，Keil 镜像正常）。定位它靠的不是猜测，而是用调试器读内核故障寄存器（CFSR）拿到直接证据。这一点也再次说明：当上位机界面"看不到现象"时，直接读取芯片寄存器与引脚电平，往往比界面显示更接近真相。
