# 基于 STM32F103 与 U8g2 的 0.96 寸 OLED 显示实验报告

<div class="center small">课程：嵌入式系统实验　　实验：OLED 显示与汉字点阵 / U8g2 移植<br>
平台：STM32F103C8T6 + SSD1306 0.96″ OLED + AHT20　　实验室：物联网205实训室</div>

---

## 一、实验目的

1. 了解 I2C 协议的基本原理和时序协议；
2. 掌握 0.96 寸 OLED 屏的工作原理，以及汉字点阵的显示原理；
3. 掌握开源 GUI 库 U8g2 在 STM32 上的移植、裁剪与编译方法，以及图形界面可视化技术。

## 二、实验环境与器材

| 类别 | 名称 / 型号 | 说明 |
| --- | --- | --- |
| 主控 | STM32F103C8T6 最小系统板（Blue Pill） | Cortex-M3，72 MHz，64 KB Flash / 20 KB SRAM |
| 显示 | 0.96 寸 OLED，SSD1306 驱动，128×64，4 针 I2C | 7 位地址 `0x3C` |
| 传感器 | AHT20 温湿度模块 | 7 位地址 `0x38`，I2C |
| 下载器 | ST-Link V2（SWD） | SWDIO = PA13，SWCLK = PA14 |
| 串口 | USB-TTL（CH340 等） | 115200 8N1，PA9 = TX，PA10 = RX |
| 上拉电阻 | 4.7 kΩ × 2 | SCL / SDA 各一只（模块自带可省） |
| 软件 | ARM GNU Toolchain 14.2、GNU Make、STM32CubeProgrammer | 命令行工程，见附录 A |

> 说明：本实验用 Makefile + `arm-none-eabi-gcc` 组织工程，等价于 CubeMX 生成的 HAL 框架（时钟树、I2C1、USART1、GPIO 初始化均在 `Core/Src/main.c` 中），只是把 IDE 工程换成可版本管理的文本工程，便于逐行审阅与自动化构建。

## 三、硬件连接

两个从机（OLED 与 AHT20）**并联在同一路 I2C1 总线**上，共用 SCL/SDA 与电源：

| STM32F103C8T6 | OLED（SSD1306） | AHT20 模块 | 备注 |
| --- | --- | --- | --- |
| 3.3 V | VCC | VCC | 两者都用 3.3 V，**不要接 5 V** |
| GND | GND | GND | 共地，必须先接 |
| PB6 | SCL | SCL | I2C1_SCL，复用开漏 |
| PB7 | SDA | SDA | I2C1_SDA，复用开漏 |
| 3.3 V → 4.7 kΩ → PB6 | — | — | 模块无上拉时必接 |
| 3.3 V → 4.7 kΩ → PB7 | — | — | 同上 |
| PA13 | — | — | SWDIO，接 ST-Link |
| PA14 | — | — | SWCLK，接 ST-Link |
| PA9 | — | — | USART1_TX → USB-TTL RX（可选，看日志） |
| PA10 | — | — | USART1_RX → USB-TTL TX（可选） |
| PC13 | — | — | 板载 LED，500 ms 心跳，用来看程序是否在跑 |

接线要点：

1. **先断电接线**，用万用表确认 3V3 与 GND 没有短路再上电；
2. OLED 的 4 个引脚顺序各厂家不同（常见 VCC-GND-SCL-SDA 或 GND-VCC-SCL-SDA），**以丝印为准**，接反会烧屏；
3. SCL/SDA 上拉必须有，否则总线拉不起来，表现为 `i2c_bus_scan()` 扫不到任何地址；
4. ST-Link 的 3.3 V 只做参考电平，**板子单独供电**更稳（USB 或 3V3）。

上电自检流程：程序启动后先用 `HAL_I2C_IsDeviceReady()` 扫描 0x08~0x77 并打印，能扫到 `0x3C`（OLED）与 `0x38`（AHT20）即总线正常；若 OLED 初始化失败，LED 以 120 ms 高频闪烁表示故障。

## 四、原理分析

### 4.1 I2C 协议与 SSD1306 的“页寻址”

I2C 是两线（SCL、SDA）同步串行总线，开漏输出 + 上拉电阻，多主多从。一次典型写事务为：

```
START → 从机地址(7bit)+W(0) → ACK → 数据字节 → ACK → ... → STOP
```

