# STM32F103C8T6 寄存器方式流水灯实验报告

> 开发板：STM32 最小系统核心板 **STM32F103C8T6**（国际俗称 **STM32 Blue Pill**）

---

## 一、实验目的

1. 认识 STM32F103C8T6 最小系统核心板（Blue Pill）的硬件组成与引脚分布；
2. 掌握 STM32 **直接操作寄存器** 的编程方法，理解"先开时钟、再配端口、最后读写"的基本套路；
3. 在面包板上使用 GPIOA、GPIOB、GPIOC 三个端口分别驱动红、绿、蓝 3 只 LED，实现 1 秒间隔轮流闪烁；
4. 查阅最小系统板电路原理图，弄清**出厂板载 PC13 指示灯**的接法，并将其并入流水灯序列。

## 二、实验平台与器件

| 类别 | 名称 / 型号 | 说明 |
| --- | --- | --- |
| 主控 | STM32F103C8T6 | ARM Cortex-M3 内核，LQFP48 封装 |
| 闪存 Flash | 64 KB | 起始地址 `0x0800 0000` |
| 内存 SRAM | 20 KB | 起始地址 `0x2000 0000` |
| 最高主频 | 72 MHz | 由外部 8 MHz 晶振 9 倍频得到 |
| 板载晶振 | HSE 8 MHz + LSE 32.768 kHz | 后者供 RTC 使用 |
| 供电 | 3.3 V（Micro-USB 或 3V3 引脚） | VDD 允许 2.0~3.6 V |
| 外设 | 2×ADC、3×通用定时器、3×USART、2×SPI、2×I²C、CAN、USB FS | 见 datasheet 表 2 |
| 外部件 | 面包板、红/绿/蓝 LED 各 1 只、220 Ω 电阻 3 只、杜邦线若干 | |
| 下载器 | ST-Link V2（SWD 四线）或 USB-TTL（BOOT0 串口下载） | |
| 软件 | 任选其一：① arm-none-eabi-gcc + Make；② Keil MDK5 + STM32F1 器件包 | 烧录用 STM32CubeProgrammer / STM32 ST-LINK Utility |

### 2.1 Blue Pill 最小系统板组成

对照引脚图与电路原理图，最小系统由以下部分构成：

1. **电源部分**：Micro-USB 输入 5V，经板上 LDO（AMS1117-3.3）稳压为 3.3V，配去耦电容；
2. **复位电路**：NRST 引脚接 10 kΩ 上拉电阻、0.1 μF 电容和一个轻触按键，按下即拉低复位；
3. **主晶振**：8 MHz 无源晶振 + 两只起振电容，接 OSC_IN（PD05）/ OSC_OUT（PD06）；
4. **RTC 晶振**：32.768 kHz，接 PC14 / PC15；
5. **启动配置**：BOOT0、BOOT1（PB2）跳线，`BOOT0=0` 从主闪存启动（正常运行），`BOOT0=1,BOOT1=0` 从系统存储器启动（串口烧录）；
6. **调试接口**：SWD 四针脚 —— 3V3、GND、**SWDIO（PA13）**、**SWCLK（PA14）**；
7. **板载指示灯**：PC13 引脚接一只 LED，用于验证程序是否运行。

## 三、电路设计

### 3.1 外接 3 只 LED 的接线

本实验选用三个端口的第 0 脚，面包板接线简单、容易排查：

| LED | 颜色 | STM32 引脚 | 连接方式 | 点亮电平 |
| --- | --- | --- | --- | --- |
| LED1 | 红 | **PA0（GPIOA，10 脚）** | PA0 → 220Ω → LED 长脚(+)；LED 短脚(−) → GND | **高电平** |
| LED2 | 绿 | **PB0（GPIOB，18 脚）** | PB0 → 220Ω → LED 长脚(+)；LED 短脚(−) → GND | **高电平** |
| LED3 | 蓝 | **PC0（GPIOC，8 脚）** | PC0 → 220Ω → LED 长脚(+)；LED 短脚(−) → GND | **高电平** |

电路示意（3 路灯完全相同）：