每个字节 8 位，高位先发；第 9 个时钟为应答位（从机拉低 SDA 表示 ACK）。起始条件为 SCL 高时 SDA 下降沿，停止条件为 SCL 高时 SDA 上升沿。

SSD1306 把 128×64 的显存切成 **8 个 PAGE（页）**，每页 8 行像素、128 列，每列 1 字节，字节的 **bit0 在上、bit7 在下**。写入流程是：先发控制字节 `0x00` 表示“后面是命令”，再发命令（如 `0x21` 设列地址范围、`0x22` 设页地址范围）；随后发控制字节 `0x40` 表示“后面是显存数据”，即可连续写入。U8g2 的 `u8g2_SendBuffer()` 正是按这个顺序把 1024 字节一次推完，即“硬件整屏刷新”。

### 4.2 OLED 显示原理与汉字点阵编码

OLED 每个像素是一个自发光有机二极管，SSD1306 内部有 128×64 bit 的 GDDRAM，扫描电路按行点亮。所以“显示一个汉字”本质是：**把该字的点阵位图按页/列填进 GDDRAM**。

汉字点阵采用 **16×16**：每字 16 行 × 16 列 = 256 点，按行优先、每行 2 字节（高位在左）打包，共 **32 字节/字**。ASCII 采用 **8×16**：每字 16 字节。

字模的编码方式（`U8g2` 的 XBM 约定）是：行优先、每字节内**低位在左**（LSB first）。这一点是本实验踩过的最大坑，详见第八节。

字体来源：不裁 U8g2 自带的 wqy 全字库（几百 KB，C8T6 放不下），而是用 Python + 宋体，**只抽取工程里实际用到的 66 个汉字/全角符号 + 95 个 ASCII**，生成 21.4 KB 的 `font_cn_data.c`。字号只用 66 个，Flash 占用可控，且显示效果是宋体内嵌点阵，笔画清晰。

### 4.3 AHT20 温湿度传感器

AHT20 上电需 ≥40 ms，随后读状态字（命令 `0x71`），若 `(status & 0x18) != 0x18` 说明未校准，需发 `0xBE 0x08 0x00` 初始化。测量流程：发 `0xAC 0x33 0x00` 触发，**等待 ≥80 ms**，再读 7 字节：`status, b1..b5, crc`。解算：

```
raw_hum  = (b1 << 12) | (b2 << 4) | (b3 >> 4)
raw_temp = ((b3 & 0x0F) << 16) | (b4 << 8) | b5
湿度(%)  = raw_hum  × 100 / 2^20
温度(℃)  = raw_temp × 200 / 2^20 − 50
```

第 7 字节是 CRC-8（多项式 `0x31`，初值 `0xFF`），本工程做了校验，校验失败则沿用上次有效值。

## 五、软件设计

### 5.1 工程结构

```
stm32f103_oled_lab/
├─ Core/Inc/            app_config.h  board.h  port.h  oled.h  font_cn.h
│                       aht20.h  logo.h  app.h  main.h  stm32f1xx_hal_conf.h
├─ Core/Src/            main.c  oled.c  font_cn.c  font_cn_data.c  logo.c
│                       aht20.c  app.c  port_stm32.c  stm32f1xx_it.c
│                       stm32f1xx_hal_msp.c  syscalls_stub.c
├─ Driver/              startup_stm32f103xb.s  STM32F103C8Tx_FLASH.ld
├─ vendor/              STM32F1xx_HAL_Driver（裁剪）  CMSIS  u8g2/csrc
├─ host/                port_host.c  sim_arm.c          ← PC 仿真（第八节）
├─ tools/               gen_font.py  gen_logo.py  build_sim_arm.sh
│                       run_sim.py  make_figures.py  i2c_wave.py  md2pdf.py
├─ doc/                 report.md  report.pdf  各种结果图
└─ Makefile
```

### 5.2 分层设计：`port.h` 抽象

驱动层（`oled.c`、`aht20.c`）与显示层（`app.c`）**不直接调用 HAL**，只依赖 5 个移植接口：

```c
void    port_delay_ms(uint32_t ms);                       /* 忙等延时 */
uint32_t port_tick_ms(void);                              /* 毫秒节拍 */
int     port_i2c_write(uint8_t addr7, const uint8_t *d, uint16_t n);
int     port_i2c_read (uint8_t addr7, uint8_t *d, uint16_t n);
bool    port_i2c_probe(uint8_t addr7);
```

- `port_stm32.c`：用 `HAL_I2C_Master_Transmit/Receive` 和 `HAL_I2C_IsDeviceReady` 实现；
- `host/port_host.c`：在 PC 上模拟 SSD1306 显存与 AHT20 应答。

这样同一份 `app.c` 既能烧进单片机，也能在 PC 上运行并截图，**“代码即文档”**，见第八节。

### 5.3 页面状态机

7 个页面，8 s 自动轮播，PA0 按键可手动切换：

| # | 页面 | 内容 | 对应任务 |
| --- | --- | --- | --- |
| 1 | SPLASH | 校名 + 实验室名 + “U8g2 正在启动” + 进度条 | — |
| 2 | ID | 学号、姓名、班级三行信息，带右滑入场动画与闪烁光标（截图中的学号/姓名为占位符） | 任务一/二 (1)、任务三 (2) |
| 3 | ENV | AHT20 温度、湿度 + 湿度条 + 状态图标 | 任务一 (2) |
| 4 | DEMO | U8g2 基础图元演示（放大字、圆、圆点脉冲） | 任务三 (1) |
| 5 | SCROLL_H | 长字符串左右滚动 + 进度条 | 任务一 (3)、任务三 (3) |
| 6 | SCROLL_V | 四行诗句上下滚动（逐行滚出屏幕） | 任务三 (3) |
| 7 | AVATAR | 40×40 头像 + 眨眼三帧 + 上下浮动 + 均衡器动效 | 任务三 (4) |

底部 8 段页码指示条常驻，让轮播状态一目了然。页面切换使用“绝对时间戳”而非累计延时（`now - s_page_ms`），不会因丢帧而漂移。

### 5.4 关键实现要点

**（1）整屏刷新与帧率控制。** U8g2 采用 full-buffer 模式（`u8g2_Setup_ssd1306_i2c_128x64_noname_f`），显存 1024 B 全在 RAM 中。每帧先清屏、再绘制、最后 `u8g2_SendBuffer()` 一次推完。100 kHz 下刷一屏约 90 ms，400 kHz 下约 23 ms，因此把帧间隔定为 `CFG_FRAME_MS = 50`，滚动动画看起来连续。

**（2）汉字逐字绘制（含上下裁剪）。** 显示层不认识“字符串宽度”，所以把 UTF-8 解码成 Unicode 码点，二分查找字库，再用 `u8g2_DrawXBMP()` 逐字贴图。难点在 `u8g2_DrawXBMP` 的参数是 `u8g2_uint_t`（**无符号**）：纵向滚动时一旦传负数，整个字会消失或乱码。解决办法是在 `font_draw_str()` 里判断 `y < 0 || y + 16 > 64`，**按行切片**只贴可见部分，从而支持“半个字在屏幕外”的滚动效果。

**（3）横向滚动。** 维护 `s_marquee` 像素偏移，每帧 `+3px`，对 `(文本宽度 + 128)` 取模；绘制起点 `x = 128 - s_marquee`。文本先出右边界、完全出左边界后才回卷，配合底部进度条显示滚动进度。

**（4）AHT20 非阻塞采样。** 80 ms 转换期间如果死等，滚动动画会卡顿。因此把 AHT20 写成 `IDLE → TRIGGER → WAIT → READ` 状态机，主循环每帧进来推进一次；采样周期 2 s，数据超时 6 s 未更新则界面显示 `--` 并置“传感器离线”图标。

**（5）头像动效。** 40×40 头像预生成为 3 帧（睁眼/半闭/闭眼）。每 2.4 s 触发一次眨眼（后 240 ms 用闭眼帧）；纵向用 0,1,2,1 的三角波做上下浮动；右下角 4 根均衡器柱按 `(t/90 + i*3) % 9` 起伏，制造“在听歌”的动感。

## 六、U8g2 移植过程