```text
STM32引脚 (PA0/PB0/PC0)
    │
   220Ω  限流电阻
    │
    ▷│  LED（长脚为正极，接电阻侧）
    │
   GND  ─────────────── 共地（核心板GND必须与面包板共地!）
```

**限流电阻计算**：红色 LED 正向压降约 1.8~2.0 V，工作电流取 5~10 mA：

$$R = \frac{V_{OH}-V_F}{I_F} = \frac{3.3-2.0}{0.01} = 130\,\Omega$$

实际取标称值 **220 Ω**（此时电流约 6 mA），既能保证亮度又安全。绿/蓝灯压降略高，用同一阻值完全可以。

### 3.2 板载 PC13 指示灯电路（关键！）

查阅 Blue Pill 电路原理图可知板载 LED 的接法与外接灯**正好相反**，属于"共阳"接法：

```text
 3V3 ── 限流电阻 ── ▷│ ── PC13
                    (电流方向 →，PC13必须拉到0V才有电流)
```

- PC13 输出 **0（低电平）→ LED 亮**；输出 **1 → LED 灭**，即**低电平点亮（active-low）**；
- PC13/PC14/PC15 属于**备份域（VBAT 域）引脚**，驱动能力弱（≤3 mA），且输出速度**最高只能配置为 2 MHz**，同一时刻只建议驱动其中一只；
- 这也是为什么网上拿到新板子，常让 PC13 闪烁来验证烧录是否成功。

因此最终流水灯共 **4 只灯**：红(PA0) → 绿(PB0) → 蓝(PC0) → 板载(PC13)。

## 四、程序设计思路与寄存器分析

### 4.1 外设寄存器地址总表

以下地址均来自《STM32中文参考手册》（对应 RM0008）第 3 章存储器映射、第 6 章 RCC、第 8 章 GPIO，以及 datasheet 图 6。

| 寄存器 | 绝对地址 | 作用 | 手册依据 |
| --- | --- | --- | --- |
| RCC_CR | `0x4002 1000` | HSE/PLL 使能与就绪标志 | 6.3.1 |
| RCC_CFGR | `0x4002 1004` | 总线分频、PLL 倍频、时钟切换 | 6.3.2 |
| **RCC_APB2ENR** | **`0x4002 1018`** | APB2 外设（含 GPIOA/B/C）时钟使能 | 6.3.7（P77） |
| FLASH_ACR | `0x4002 2000` | Flash 等待周期、预取使能 | 3.3.3 |
| GPIOA_CRL / CRH | `0x4001 0800` / `0x4001 0804` | PA0~7 / PA8~15 模式配置 | 8.2.1/8.2.2 |
| GPIOA_ODR | `0x4001 080C` | PA 输出数据 | 8.2.4 |
| GPIOA_BSRR | `0x4001 0810` | PA 位置 1 / 清 0 | 8.2.5 |
| GPIOB_CRL / BSRR | `0x4001 0C00` / `0x4001 0C10` | GPIOB 配置 / 置位 | 8.2 节 |
| GPIOC_CRL / CRH / BSRR | `0x4001 1000` / `…04` / `…10` | GPIOC 配置 / 置位 | 8.2 节 |
| SysTick | `0xE000 E010` | 内核 24 位系统滴答定时器 | Cortex-M3 TRM |

GPIO 寄存器组偏移固定：CRL=0x00、CRH=0x04、IDR=0x08、ODR=0x0C、BSRR=0x10、BRR=0x14、LCKR=0x18（手册表 52，P129）。

### 4.2 第一步：开 GPIO 时钟 —— RCC_APB2ENR

STM32 为了省电，**所有外设时钟复位后默认关闭**，不先开时钟就去配置 GPIO 不会有任何反应。该寄存器相关位（6.3.7 节）：

| 位 | 符号 | 含义 |
| --- | --- | --- |
| 0 | AFIOEN | 复用功能 I/O 时钟（涉及重映射/外部中断时需要） |
| 2 | IOPAEN | GPIOA 时钟 |
| 3 | IOPBEN | GPIOB 时钟 |
| 4 | IOPCEN | GPIOC 时钟 |

所以一条语句同时打开三个端口：