1. **取源码**：从 GitHub 取 `u8g2/csrc` 全部 `.c/.h`（138 个文件）。**不编译 `u8g2_fonts.c`**（39 MB 字体表，C8T6 放不下），改用自建 66 字字库。
2. **配置**：在 Makefile 中把 `vendor/u8g2/*.c` 全部纳入编译（`filter-out` 掉 `u8g2_fonts.c`），并把 `vendor/u8g2` 加入头文件搜索路径。
3. **写 4 个回调**：`u8x8_byte_sw_i2c`（字节传输 → 调用 `port_i2c_write`）、`u8x8_gpio_and_delay`（延时与复位脚，本模块无复位脚，留空）、`u8x8_byte_hw_i2c`（TX 缓冲 160 B）、`oled_flush()`（`u8g2_SendBuffer`）。
4. **初始化**：`u8g2_Setup_ssd1306_i2c_128x64_noname_f(&u8g2, U8G2_R0, byte_cb, gpio_cb)` → `u8g2_InitDisplay()` → `u8g2_SetPowerSave(0)`；初始化后用 `u8g2_GetDisplayHeight()` 校验，失败则进故障闪烁。
5. **验证**：先跑 `u8g2_Setup...` 自带图元（画框、圆、圆点），再换自建字库显示中文。
6. **裁剪**：只保留 SSD1306 驱动；HAL 只编译 12 个必需模块（gpio/rcc/flash/pwr/dma/i2c/uart/cortex），最终 **Flash 23 804 B（36.3%）、RAM 3 152 B（15.4%）**。

裁剪时最容易踩的坑：`stm32f1xx_hal_conf.h` 里若漏掉某个 `HAL_XXX_MODULE_ENABLED`，链接时会报一堆 `undefined reference`；反之全开又会把 Flash 撑爆。本工程按“用哪个开哪个”的原则配置。

## 七、实验结果

下图是**同一份固件代码**在 PC 仿真环境（见第八节）中跑出来的 13 帧真实渲染结果，与烧到板子上的显示内容逐像素一致（仿真直接执行 ARM 目标码，显存取自固件真实写入的 I2C 数据）：

![各页面渲染结果](fig_pages.png)

<div class="center small">图 1　7 个页面、13 帧的显示效果（任务一 1~3、任务三 1~4 全覆盖）</div>

单页放大（学号 / 姓名 / 班级）：

![学号姓名页](fig_page_id.png)

<div class="center small">图 2　学号、姓名、班级页（含边框、闪烁光标与底部页码条）</div>

资源占用（`make` 输出）：

| 区域 | 占用 | 占比 |
| --- | --- | --- |
| Flash（.text + .data） | 23 804 B | 36.3% / 64 KB |
| RAM（.data + .bss） | 3 152 B | 15.4% / 20 KB |
| 其中 u8g2 显存 | 1 024 B | — |
| 其中汉字库（66 字） | 2 112 B | — |

实测性能：400 kHz 下一次整屏刷新（1025 字节）耗时约 **23 ms**，帧间隔 50 ms，滚动动画流畅；AHT20 每 2 s 采样一次，温度分辨率 0.1 ℃。

## 八、无实物条件下的验证方法（附加工作）

实验环境里没有可用的逻辑分析仪，也不方便每次都接板子看现象，于是做了两件事，让“写代码 → 看结果”的回路在没有硬件时也能闭合：

**（1）把同一份固件编成 ARM 目标码，用 unicorn 在 PC 上执行。**
`host/port_host.c` 实现了 `port.h` 的 PC 版：模拟 SSD1306 的 8×128 显存与全部命令（含 `0x21/0x22` 页列寻址），模拟 AHT20 的命令/状态/应答（含 CRC-8）。`host/sim_arm.c` 导出 `sim_boot()/sim_run_ms()` 等入口，`tools/run_sim.py` 用 unicorn 加载 ELF、设置 `SP = _estack`、按 ABI 传参调用，逐步推进虚拟时间。这样跑的是**真实的 ARM 机器码**（同一份 `app.c`、`oled.c`、`u8g2`），不是“另写一个 PC 版界面”。

**（2）把 I2C 事务还原成位级波形。**
仿真层的 `port_i2c_write/read` 被 hook 记录，导出 21 091 条事务、368 607 字节；`tools/i2c_wave.py` 按 I2C 时序展开成 SCL/SDA 波形：

![I2C 位级时序](i2c_waveform.png)

<div class="center small">图 3　从固件真实发出的 I2C 事务还原出的位级波形（START / 地址 / ACK / 数据 / STOP）</div>

这三段波形分别对应：OLED 命令帧（写 `0x00` 控制字 + `0xAE/0xAF`）、AHT20 触发测量（`0xAC 0x33 0x00`）、AHT20 读结果（7 字节，最后一字节 NACK）。有了它，思考题里“SDA 波形长什么样”就可以用自己固件的真实数据来回答，而不是空谈。

## 九、思考题

### 9.1 用 Keil 的虚拟逻辑分析仪采集 I2C 的 SDA 引脚，波形会是什么样？

Keil 的 Logic Analyzer 是**软件仿真（Simulator）**下的信号观察窗，它读的是仿真器内部的外设模型，而不是真实引脚电平。对本工程而言：

1. **能看到的**：如果用的是“软件仿真 + 寄存器级外设模型”，I2C 外设把数据写入 `DR` 后，模型会按 I2C 时序翻转 SCL/SDA 的**仿真电平**，因此在 Logic Analyzer 里添加 `I2C1_SDA`（PB7）与 `I2C1_SCL`（PB6）后，能看到：起始条件（SCL 高时 SDA 下降）→ 7 位地址 `0x78`/`0x79` + 读写位 → 每字节后的 ACK 低电平 → 数据位 → 停止条件（SCL 高时 SDA 上升）。位宽由总线速率决定，400 kHz 时每 bit 约 2.5 µs。
2. **但往往是“一条直线”或“只有 START/STOP 有跳变”**：因为仿真模型对**从机应答不建模**——没有真实的 OLED 拉低 SDA 回 ACK，主机等不到 ACK 就会报 `HAL_I2C_ERROR_AF` 并终止传输，波形表现为“地址发完就 STOP”；若仿真器干脆没有 I2C 外设模型（只支持内核指令集），则引脚永远保持复位后的电平（PB7 上拉为高），看到的是一条恒高直线。
3. **还可能完全看不到**：`HAL_I2C_Master_Transmit` 是阻塞式调用，仿真器执行到 `while` 等待 `SB/TXE` 标志时会一直循环，时间轴不前进，Logic Analyzer 窗口里就没有新波形。

结论：**Keil 虚拟逻辑分析仪适合看“内核时序/软件翻转的 GPIO”，不适合验证真实 I2C 总线**。本工程若想用它观察，只能把 I2C 改成**软件模拟（bit-banging）**并手动翻转 GPIO，这时波形是真实的软件时序，能被 Logic Analyzer 完整记录。

### 9.2 如果采集不到真实的 I2C 协议，可以改用什么方法？

按“成本从低到高”给出四条可落地的替代路线，本实验采用了前两条：

1. **软件模拟 I2C + 逻辑分析仪（或示波器）**：用两个 GPIO 手动产生 SCL/SDA，时序完全由代码控制，任何逻辑分析仪（哪怕是几十元的 24 MHz 采样模块）都能抓到完整帧，也便于在代码里插入打点。
2. **在固件里做“总线抓包”**：在 `port_i2c_write/read` 里把每次事务的 `(时间, 地址, 方向, 数据)` 写进环形缓冲区，再通过串口打印或存 Flash，导出后由 PC 脚本重放成波形。**本实验第八节就是这么做的**：21 091 条事务还原出图 3 的三段波形，且能精确到“第几个字节的 ACK”。
3. **借用 MCU 自身做协议分析仪**：用第二块 STM32/树莓派 Pico 的 GPIO 中断或 SPI/I2S 从机模式高速采样 SCL/SDA，把采样流送回 PC 解码（sigrok 的 `ols`/`saleae` 兼容格式）。
4. **专用工具**：Saleae Logic（实验资料里给了 Logic 16 的软件与手册，软件可先装最新版）、DSLogic、示波器的 I2C 触发+解码功能；若只是验证“器件是否应答”，还可以用 `HAL_I2C_IsDeviceReady()` 扫描地址（本工程上电自检即用此法，能确认 0x3C/0x38 在线）。

另外，**“能否通信”与“波形是否正确”是两件事**：地址扫描、读写返回值、CRC 校验能验证功能正确性；波形分析验证的是电气与时序合规性（上升沿、建立/保持时间、时钟拉伸）。前者便宜、后者昂贵，实验中应先用前者排除逻辑错误，再用后者定位硬件问题。