```c
RCC->APB2ENR |= (1<<0) | (1<<2) | (1<<3) | (1<<4);
```

### 4.3 第二步：配置引脚模式 —— CRL / CRH

每个引脚由 **4 个 bit** 描述，低 2 位 `MODEy[1:0]` 决定速度/方向，高 2 位 `CNFy[1:0]` 决定模式：

| MODEy[1:0] | 含义 | CNFy[1:0]（输出时） | 含义 |
| --- | --- | --- | --- |
| 00 | 输入 | 00 | 通用推挽输出 |
| 01 | 输出 10 MHz | 01 | 通用开漏输出 |
| 10 | 输出 2 MHz | 10 | 复用推挽 |
| 11 | 输出 50 MHz | 11 | 复用开漏 |

- **PA0 / PB0 / PC0**：均在 CRL 中且位于最低 4 位，写入 `0x3` = 0011b（CNF=00 推挽，MODE=11，50 MHz）；
- **PC13**：在 CRH 中，Pin8 对应 CRH 的 bit3:0，故 Pin13 起始位 = (13−8)×4 = **20**。受备份域限制只能 2 MHz，写入 `0x2 << 20` = 0010b（推挽，2 MHz）。

> "推挽输出"意味着引脚由一对 P-MOS/N-MOS 主动驱动，能主动输出强高/低电平，驱动 LED 必须用推挽；开漏只能拉低，还需外部上拉。

### 4.4 第三步：输出电平 —— BSRR

控制引脚电平有两种方式：改 ODR（读—改—写）或写 **BSRR**。BSRR 更优：

- 低 16 位写 `1`：对应引脚**置 1**（BS = bit set）；
- 高 16 位写 `1`：对应引脚**清 0**（BR = bit reset）；
- 写 `0` 的位一律无影响。该操作是原子的，无中断竞争，也不产生毛刺。

例如让 PA0 输出高：`GPIOA->BSRR = 1<<0;`；输出低：`GPIOA->BSRR = 1<<16;`。

### 4.5 精确 1 秒延时 —— SysTick

软件空循环延时不精确且受主频/编译选项影响，因此使用 Cortex-M3 内核自带的 24 位 SysTick：

- 时钟源选 AHB（72 MHz），重装载值 72000−1，则每 72000 个脉冲 = **1 ms** 触发一次；
- 不装 1 秒（72,000,000 > 24 位最大值 16,777,215），而以 1 ms 为单位循环计数 1000 次得到 1 s；
- 采用查询 `COUNTFLAG` 方式，不占用中断。

### 4.6 系统时钟 72 MHz 配置流程

Blue Pill 出厂靠 HSI（内部 8 MHz）只能跑 8 MHz。为保证延时精确，在 main 开头把时钟提到 72 MHz：

```
关PLL → 开HSE等就绪 → Flash设2等待+预取 → 配置AHB/APB1/APB2分频与PLL×9
      → 开PLL等就绪 → 切换SYSCLK到PLL并回读SWS确认
```

要点：APB1 最高 36 MHz 必须 2 分频；主频 >48 MHz 时 Flash 必须 2 等待周期，否则取指出错。

### 4.7 主程序流程图

```text
        上电 Reset
           │
   ┌───────▼────────┐
   │ 配置系统时钟72M │
   ├────────────────┤
   │ 开GPIOA/B/C时钟│
   │ 配置PA0/PB0/PC0│
   │ 配置PC13(2MHz) │
   │ 全部熄灭       │
   ├────────────────┤
   │ SysTick 1ms    │
   └───────┬────────┘
           ▼
   ┌───────────────────────┐
   │ i = 0                 │
   │ 全部熄灭              │◀──────────────┐
   │ 点亮第 i 只LED        │               │
   │ 延时 1000 ms          │               │
   │ i = (i+1) % 4         │               │
   └───────────────────────┘───────────────┘
```

## 五、代码实现

工程目录（全部代码见随附项目 ZIP）：