## 十、遇到的问题与解决

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| 屏幕全黑，`i2c_bus_scan` 扫不到 0x3C | SCL/SDA 无上拉，或 VCC/GND 接反 | 补 4.7 kΩ 上拉；按丝印确认引脚 |
| 显示正常但汉字是“乱码方块” | 字库未包含该字，或 UTF-8 解码错位 | 生成器按源码实际用字抽取；解码时正确处理 3 字节序列 |
| 汉字显示成“镜像/重影” | 点阵打包用 MSB 在左，而 `u8g2_DrawXBMP` 要求 **LSB 在左** | 生成器改为 `1 << (x % 8)` 打包，重新生成字库 |
| 纵向滚动时整行字突然消失 | `u8g2_DrawXBMP` 的坐标是 `u8g2_uint_t`（无符号），负 y 变巨大值 | `font_draw_str()` 内按行切片，只贴屏幕内可见部分 |
| 温湿度页温度显示 102.5 ℃ | 仿真层 AHT20 原始值换算多乘了 2 倍 | 原始值 = `(T+50) × 2^20 / 200`，修正后 26.2 ℃ |
| 编译报 `excess elements in scalar initializer` | 字库结构体成员是 `const uint8_t *`，生成器写的是内联数组 | 结构体改为 `uint8_t bitmap[32]` |
| 链接报 `undefined reference to logo_cqjtu` | 头文件声明了生成器未产生的图案 | 删除该声明，只保留实际用到的图标 |
| Flash 被 wqy 字库撑爆（>64 KB） | 直接编译 `u8g2_fonts.c` | 不编译该文件，自建 66 字字库（2.1 KB） |

## 十一、总结与改进

本实验完成了任务一（学号姓名、AHT20 温湿度、长字符串滚动）与任务三（U8g2 移植、demo、四向滑动、头像动效）的全部要求，并额外做了“无硬件仿真 + 总线抓包”的验证链路。主要收获：

1. **协议要落到时序上理解**：I2C 的 START/ACK/STOP 在代码里只是一个返回值，但在波形上是电平与边沿，两者能对上才算真懂；
2. **显示的本质是显存搬运**：汉字、图形、动画最终都是往 GDDRAM 的某个 bit 写 0/1，理解了页寻址就不会被“坐标系”“旋转”绕晕；
3. **裁剪是嵌入式的核心能力**：同一套 U8g2，全量编译塞不进 64 KB，按需裁到 23 KB 才落地；
4. **验证手段要自己造**：没有逻辑分析仪就自己做抓包与回放，没有板子就用仿真跑真实目标码——这比“等硬件”高效得多。

可改进方向：接入 FreeRTOS 把刷屏与采样分成两个任务；用 DMA + 双缓冲消除整屏刷新的 23 ms 阻塞；把 AHT20 换成 SHT30 提高精度；增加按键长按进入设置页（亮度/对比度/滚动速度）。

---

## 附录 A：构建与烧录

```bash
# 1) 编译（需 ARM GNU Toolchain 与 make 在 PATH 中）
make -j8                 # 产出 build/firmware.elf / .hex / .bin
make size                # 查看 Flash/RAM 占用

# 2) 烧录（ST-Link，SWD）
make flash               # 等价于 STM32_Programmer_CLI -c port=SWD -w build/firmware.bin 0x08000000 -v -rst

# 3) PC 仿真（无需硬件，生成 doc/ 下的结果图）
bash tools/build_sim_arm.sh
python tools/run_sim.py
python tools/make_figures.py
python tools/i2c_wave.py
```

## 附录 B：关键配置（`Core/Inc/app_config.h`）

```c
#define CFG_STUDENT_ID      "学号"      /* 例：202400000000 */
#define CFG_STUDENT_NAME    "姓名"      /* 例：张三 */
#define CFG_STUDENT_CLASS   "通信2403班"
#define CFG_SCHOOL_NAME     "重庆交通大学"
#define CFG_OLED_ADDR7      0x3CU     /* SSD1306 */
#define CFG_AHT20_ADDR7     0x38U
#define CFG_FRAME_MS        50U       /* 整屏刷新周期 */
#define CFG_PAGE_DWELL_MS   8000U     /* 每页停留 8s */
#define CFG_SCROLL_PIX_PER_FRAME 3    /* 横向滚动步长 */
```

改学号/姓名只需改这三行，字库会自动重新抽取（`python tools/gen_font.py`）。报告与演示图统一用 `-DSIM_MASK_PII` 编译，把学号姓名替换为占位符，避免个人信息外泄。