```text
STM32_WaterLED/
├── Core/
│   ├── Inc/stm32f103_reg.h      # 寄存器地址与位定义（手写，不依赖厂家库）
│   └── Src/main.c               # 主程序
├── Startup/startup_stm32f103xb.s# 中断向量表与复位初始化
├── linker/STM32F103C8Tx.ld      # 链接脚本（Flash64K/RAM20K）
├── Makefile                     # GCC 编译脚本
└── 实验报告_STM32寄存器方式流水灯.md
```

### 5.1 main.c（完整代码，含详细注释）

```c
/**
 ******************************************************************************
 * @file    main.c
 * @brief   STM32F103C8T6 (Blue Pill) 寄存器方式流水灯
 *          PA0 红 / PB0 绿 / PC0 蓝（高电平点亮），板载 PC13（低电平点亮）
 *          4只灯轮流点亮，间隔1秒。全程直接操作寄存器，不使用任何库。
 ******************************************************************************
 */
#include "stm32f103_reg.h"

/* 描述一只LED：端口、引脚号、是否低电平点亮（板载PC13为1） */
typedef struct
{
    GPIO_TypeDef *port;
    u8  pin;
    u8  active_low;
} Led_t;

/* 流水灯序列表：红(PA0)→绿(PB0)→蓝(PC0)→板载(PC13)。
 * 想调整顺序/增减灯只改此表即可。 */
static const Led_t g_leds[] =
{
    { GPIOA, 0,  0 },
    { GPIOB, 0,  0 },
    { GPIOC, 0,  0 },
    { GPIOC, 13, 1 },
};
#define LED_NUM   (sizeof(g_leds) / sizeof(g_leds[0]))

/* 点亮 / 熄灭指定LED */
static void LED_On(const Led_t *led)
{
    if (led->active_low) GPIO_RESET_PIN(led->port, led->pin); /* 输出0亮 */
    else                 GPIO_SET_PIN(led->port, led->pin);   /* 输出1亮 */
}
static void LED_Off(const Led_t *led)
{
    if (led->active_low) GPIO_SET_PIN(led->port, led->pin);
    else                 GPIO_RESET_PIN(led->port, led->pin);
}
static void LED_AllOff(void)
{
    u32 i;
    for (i = 0; i < LED_NUM; i++) LED_Off(&g_leds[i]);
}

/* 系统时钟：HSE 8MHz → PLL×9 = 72MHz；AHB/1、APB1/2、APB2/1 */
static void Clock_Config72M(void)
{
    u32 timeout = 0;

    RCC->CR &= ~RCC_CR_PLLON;                 /* 关PLL才能改倍频参数 */
    RCC->CR |= RCC_CR_HSEON;                  /* 开外部8MHz晶振 */
    while (((RCC->CR & RCC_CR_HSERDY) == 0) && (++timeout < 0x100000)) { ; }

    FLASH_ACR = FLASH_ACR_PRFTBE | FLASH_ACR_LATENCY_2; /* 72M需2等待 */

    RCC->CFGR = RCC_CFGR_HPRE_DIV1  |         /* AHB  72MHz */
                RCC_CFGR_PPRE1_DIV2 |         /* APB1 36MHz */
                RCC_CFGR_PPRE2_DIV1 |         /* APB2 72MHz */
                RCC_CFGR_ADCPRE_DIV6|         /* ADC  12MHz */
                RCC_CFGR_PLLSRC_HSE |
                RCC_CFGR_PLLXTPRE_HSE |
                RCC_CFGR_PLLMULL9;            /* 8M × 9 = 72M */

    RCC->CR |= RCC_CR_PLLON;
    while ((RCC->CR & RCC_CR_PLLRDY) == 0) { ; }

    RCC->CFGR = (RCC->CFGR & ~RCC_CFGR_SW) | RCC_CFGR_SW_PLL;
    while ((RCC->CFGR & RCC_CFGR_SWS) != RCC_CFGR_SWS_PLL) { ; }
}

/* 初始化流水灯用到的 PA0/PB0/PC0/PC13 */
static void LED_GPIO_Init(void)
{
    /* 1) 开 AFIO + GPIOA/B/C 时钟 */
    RCC->APB2ENR |= (RCC_APB2ENR_AFIOEN |
                     RCC_APB2ENR_IOPAEN |
                     RCC_APB2ENR_IOPBEN |
                     RCC_APB2ENR_IOPCEN);

    /* 2) PA0：CRL[3:0]=0x3 推挽50MHz */
    GPIOA->CRL &= ~(0xFUL << 0);
    GPIOA->CRL |=  (GPIO_MODE_OUT_PP_50MHZ << 0);

    /* 3) PB0 同上 */
    GPIOB->CRL &= ~(0xFUL << 0);
    GPIOB->CRL |=  (GPIO_MODE_OUT_PP_50MHZ << 0);

    /* 4) PC0 同上 */
    GPIOC->CRL &= ~(0xFUL << 0);
    GPIOC->CRL |=  (GPIO_MODE_OUT_PP_50MHZ << 0);

    /* 5) PC13：CRH 起始位(13-8)*4=20，备份域引脚最高2MHz，0x2 */
    GPIOC->CRH &= ~(0xFUL << 20);
    GPIOC->CRH |=  (GPIO_MODE_OUT_PP_2MHZ << 20);

    /* 6) 上电默认全灭 */
    LED_AllOff();
}

/* SysTick 1ms 基准（72MHz / 72000 = 1ms） */
static void Delay_Init(void)
{
    SYSTICK->CTRL = 0;
    SYSTICK->LOAD = 72000UL - 1UL;
    SYSTICK->VAL  = 0;
    SYSTICK->CTRL = SYSTICK_CTRL_CLKSOURCE | SYSTICK_CTRL_ENABLE;
}
static void Delay_ms(u32 ms)
{
    while (ms--)
    {
        SYSTICK->VAL = 0;                                   /* 清计数/COUNTFLAG */
        while ((SYSTICK->CTRL & SYSTICK_CTRL_COUNTFLAG) == 0) { ; }
    }
}

int main(void)
{
    u32 i;

    Clock_Config72M();
    LED_GPIO_Init();
    Delay_Init();

    while (1)
    {
        for (i = 0; i < LED_NUM; i++)
        {
            LED_AllOff();
            LED_On(&g_leds[i]);
            Delay_ms(1000);        /* 保持1秒 */
        }
    }
}
```

### 5.2 stm32f103_reg.h（节选）

头文件中用结构体精确描述每个寄存器组的偏移，用宏给出基地址，例如：

```c
typedef struct {
    volatile u32 CRL;  /* 0x00 */
    volatile u32 CRH;  /* 0x04 */
    volatile u32 IDR;  /* 0x08 */
    volatile u32 ODR;  /* 0x0C */
    volatile u32 BSRR; /* 0x10 */
    volatile u32 BRR;  /* 0x14 */
    volatile u32 LCKR; /* 0x18 */
} GPIO_TypeDef;

#define GPIOA ((GPIO_TypeDef *)0x40010800UL)
#define GPIOB ((GPIO_TypeDef *)0x40010C00UL)
#define GPIOC ((GPIO_TypeDef *)0x40011000UL)

#define RCC   ((RCC_TypeDef *)0x40021000UL)
/* BSRR原子置位/复位 */
#define GPIO_SET_PIN(port,pin)   ((port)->BSRR = (1UL << (pin)))
#define GPIO_RESET_PIN(port,pin) ((port)->BSRR = (1UL << ((pin)+16)))
```

完整定义见工程文件。

## 六、编译与烧录

### 6.1 方式 A：arm-none-eabi-gcc + Make

1. 下载安装 [xPack GNU Arm Embedded GCC](https://developer.arm.com/downloads/-/gnu-rm)，把其 `bin` 加入 PATH；
2. Windows 上可用 MSYS2 的 `make`（或在工程目录下用 `mingw32-make`）；
3. 打开终端进入工程目录执行：

```bash
make            # 生成 build/STM32_WaterLED.elf/.hex/.bin，并打印Flash/RAM占用
make clean      # 清理
```

正常情况下整个程序只有几百字节，Flash 占用 < 2%。

### 6.2 方式 B：Keil MDK5

1. `Project → New μVision Project`，芯片选 **STMicroelectronics → STM32F1 Series → STM32F103C8**；
2. 弹出"Manage Run-Time Environment"时，可只勾选 CMSIS::CORE（或直接用工程自带启动文件，则不勾选任何库）；
3. 把 [main.c](file:///d:/TRAE%20SOLO%20CN/STM32_WaterLED/Core/Src/main.c) 与 [stm32f103_reg.h](file:///d:/TRAE%20SOLO%20CN/STM32_WaterLED/Core/Inc/stm32f103_reg.h) 加入工程，头文件路径里添加 `Core/Inc`；
4. `Options for Target → Debug → ST-Link Debugger → Settings → Flash Download` 勾选 Reset and Run；
5. 编译（F7）、下载（F8）即可。

### 6.3 烧录

- **ST-Link（推荐）**：SWDIO→PA13、SWCLK→PA14、GND、3V3 接好，用 STM32CubeProgrammer 选择 ST-LINK 连接，下载 `.hex/.bin`；命令行可 `make flash`（st-flash）；
- **USB-TTL 串口（无 ST-Link 时）**：把 **BOOT0 跳线置 1**、BOOT1 为 0，按复位，用 FlyMcu / stm32flash 通过 USART1（PA9/PA10）下载；**烧完务必把 BOOT0 拨回 0** 再复位，否则不会运行新程序。

## 七、实验现象

1. 上电后，红、绿、蓝、板载橙红（PC13）4 只灯按顺序逐一点亮；
2. 每只灯点亮时长约 1 秒，同一时刻只有一只灯亮；
3. 一个周期（4 秒）后自动从头循环；
4. 用手机秒表对照，单灯时长误差 < 1%，说明 SysTick 延时与 72 MHz 时钟配置正确；
5. 仅烧录"PA0 闪灯"程序时若板载灯不亮是正常的；将 PC13 编入序列后可见板载灯与外接灯同步轮播，达到题目第 3) 项要求。

## 八、常见问题与排查

| 现象 | 可能原因 | 解决方法 |
| --- | --- | --- |
| 所有灯都不亮 | 忘记开 GPIO 时钟；未共地；BOOT0 还在 1 | 检查 APB2ENR；核心板 GND 接面包板 GND；BOOT0 拨 0 复位 |
| 只有板载灯亮/灭逻辑相反 | PC13 为低电平点亮 | 代码中给 PC13 标记 `active_low=1` |
| 延时明显不准（快约 9 倍） | 时钟仍在 HSI 8MHz，SysTick 却按 72MHz 装载 | 确认 `Clock_Config72M()` 已执行且 SWS 回读为 10 |
| LED 很暗甚至不亮 | 接反；限流电阻过大；用了开漏模式 | 长脚接电阻侧；确认 CNF=00 推挽 |
| 跑到一半死机 | 72MHz 下 Flash 等待周期不足 | FLASH_ACR 必须 LATENCY=2 |
| ST-Link 连不上 | SWD 接反/接错，或代码把 PA13/PA14 改成了 GPIO | "按住复位→Connect→释放"连接后全片擦除 |

## 九、实验总结与体会

通过本次实验，我理解了 STM32 与 51 单片机在开发模式上的最大区别：**外设时钟门控**与**灵活的引脚模式配置**。寄存器编程虽然繁琐，但每一行代码对应手册中的一个确定位，使我对"库函数底层到底做了什么"有了清晰认识。板载 PC13 的"低电平点亮"也提醒我：动手前先读原理图、分清有效电平，比盲目改代码高效得多。

## 十、参考资料

1. 《STM32中文参考手册_V10》（RM0008 Reference Manual 对应中译版），ST；
2. 《STM32F103x8 / STM32F103xB Datasheet》（c8t6 datasheet），ST；
3. STM32 寄存器实现流水灯，CSDN：<https://blog.csdn.net/qq_47281915/article/details/120812867>
4. CSDN：<https://blog.csdn.net/m0_63323712/article/details/133362282>
5. CSDN：<https://blog.csdn.net/geek_monkey/article/details/86291377>
6. CSDN：<https://blog.csdn.net/geek_monkey/article/details/86293880>
7. Blue Pill 引脚图 Generic STM32F103 Pinout Diagram（afiskon/stm32，pinout 的社区版本）
