# STM32F103C8T6 定时器PWM实验报告

> **课程实验**：定时器与PWM实验
> **开发板**：STM32 最小系统核心板 STM32F103C8T6（国际俗称 **Blue Pill**）
> **实验任务**：① 用定时器**计数方式**控制 LED 以 2 s 周期亮-灭；② 用定时器 **PWM 模式**实现呼吸灯（周期 1~2 s），并用 Keil 虚拟示波器观察波形
> **实现方式**：每个任务均给出 **寄存器 / 标准外设库 / HAL 库** 三种实现（同一工程内用宏 `TASK_SEL` 切换任务）

---

## 目录

- [一、实验目的](#一实验目的)
- [二、实验任务与指标](#二实验任务与指标)
- [三、实验平台与器件](#三实验平台与器件)
- [四、硬件设计与接线步骤（含连接图）](#四硬件设计与接线步骤含连接图)
- [五、定时器与 PWM 原理](#五定时器与-pwm-原理)
- [六、任务1 软件实现（定时器计数方式）](#六任务1-软件实现定时器计数方式)
- [七、任务2 软件实现（PWM 呼吸灯）](#七任务2-软件实现pwm-呼吸灯)
- [八、Keil 虚拟示波器（逻辑分析仪）观察波形](#八keil-虚拟示波器逻辑分析仪观察波形)
- [九、编译验证、烧录与提交件清单](#九编译验证烧录与提交件清单)
- [十、三种实现方式对比](#十三种实现方式对比)
- [十一、常见问题与排查](#十一常见问题与排查)
- [十二、实验总结与体会](#十二实验总结与体会)
- [十三、参考资料](#十三参考资料)

---

## 一、实验目的

1. 深入了解 STM32 **通用定时器（TIM2~TIM5）** 的内部结构：预分频器 PSC、计数器 CNT、自动重装载寄存器 ARR、更新事件与更新中断；
2. 掌握 **脉宽调制（PWM）** 的生成方法：PWM 模式1、捕获/比较寄存器 CCR 与占空比的关系、预装载机制；
3. 理解 **GPIO 管脚复用**：定时器通道与 GPIO 引脚（本例 PA0 = TIM2_CH1）的对应关系，复用推挽输出的配置方法；
4. 掌握定时器**更新中断**与 NVIC 的编程：任务1用"计数溢出中断"翻转 LED，任务2用"更新中断"动态调整占空比；
5. 学会用 **Keil 软件仿真 + 逻辑分析仪（Logic Analyzer）** 观察 PWM 输出波形，测量周期与占空比；
6. 通过同一功能在"寄存器 → 标准外设库 → HAL 库"三种方式下的实现，理解三种开发方式的封装层次差异。

## 二、实验任务与指标

### 2.1 任务要求（按任务书）

| 任务 | 要求 | 实现要点 | 验收指标 |
| --- | --- | --- | --- |
| 任务1 | 使用 Tim2~Tim5 某一定时器的某一通道 pin（与 GPIOx 管脚复用）连接一个 LED，用**定时器计数方式**控制 LED 以 **2 s 周期**周期性地亮-灭 | TIM2_CH1（PA0）；TIM2 计数至 1 Hz 溢出，更新中断里翻转 PA0 | 亮 1 s、灭 1 s，完整亮灭周期 2 s |
| 任务2 | 接任务1，采用**定时器 PWM 模式**让 LED 以呼吸灯方式渐亮渐灭，周期 **1~2 s**；用 **Keil 虚拟示波器**观察 PWM 输出波形 | TIM2_CH1 输出 1 kHz PWM；更新中断按正弦表调制占空比 | 呼吸周期 1.5 s；逻辑分析仪可见 1 kHz 方波及占空比渐变 |

### 2.2 本设计的具体参数

| 项目 | 任务1（计数方式） | 任务2（PWM 呼吸灯） |
| --- | --- | --- |
| 定时器 / 通道 | TIM2（不用通道功能，只用计数器） | TIM2 **CH1** |
| 引脚 | **PA0**（TIM2_CH1 复用引脚） | **PA0**（复用推挽输出） |
| PSC 预分频 | 7199（÷7200） | 71（÷72） |
| ARR 自动重装 | 9999 | 999 |
| 计数/载波频率 | 计数 10 kHz，溢出 1 Hz | PWM 载波 **1 kHz** |
| 中断频率 | 1 Hz（每 1 s） | 1 kHz（每 1 ms） |
| 更新节拍 | 中断里翻转 PA0 | 中断内十分频：每 10 ms 更新一次 CCR1 |
| 现象 | 亮 1 s → 灭 1 s，周期 2 s | 占空比按正弦 0→100%→0 循环，呼吸周期 **1.5 s** |

> 两个任务**接线完全相同**（同一个 LED、同一根 PA0 线），只是烧录的程序不同——这正是任务书"接任务1"的含义。

## 三、实验平台与器件

### 3.1 主控芯片参数

| 类别 | 名称 / 型号 | 说明 |
| --- | --- | --- |
| 主控 | STM32F103C8T6 | ARM **Cortex-M3** 内核，LQFP48 封装 |
| 主频 | 最高 72 MHz | 由外部 8 MHz 晶振经 PLL 9 倍频得到 |
| Flash | 64 KB | 起始地址 0x0800 0000（中等容量器件，宏 STM32F10X_MD / STM32F103xB） |
| SRAM | 20 KB | 起始地址 0x2000 0000 |
| 定时器 | **TIM2 / TIM3 / TIM4** 通用定时器 | 均为 16 位，具备 4 个独立通道的输入捕获/输出比较/PWM |
| 总线归属 | TIM2~TIM4 挂 **APB1** | APB1 分频系数≠1 时定时器时钟 = PCLK1 × 2 = 72 MHz |

### 3.2 元器件与工具

| 类别 | 名称 | 数量/说明 |
| --- | --- | --- |
| 元器件 | LED（红/绿/蓝任一色） | 1 只（两只脚：长脚=阳极 +，短脚=阴极 −） |
| 元器件 | 限流电阻 220 Ω | 1 只（0.25 W 色环电阻即可，不分极性） |
| 元器件 | 面包板 | 1 块（400 孔即可） |
| 元器件 | 杜邦线（公对公/公对母） | 3~4 根 |
| 下载器 | ST-Link V2（SWD 四线） | 首选；亦可用 USB-TTL（BOOT0=1 串口 ISP） |
| 开发环境 | **Keil MDK5** + Keil.STM32F1xx_DFP 器件包 | 课程主要使用（含软件仿真 + 逻辑分析仪） |
| 备用工具链 | arm-none-eabi-gcc 14.2.1 | 命令行编译校验（每个工程附 Makefile） |
| 烧录工具 | STM32CubeProgrammer（串口 ISP 时使用） | 备用烧录通道 |

### 3.3 为什么选 PA0 / TIM2_CH1

对照 STM32F103C8T6 数据手册的"引脚定义表（Table 5）"与 Blue Pill 排针图：

- **PA0** 的复用功能之一是 **TIM2_CH1（定时器2 通道1）**；
- 任务1虽然只用"计数器溢出的中断"翻转电平，但任务书要求引脚"与定时器通道复用"，选 PA0 后两个任务可以**共用同一根线**，无需重新接线；
- TIM2 挂在 APB1 上、功能完整（4 通道），是教学中最典型的通用定时器；
- PA0 在 Blue Pill **右侧排针从上往下数第 16 个脚**，接线方便。

常见备选：PA6/PA7（TIM3_CH1/CH2）、PB6~PB9（TIM4_CH1~CH4），代码只需改通道号与 GPIO 配置即可迁移。

## 四、硬件设计与接线步骤（含连接图）

### 4.1 电路原理与限流电阻计算

LED 与 STM32 引脚的连接只有两种形式，本实验采用"高电平点亮"：

```text
STM32 PA0 ──── 220Ω ──── LED长脚(+) ──>│── LED短脚(−) ──── GND（面包板负极轨）── 板子GND
                                    （电流方向：引脚 → 电阻 → LED → 地）
```

限流电阻计算（红色 LED 正向压降约 2.0 V，取工作电流 6 mA）：

```text
R = (3.3 V − 2.0 V) / 6 mA ≈ 217 Ω  →  取标称值 220 Ω
```

> **必须串联限流电阻**。若 LED 直接接在引脚与 GND 之间，电流仅受引脚内阻限制，会把 LED 烧坏、甚至损伤引脚。

### 4.2 接线步骤（按顺序操作，共 6 步）

**准备**：实验板不插电，先接好线再上电。

1. **插 LED**：在面包板上选两个相邻竖列，LED 长脚插入"靠 PA0 一侧"的列，短脚插入另一列。
   注意：面包板每"竖列 5 孔"内部是连通的，LED 两只脚必须插在**不同竖列**，否则被短路。
2. **接电阻**：220 Ω 电阻一端插入 LED 长脚所在列，另一端（另一只脚）插入**向外的一列**（即"PA0 通道"列）。电阻不分极性，随便插。
3. **连 PA0**：杜邦线一端接 Blue Pill **右列第 16 脚（PA0）**，另一端插在电阻外侧那一列。
4. **连 GND**：杜邦线一端接 Blue Pill **左列第 19 脚（GND）**，另一端插在 **LED 短脚所在列**（也即面包板负极轨，如果面包板有红色/蓝色电源轨条，建议把两条都接到蓝轨上再接到 LED，接线更整洁）。
5. **共地检查**：确认"板子 GND ↔ 面包板负极轨 ↔ LED 短脚"这条路径通。共地是必须的——两块电路不共地，LED 不会亮。
6. **上电**：先用 micro-USB 线给核心板供电（板载 D2 电源灯亮）；此时 PA0 默认低电平，LED 应**灭**。程序烧录后开始按任务现象运行。

### 4.3 上电前检查清单

| 检查项 | 正确状态 |
| --- | --- |
| LED 方向 | 长脚（+）朝电阻一侧；短脚（−）朝 GND 一侧 |
| 限流电阻 | 已串入 PA0 与 LED 之间；阻值 220 Ω（红红棕金） |
| 共地 | 板子 GND 与面包板负极轨已连通 |
| 电源 | 用 micro-USB 5 V 供电；不要用 USB-TTL 模块的 3.3 V 引脚给整板供电 |
| 引脚位置 | PA0 = 右列第 16 脚（USB 朝上、从上往下数）；GND = 左列第 19 脚 |

### 4.4 硬件连接图

![图 4-1　实验接线图（左：Blue Pill 排针定位；右：PA0 → 220Ω → LED → 负极轨接线原理；底部为接线要点）](resources/fig_pwm_wiring.png)

### 4.5 硬件常见问题速查

| 现象 | 可能原因 | 处理 |
| --- | --- | --- |
| 上电即亮且不灭 | LED 接错（未串电阻直连 3V3 或短接）| 断电检查 LED 所在竖列与电阻位置 |
| 程序运行但 LED 完全不亮 | ① LED 极性接反 ② 共地断开 ③ 引脚插错位置 | 用万用表二极管挡测 LED；检查 PA0 与 GND 两处杜邦线 |
| 亮度很暗 | 用了高阻值电阻（如 10 kΩ） | 换 220 Ω（或 330 Ω / 1 kΩ 也可临时用，亮度略降） |
| 呼吸灯"一跳一跳"不平滑 | 占空比更新步进太大（如 50 ms 一调）| 保持本项目参数：每 10 ms 调 1 级、共 150 级 |

## 五、定时器与 PWM 原理

### 5.1 通用定时器 TIM2 的时钟与计数过程

TIM2 挂载在 **APB1** 总线上。参考手册的时钟树规定：

- APB1 预分频系数为 1 时，TIM2 时钟 = PCLK1；
- APB1 预分频系数≠1 时，**TIM2 时钟 = PCLK1 × 2**。

本工程 APB1 = 72 MHz/2 = 36 MHz，所以 **TIM2 实际工作时钟 = 72 MHz**（这一点常被忽略，PSC 计算要用 72 MHz 而不是 36 MHz）。

计数过程（向上计数模式）：

```text
TIM2_CLK(72MHz) → [PSC 预分频 ÷(PSC+1)] → CNT 每来一个脉冲加1
     → CNT 加到 ARR 后回 0，同时产生"更新事件(UEV)"
     → 若 DIER 的 UIE=1 且 NVIC 已使能 → 进入更新中断
```

溢出频率公式：

```text
f_overflow = 72 MHz / (PSC+1) / (ARR+1)
```

**任务1**：PSC=7199、ARR=9999 → 72 MHz / 7200 / 10000 = **1 Hz**，即每 1 s 中断一次；中断里翻转 PA0 → LED 亮 1 s、灭 1 s，**完整周期 2 s**。

### 5.2 PWM 模式1 的生成原理

把通道配置为"输出比较 + PWM 模式1"后，输出电平不再由软件控制，而是由硬件比较器实时决定：

```text
PWM 模式1：CNT < CCRx  →  通道输出有效电平（本例高电平）
           CNT ≥ CCRx  →  通道输出无效电平（低电平）
占空比 = CCRx / (ARR+1) = CCRx / 1000（本例）
```

![图 5-1　PWM 模式1 原理：CNT 锯齿计数与 CCR1 比较决定输出电平（示例 CCR1=600，占空比 60%）](resources/fig_pwm_principle.png)

CCR 寄存器的**预装载机制（OC1PE）**：给 CCR1 写入新值时，不会立刻改变占空比，而是在下一次更新事件(UEV)时才整体生效，从而保证每个 PWM 周期都是"完整的方波"，不会产生缺角毛刺。这正是任务2在中断里反复改写 CCR1 的前提。

### 5.3 任务1 预期波形（计数方式：2 s 周期亮-灭）

![图 5-2　任务1 预期波形：更新中断每 1 s 翻转一次 PA0，亮灭完整周期 2 s](resources/fig_pwm_wave_task1.png)

### 5.4 任务2 呼吸灯调制原理

**载波**：PSC=71、ARR=999 → PWM 频率 = 72 MHz/72/1000 = **1 kHz**（远高于人眼闪烁频率，视觉上是连续亮度）。

**调制**：更新中断的频率与载波相同（1 kHz，每 1 ms 一次）。在中断里用一个十分频计数器 `s_div10`，每 10 次中断（= 10 ms）把 CCR1 换成正弦表的下一点：

```text
duty(k) = 500 × (1 − cos(2π·k/150))，k = 0,1,…,149，取值 0~1000
呼吸周期 = 150 点 × 10 ms = 1.5 s
```

占空比以正弦规律 0 → 100% → 0 平滑循环，LED 呈"渐亮渐灭"的呼吸效果；完整周期 1.5 s，满足任务书 1~2 s 要求。

为什么不直接用三角波（线性渐变）？正弦调制在"亮"和"灭"两端的变化率最小、中间最大，人眼观感最接近自然呼吸；三角波在线性递变时两端切换略显突兀。

![图 5-3　任务2 波形：上为占空比正弦调制（周期 1.5 s），下为三个时刻的 PWM 波形细节（25%、50%、75%）](resources/fig_pwm_wave_task2.png)

## 六、任务1 软件实现（定时器计数方式）

### 6.0 工程与代码组织说明

三个工程**结构完全一致**，每份源码中都同时包含任务1与任务2的代码，用文件顶部的宏切换：

```c
/* TASK_SEL = 1 → 任务1：定时器计数方式，LED 以 2 s 周期亮-灭
 * TASK_SEL = 2 → 任务2：PWM 呼吸灯，周期 1.5 s
 * 修改此值后重新编译即可切换任务（命令行编译可用 -DTASK_SEL=x 覆盖） */
#define TASK_SEL 1
```

| 工程目录 | 开发方式 | 工程文件 | 切换宏位置 |
| --- | --- | --- | --- |
| STM32_TimerPWM | 直接操作寄存器 | Project/TimerPWM_Reg.uvprojx | Core/Src/main.c 顶部 |
| STM32_TimerPWM_StdPeriph | 标准外设库 | Project/TimerPWM_StdPeriph.uvprojx | User/tim.h 顶部 |
| STM32_TimerPWM_HAL | HAL 库（CubeMX 风格） | MDK-ARM/TimerPWM_HAL.uvprojx | Core/Inc/main.h 顶部 |

### 6.1 寄存器方式

**核心文件**：`Core/Src/main.c`、`Core/Inc/stm32f103_reg.h`（在流水灯版本基础上新增 TIM2 与 NVIC 寄存器定义）。

先在头文件中补齐寄存器定义（只列新增的 TIM2 部分）：

```c
/*==================== 第15章 通用定时器 TIM2 ====================*/
typedef struct
{
    volatile u32 CR1;    /* 0x00 控制寄存器1 (CEN 使能计数、ARPE 自动重装预装载) */
    volatile u32 CR2;    /* 0x04 */
    volatile u32 SMCR;   /* 0x08 从模式控制 */
    volatile u32 DIER;   /* 0x0C 中断使能 (UIE 更新中断使能) */
    volatile u32 SR;     /* 0x10 状态寄存器 (UIF 更新中断标志) */
    volatile u32 EGR;    /* 0x14 事件产生 (UG 软件产生更新事件) */
    volatile u32 CCMR1;  /* 0x18 捕获/比较模式1 (OC1M、OC1PE) */
    volatile u32 CCMR2;  /* 0x1C */
    volatile u32 CCER;   /* 0x20 捕获/比较使能 (CC1E 通道1输出使能) */
    volatile u32 CNT;    /* 0x24 计数器 */
    volatile u32 PSC;    /* 0x28 预分频器 */
    volatile u32 ARR;    /* 0x2C 自动重装载寄存器 */
    volatile u32 RCR;    /* 0x30 重复计数器(仅TIM1/TIM8) */
    volatile u32 CCR1;   /* 0x34 捕获/比较寄存器1(占空比) */
    volatile u32 CCR2;   /* 0x38 */
    volatile u32 CCR3;   /* 0x3C */
    volatile u32 CCR4;   /* 0x40 */
    volatile u32 BDTR;   /* 0x44 刹车寄存器(仅高级定时器) */
    volatile u32 DCR;    /* 0x48 */
    volatile u32 DMAR;   /* 0x4C */
} TIM_TypeDef;

#define TIM2_BASE       (0x40000000UL)   /* APB1 总线 */
#define TIM2            ((TIM_TypeDef *)TIM2_BASE)

/* 关键位定义 */
#define TIM_CR1_CEN         (1UL << 0)   /* 计数器使能 */
#define TIM_CR1_ARPE        (1UL << 7)   /* 自动重装载预装载使能 */
#define TIM_DIER_UIE        (1UL << 0)   /* 更新中断使能 */
#define TIM_SR_UIF          (1UL << 0)   /* 更新中断标志 */
#define TIM_EGR_UG          (1UL << 0)   /* 软件产生更新事件 */
#define TIM_CCMR1_OC1PE     (1UL << 3)       /* 输出比较1预装载使能 */
#define TIM_CCMR1_OC1M_PWM1 (0x6UL << 4)     /* OC1M=110：PWM模式1 */
#define TIM_CCER_CC1E       (1UL << 0)   /* 通道1输出使能 */

/* RCC_APB1ENR：TIM2 挂 APB1（与 GPIO 的 APB2 不同！） */
#define RCC_APB1ENR_TIM2EN      (1UL << 0)

/* Cortex-M3 NVIC（基地址 0xE000E100） */
#define NVIC_ISER0          (*(volatile u32 *)(0xE000E100UL + 0x000UL))
#define NVIC_IPR_BYTE(irq)  (*(volatile u8  *)(0xE000E100UL + 0x300UL + (u32)(irq)))
#define TIM2_IRQn           (28)   /* RM0008 中断向量表：TIM2 中断号 28 */
```

任务1的核心代码（`Core/Src/main.c`，`TASK_SEL==1` 分支）：

```c
/**
 * @brief 任务1初始化：PA0 推挽输出 + TIM2 计数至 1Hz 更新中断
 */
static void Task1_LED_Timer_Init(void)
{
    /* (1) 开时钟：GPIOA 在 APB2、TIM2 在 APB1 */
    RCC->APB2ENR |= RCC_APB2ENR_IOPAEN;   /* GPIOA 时钟 */
    RCC->APB1ENR |= RCC_APB1ENR_TIM2EN;   /* TIM2 时钟（注意在 APB1！） */

    /* (2) PA0 配置为通用推挽输出，50MHz（本任务引脚作普通输出口用） */
    GPIOA->CRL &= ~(0xFUL << 0);
    GPIOA->CRL |=  (GPIO_MODE_OUT_PP_50MHZ << 0);

    /* (3) 时基：PSC=7199，ARR=9999 → 溢出周期 = 1/1Hz = 1s */
    TIM2->PSC = 7199;
    TIM2->ARR = 9999;
    TIM2->CNT = 0;

    /* (4) 软件产生一次更新事件(UG)，让PSC/ARR立即装载生效；
     *     随即清掉UG造成的UIF标志，避免刚开中断就误进一次 */
    TIM2->EGR = TIM_EGR_UG;
    TIM2->SR  = 0;

    /* (5) 使能更新中断：计数溢出时硬件置UIF并申请中断 */
    TIM2->DIER |= TIM_DIER_UIE;

    /* (6) NVIC 使能 TIM2 中断（中断号28：ISER0 的 bit28 写1） */
    NVIC_IPR_BYTE(TIM2_IRQn) = 0x80;          /* 优先级0x80（数值越小越高） */
    NVIC_ISER0 = (1UL << TIM2_IRQn);

    /* (7) 启动计数器 */
    TIM2->CR1 |= TIM_CR1_CEN;

    /* 上电先点亮LED，便于确认程序已开始运行 */
    GPIO_SET_PIN(GPIOA, 0);
}

/**
 * @brief TIM2 更新中断服务程序：每 1s 进入一次，翻转 PA0 电平
 */
void TIM2_IRQHandler(void)
{
    if (TIM2->SR & TIM_SR_UIF)        /* 检查是否为"更新"事件 */
    {
        TIM2->SR = (u16)~TIM_SR_UIF;  /* 写0清UIF标志（其余位写1无影响） */
        GPIOA->ODR ^= (1UL << 0);     /* 异或翻转 PA0 → LED 亮/灭交替 */
    }
}
```

**关键点解析**：

- 时钟分别在 **APB2ENR（GPIOA）** 和 **APB1ENR（TIM2）** 两处使能，这是初学者最容易漏的地方；
- 写入 PSC/ARR 后先写 `EGR=UG` 产生一次软件更新事件，让新参数立即生效（否则要等第一个完整计数周期）；
- 清标志用 `SR = (u16)~UIF`："写 0 清除"是 TIM 状态寄存器的特殊机制，直接赋 1 到该位反而无效；
- 中断里用 `ODR ^= 1` 翻转电平（该操作只在中断一处执行，不会产生竞争）。

任务1程序流程：

![图 6-1　任务1 程序流程图（左：初始化主流程；右：TIM2 更新中断服务）](resources/fig_pwm_flow_task1.png)

**完整工程还包含**：72 MHz 时钟配置 `Clock_Config72M()`（沿用流水灯工程）、Keil 启动文件所需的 `SystemInit()` 空实现、空转主循环，完整源码见工程 `Core/Src/main.c`。

### 6.2 标准外设库方式

**核心文件**：`User/tim.c`、`User/tim.h`（新增模块）、`User/main.c`、`User/stm32f10x_conf.h`（增加 TIM 模块）。

`stm32f10x_conf.h` 中新增（否则库不编译 TIM 模块）：

```c
#include "stm32f10x_tim.h"        /* 通用定时器：TIM_TimeBaseInit / TIM_OC1Init 等 */
```

任务1核心代码（`User/tim.c`）：

```c
/**
 * @brief  任务1初始化：PA0 推挽输出 + TIM2 计数至 1s 溢出中断
 */
void Tim_LED_Count_Init(void)
{
    GPIO_InitTypeDef        GPIO_InitStructure;
    TIM_TimeBaseInitTypeDef TIM_TimeBaseStructure;
    NVIC_InitTypeDef        NVIC_InitStructure;

    /* (1) 开时钟：GPIOA 在 APB2，TIM2 在 APB1 */
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2,  ENABLE);

    /* (2) PA0 通用推挽输出（本任务作普通 LED 输出口用） */
    GPIO_InitStructure.GPIO_Pin   = GPIO_Pin_0;
    GPIO_InitStructure.GPIO_Mode  = GPIO_Mode_Out_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    /* (3) TIM2 时基：分频 7200；计满 10000 个数溢出一次 = 1s */
    TIM_TimeBaseStructure.TIM_Prescaler     = 7199;      /* PSC=7199 → ÷7200  */
    TIM_TimeBaseStructure.TIM_Period        = 9999;      /* ARR=9999 → 10000个 */
    TIM_TimeBaseStructure.TIM_CounterMode   = TIM_CounterMode_Up;
    TIM_TimeBaseStructure.TIM_ClockDivision = TIM_CKD_DIV1;
    TIM_TimeBaseInit(TIM2, &TIM_TimeBaseStructure);

    /* (4) 清更新标志（TimeBaseInit内部写UG事件会置位），使能更新中断 */
    TIM_ClearFlag(TIM2, TIM_FLAG_Update);
    TIM_ITConfig(TIM2, TIM_IT_Update, ENABLE);

    /* (5) NVIC：使能 TIM2 中断 */
    NVIC_InitStructure.NVIC_IRQChannel                   = TIM2_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 1;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority        = 0;
    NVIC_InitStructure.NVIC_IRQChannelCmd                = ENABLE;
    NVIC_Init(&NVIC_InitStructure);

    /* (6) 启动定时器 */
    TIM_Cmd(TIM2, ENABLE);

    /* 上电先点亮LED，便于确认程序已开始运行 */
    GPIO_SetBits(GPIOA, GPIO_Pin_0);
}

/**
  * @brief  TIM2 更新中断服务程序：每 1s 进入一次，翻转 PA0
  */
void TIM2_IRQHandler(void)
{
    if (TIM_GetITStatus(TIM2, TIM_IT_Update) != RESET)  /* 是否发生更新事件 */
    {
        TIM_ClearITPendingBit(TIM2, TIM_IT_Update);     /* 清中断标志 */
        GPIOA->ODR ^= GPIO_Pin_0;                       /* 翻转 PA0 → 亮/灭交替 */
    }
}
```

`User/main.c`：

```c
int main(void)
{
    /* 中断优先级分组：2位抢占+2位子优先级（NVIC_Init 里配置中断优先级用） */
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);

#if (TASK_SEL == 1)
    Tim_LED_Count_Init();    /* 任务1：定时器计数方式，LED 2s 周期亮-灭 */
#else
    Tim_PWM_Breath_Init();   /* 任务2：PWM 呼吸灯，周期 1.5s */
#endif

    while (1)
    {
        /* 全部工作由定时器硬件 + 中断自动完成，主循环空闲 */
    }
}
```

**与寄存器方式的对应关系**：`TIM_TimeBaseInit` 封装了 PSC/ARR/CNT/EGR 的写入；`TIM_ITConfig` 对应 `DIER |= UIE`；`NVIC_Init` 对应 ISER/IPR 的位操作；中断处理遵循"**取状态 → 清标志 → 干活**"三步曲。库函数版把"位运算"变成了"填结构体参数"，可读性更好，但每一步都能在寄存器中找到对应。

### 6.3 HAL 库方式（CubeMX 风格）

**核心文件**：`Core/Src/main.c`（含 MSP 回调、更新回调）、`Core/Src/stm32f1xx_it.c`（中断转发）、`Core/Src/stm32f1xx_hal_msp.c`（时钟/NVIC）。

HAL 方式最大的结构差异是**回调机制**：中断里只调用 `HAL_TIM_IRQHandler()`，由库判断事件类型后自动调用用户实现的回调函数。

MSP 回调（`stm32f1xx_hal_msp.c`，由 `HAL_TIM_Base_Init()` 自动回调）：

```c
/**
  * @brief  定时器基础功能 MSP（任务1）
  */
void HAL_TIM_Base_MspInit(TIM_HandleTypeDef *htim_base)
{
    if (htim_base->Instance == TIM2)
    {
        __HAL_RCC_TIM2_CLK_ENABLE();               /* TIM2 时钟（APB1 总线） */

        HAL_NVIC_SetPriority(TIM2_IRQn, 1, 0);     /* 抢占1、子优先级0 */
        HAL_NVIC_EnableIRQ(TIM2_IRQn);             /* 使能 TIM2 中断 */
    }
}
```

任务1初始化与回调（`Core/Src/main.c`）：

```c
/**
  * @brief  任务1：TIM2 时基初始化（计数 1Hz 溢出）
  */
static void MX_TIM2_Init(void)
{
    htim2.Instance = TIM2;

    htim2.Init.Prescaler         = 7199;    /* ÷7200 → 10kHz 计数 */
    htim2.Init.CounterMode       = TIM_COUNTERMODE_UP;
    htim2.Init.Period            = 9999;    /* 10000个数 → 1Hz 溢出 */
    htim2.Init.ClockDivision     = TIM_CLOCKDIVISION_DIV1;
    htim2.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_DISABLE;
    if (HAL_TIM_Base_Init(&htim2) != HAL_OK)
    {
        Error_Handler();
    }
}

/**
  * @brief  定时器更新（溢出）事件回调
  * @note   由 HAL_TIM_IRQHandler() 在 TIM2 更新中断里自动调用
  */
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM2)
    {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_0);      /* 翻转 PA0 → 亮/灭交替 */
    }
}
```

中断向量转发（`Core/Src/stm32f1xx_it.c`）：

```c
void TIM2_IRQHandler(void)
{
    HAL_TIM_IRQHandler(&htim2);    /* 库内部：判事件 → 清标志 → 调回调 */
}
```

启动（`main()`）：

```c
    MX_GPIO_Init();          /* PA0 推挽输出（任务1） */
    MX_TIM2_Init();

    HAL_TIM_Base_Start_IT(&htim2);   /* 启动计数并开更新中断 */
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_0, GPIO_PIN_SET);   /* 上电点亮 */
```

**三层封装关系总结**：应用层（main/回调）→ HAL 驱动层（`HAL_TIM_IRQHandler` / `MX_TIM2_Init`）→ MSP 层（时钟、NVIC，板级相关）→ 寄存器。HAL 版把"板子相关"与"逻辑相关"彻底分离，同一套主逻辑换芯片时只需替换 MSP。

## 七、任务2 软件实现（PWM 呼吸灯）

### 7.1 寄存器方式

`Core/Src/main.c` 中 `TASK_SEL==2` 分支的核心代码：

```c
/* 正弦调制表：duty(k) = 500 × (1 − cos(2πk/150))，占空比 0~1000 循环150点 */
static const u16 s_breath_table[150] =
{
       0,    0,    2,    4,    7,   11,   16,   21,   28,   35,
      43,   52,   62,   72,   84,   95,  108,  122,  136,  150,
     165,  181,  198,  215,  232,  250,  268,  287,  306,  326,
     345,  366,  386,  406,  427,  448,  469,  490,  510,  531,
     552,  573,  594,  614,  634,  655,  674,  694,  713,  732,
     750,  768,  785,  802,  819,  835,  850,  864,  878,  892,
     905,  916,  928,  938,  948,  957,  965,  972,  979,  984,
     989,  993,  996,  998, 1000, 1000, 1000,  998,  996,  993,
     989,  984,  979,  972,  965,  957,  948,  938,  928,  916,
     905,  892,  878,  864,  850,  835,  819,  802,  785,  768,
     750,  732,  713,  694,  674,  655,  634,  614,  594,  573,
     552,  531,  510,  490,  469,  448,  427,  406,  386,  366,
     345,  326,  306,  287,  268,  250,  232,  215,  198,  181,
     165,  150,  136,  122,  108,   95,   84,   72,   62,   52,
      43,   35,   28,   21,   16,   11,    7,    4,    2,    0,
};

static volatile u16 s_div10 = 0;   /* 1kHz → 100Hz 十分频计数器 */
static volatile u16 s_idx   = 0;   /* 正弦表下标 0~149 */

/**
 * @brief 任务2初始化：PA0 复用推挽输出 + TIM2_CH1 PWM 1kHz + 更新中断
 */
static void Task2_PWM_Breath_Init(void)
{
    /* (1) 开时钟：GPIOA + TIM2 */
    RCC->APB2ENR |= RCC_APB2ENR_IOPAEN;
    RCC->APB1ENR |= RCC_APB1ENR_TIM2EN;

    /* (2) PA0 配置为"复用推挽输出"：引脚控制权交给 TIM2_CH1
     *     0xB = CNF=10(复用推挽) | MODE=11(50MHz) */
    GPIOA->CRL &= ~(0xFUL << 0);
    GPIOA->CRL |=  (GPIO_MODE_AF_PP_50MHZ << 0);

    /* (3) 时基：PSC=71，ARR=999 → PWM 频率 = 72MHz/(72×1000) = 1kHz */
    TIM2->PSC = 71;
    TIM2->ARR = 999;
    TIM2->CNT = 0;

    /* (4) 通道1配置为 PWM 模式1：
     *     OC1M=110：CNT<CCR1 输出有效电平；
     *     OC1PE=1：CCR1 带预装载——中断里写 CCR1 在下次更新事件生效，无毛刺 */
    TIM2->CCMR1 = TIM_CCMR1_OC1M_PWM1 | TIM_CCMR1_OC1PE;

    /* (5) 通道1输出使能（CC1E=1），极性高电平有效（CC1P=0）→ 占空比越大越亮 */
    TIM2->CCER = TIM_CCER_CC1E;

    /* (6) 让 PSC/ARR/CCR 立即装载，并清标志 */
    TIM2->EGR = TIM_EGR_UG;
    TIM2->SR  = 0;

    /* (7) 开更新中断（1kHz），用于驱动正弦表步进 */
    TIM2->DIER |= TIM_DIER_UIE;
    NVIC_IPR_BYTE(TIM2_IRQn) = 0x80;
    NVIC_ISER0 = (1UL << TIM2_IRQn);

    /* (8) ARPE=1（ARR预装载）+ CEN=1 启动 */
    TIM2->CR1 |= TIM_CR1_ARPE | TIM_CR1_CEN;

    /* 起始占空比 0（LED 灭），由中断逐渐增大 */
    TIM2->CCR1 = 0;
}

/**
 * @brief TIM2 更新中断：每1ms进入一次；每10次(10ms)按正弦表更新一次占空比
 */
void TIM2_IRQHandler(void)
{
    if (TIM2->SR & TIM_SR_UIF)
    {
        TIM2->SR = (u16)~TIM_SR_UIF;

        if (++s_div10 >= 10)                      /* 每10次中断 = 10ms */
        {
            s_div10 = 0;
            TIM2->CCR1 = s_breath_table[s_idx];   /* 查表取新占空比 */
            s_idx = (s_idx + 1) % 150;            /* 循环推进 0~149 */
        }
    }
}
```

**关键点解析**：

- **引脚模式从"通用推挽"改为"复用推挽"**（CRL 低 4 位由 0x3 → 0xB）：这是"GPIO 管脚复用"落到寄存器上的直接体现——复用模式下引脚由片内外设（TIM2_CH1）驱动，软件写 ODR 不再影响引脚；
- `CCMR1` 同时设置 **OC1M=110（PWM 模式1）** 与 **OC1PE=1（CCR 预装载）**；
- 中断在 1 kHz 频率下运行，但每次只做两次自增和一次比较，真正"改占空比"每 10 ms 一次，CPU 开销很小；
- 正弦表存入 Flash（`static const`），150 × 2 B = 300 B。

### 7.2 标准外设库方式

`User/tim.c` 中任务2的核心代码：

```c
/**
  * @brief  任务2初始化：PA0 复用推挽输出 + TIM2_CH1 PWM 模式 1kHz
  */
void Tim_PWM_Breath_Init(void)
{
    GPIO_InitTypeDef        GPIO_InitStructure;
    TIM_TimeBaseInitTypeDef TIM_TimeBaseStructure;
    TIM_OCInitTypeDef       TIM_OCInitStructure;
    NVIC_InitTypeDef        NVIC_InitStructure;

    /* (1) 开时钟 */
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2,  ENABLE);

    /* (2) PA0 复用推挽输出：引脚交给 TIM2_CH1 的外设功能控制 */
    GPIO_InitStructure.GPIO_Pin   = GPIO_Pin_0;
    GPIO_InitStructure.GPIO_Mode  = GPIO_Mode_AF_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    /* (3) TIM2 时基：÷72 后 1MHz 计数，计满 1000 个数 = 1kHz PWM */
    TIM_TimeBaseStructure.TIM_Prescaler     = 71;        /* PSC=71 → ÷72      */
    TIM_TimeBaseStructure.TIM_Period        = 999;       /* ARR=999 → 1000个  */
    TIM_TimeBaseStructure.TIM_CounterMode   = TIM_CounterMode_Up;
    TIM_TimeBaseStructure.TIM_ClockDivision = TIM_CKD_DIV1;
    TIM_TimeBaseInit(TIM2, &TIM_TimeBaseStructure);

    /* (4) 通道1 配置为 PWM 模式1：CNT < CCR1 输出有效电平 */
    TIM_OCInitStructure.TIM_OCMode      = TIM_OCMode_PWM1;
    TIM_OCInitStructure.TIM_OutputState = TIM_OutputState_Enable;
    TIM_OCInitStructure.TIM_OCPolarity  = TIM_OCPolarity_High;  /* 高电平点亮 */
    TIM_OCInitStructure.TIM_Pulse       = 0;                    /* 初始占空比0 */
    TIM_OC1Init(TIM2, &TIM_OCInitStructure);
    TIM_OC1PreloadConfig(TIM2, TIM_OCPreload_Enable);  /* CCR1预装载，更新无毛刺 */

    /* (5) ARR 预装载 */
    TIM_ARRPreloadConfig(TIM2, ENABLE);

    /* (6) 清更新标志，使能更新中断（驱动正弦表步进） */
    TIM_ClearFlag(TIM2, TIM_FLAG_Update);
    TIM_ITConfig(TIM2, TIM_IT_Update, ENABLE);

    /* (7) NVIC 使能 TIM2 中断 */
    NVIC_InitStructure.NVIC_IRQChannel                   = TIM2_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 1;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority        = 0;
    NVIC_InitStructure.NVIC_IRQChannelCmd                = ENABLE;
    NVIC_Init(&NVIC_InitStructure);

    /* (8) 启动定时器 */
    TIM_Cmd(TIM2, ENABLE);
}

/**
  * @brief  TIM2 更新中断：每1ms一次；每10次(10ms)按正弦表更新一次 CCR1
  */
void TIM2_IRQHandler(void)
{
    if (TIM_GetITStatus(TIM2, TIM_IT_Update) != RESET)
    {
        TIM_ClearITPendingBit(TIM2, TIM_IT_Update);

        if (++s_div10 >= 10)                       /* 每10次中断 = 10ms */
        {
            s_div10 = 0;
            TIM_SetCompare1(TIM2, s_breath_table[s_idx]);   /* 写入新占空比 */
            s_idx = (s_idx + 1) % 150;                      /* 循环推进 0~149 */
        }
    }
}
```

库函数与寄存器对照：`TIM_OC1Init` 内部完成 CCMR1/CCER/CCR1 的配置（`TIM_Pulse` 即 CCR1 初值）；`TIM_OC1PreloadConfig(ENABLE)` 置位 OC1PE；`TIM_SetCompare1` 即写 CCR1。正弦表与结构化模块 `tim.c/tim.h` 同 7.1 节，此处不重复。

### 7.3 HAL 库方式

`Core/Src/main.c` 中任务2的初始化与回调：

```c
/**
  * @brief  任务2：TIM2 PWM 初始化（1kHz 载波 + 通道1 PWM模式1）
  */
static void MX_TIM2_Init(void)
{
    TIM_MasterConfigTypeDef sMasterConfig = {0};
    TIM_OC_InitTypeDef sConfigOC = {0};

    htim2.Instance = TIM2;

    htim2.Init.Prescaler         = 71;      /* ÷72 → 1MHz 计数 */
    htim2.Init.CounterMode       = TIM_COUNTERMODE_UP;
    htim2.Init.Period            = 999;     /* 1000个数 → 1kHz PWM */
    htim2.Init.ClockDivision     = TIM_CLOCKDIVISION_DIV1;
    htim2.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_ENABLE;
    if (HAL_TIM_PWM_Init(&htim2) != HAL_OK)
    {
        Error_Handler();
    }

    /* 主从模式：不联级，保持默认 */
    sMasterConfig.MasterOutputTrigger = TIM_TRGO_RESET;
    sMasterConfig.MasterSlaveMode     = TIM_MASTERSLAVEMODE_DISABLE;
    if (HAL_TIMEx_MasterConfigSynchronization(&htim2, &sMasterConfig) != HAL_OK)
    {
        Error_Handler();
    }

    /* 通道1：PWM 模式1，高电平有效，初始占空比 0 */
    sConfigOC.OCMode     = TIM_OCMODE_PWM1;
    sConfigOC.Pulse      = 0;
    sConfigOC.OCPolarity = TIM_OCPOLARITY_HIGH;
    sConfigOC.OCFastMode = TIM_OCFAST_DISABLE;
    if (HAL_TIM_PWM_ConfigChannel(&htim2, &sConfigOC, TIM_CHANNEL_1) != HAL_OK)
    {
        Error_Handler();
    }
}

/* 更新回调：每 1ms 调一次，每 10ms 调一次占空比 */
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM2)
    {
        if (++s_div10 >= 10)
        {
            s_div10 = 0;
            __HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, s_breath_table[s_idx]);
            s_idx = (s_idx + 1) % 150;              /* 循环推进 0~149 */
        }
    }
}
```

启动代码（`main()` 中 `TASK_SEL==2` 分支）：

```c
    HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_1);   /* 启动 PWM 输出（内部等效：CC1E=1 + CEN=1） */
    HAL_TIM_Base_Start_IT(&htim2);              /* 开启更新中断，驱动正弦表 */
```

`__HAL_TIM_SET_COMPARE()` 是宏，展开后就是 `TIM2->CCR1 = 值`；`HAL_TIM_PWM_Start` 内部依次执行"使能通道输出 → 使能计数器"。可见 HAL 层的每个动作都能在寄存器层找到一一对应。

任务2程序流程：

![图 7-1　任务2 程序流程图（左：初始化主流程；右：更新中断中的十分频调制逻辑）](resources/fig_pwm_flow_task2.png)

### 7.4 正弦调制表的生成

表中的 150 个数值由下式离线计算（可用 Python 一行生成，供移植时修改点数/幅值）：

```python
import math
n = 150
vals = [round(500 * (1 - math.cos(2 * math.pi * k / n))) for k in range(n)]
print(vals)
```

若要调整呼吸"柔和度"，可改变：
- 点数 n 与更新间隔（乘积 = 呼吸周期，本例 150 × 10 ms = 1.5 s）；
- 幅值 500（对应 CCR 上限 1000，即 100% 占空比；改为 400 则最亮为 80%，呼吸更柔和）。

## 八、Keil 虚拟示波器（逻辑分析仪）观察波形

任务书要求"使用 Keil 虚拟示波器观察 PWM 输出波形"。Keil MDK 的 **Logic Analyzer（逻辑分析仪）** 在**软件仿真模式**下工作，无需连接实物板即可观察引脚/寄存器的时序波形。完整操作步骤如下。

### 8.1 切换到软件仿真模式

1. 打开工程（以 `TimerPWM_Reg.uvprojx` 为例），先按 **F7 编译**通过；
2. 点击"魔术棒"按钮打开 **Options for Target** → **Debug** 选项卡；
3. 在左上角单选框中选中 **Use Simulator**（软件仿真）；
4. 点击右侧 **Settings**（Simulator 设置）确认：
   - Dialog DLL：`SARMCM3.DLL`
   - Parameter：`-pSTM32F103C8`（若为空请手工填入，否则仿真器不会建模 STM32 外设寄存器）；
5. 一路 OK 确认。此后 Ctrl+F5 进入的将是**纯软件仿真**，不占用真实硬件。

### 8.2 打开逻辑分析仪并添加信号

1. 按 **Ctrl+F5** 进入调试模式（界面出现寄存器/反汇编窗口）；
2. 菜单 **View → Analysis Windows → Logic Analyzer** 打开逻辑分析仪窗口；
3. 点击窗口左上角的 **Setup…**，进入信号配置：
   - 点 **New** 新建一行；
   - 光标落在新增行，在输入框中键入信号名 **`PORTA.0`**（观察 PA0 引脚波形），按回车；
   - 可再 New 两行，分别添加 **`TIM2_CCR1`** 与 **`TIM2_CNT`**（观察占空比数值与计数器，配合 `Peripherals → System Viewer → TIM2` 寄存器窗口一起看）；
   - 关闭 Setup 返回。若 `PORTA.0` 显示为 "unknown signal"，请回查 8.1 中 Parameter 是否为 `-pSTM32F103C8`。
4. 对 `TIM2_CCR1` 一行右键 → **Bit / Analog** 可切换为 "Analog" 显示模式，这样它将以曲线形式直观展示占空比的"正弦爬升"过程。

### 8.3 运行与观察要点

按 **F5（Run）** 开始仿真运行，通过 Zoom In/Out（或按住 Ctrl 滚轮）调整时间刻度：

| 任务 | 观察位置 | 预期波形 | 测量要点 |
| --- | --- | --- | --- |
| 任务1 | PORTA.0 | **方波**：高 1 s / 低 1 s | 刻度调到 1 s/格，两个相邻上升沿间隔 = **2 s** |
| 任务2 | TIM2_CCR1（Analog 模式）| 阶梯状**正弦包络**，谷 0、峰 1000 | 相邻两个波峰间隔 = **1.5 s**（呼吸周期） |
| 任务2 | PORTA.0 | 1 kHz **PWM 方波**，占空比随时间渐变 | 刻度 0.1 ms/格可看清单个周期：高电平宽度 = CCR1 个 1 µs；放大到 100 ms/格整体看"密度渐粗渐细"的呼吸包络 |

**截图建议**（报告中可放实测图）：任务1截"2 s 周期方波"（PORTA.0，1 s/格）；任务2截两幅——"TIM2_CCR1 Analog 正弦包络"与"PORTA.0 局部放大（含 25%/50%/75% 三处占空比）"。

### 8.4 注意事项

- 软件仿真运行要"跑够时间"才能观察到 1.5 s/2 s 级的现象：可在波形窗口观察、耐心等待仿真时间推进；
- 仿真器对外设时钟的建模由 RCC 寄存器配置推导，个别版本可能引入微小时间偏差；**波形规律（周期比例、占空比关系）是正确的**，最终以真机现象与逻辑分析仪截图为准；
- 观察完毕按 **Ctrl+F5** 退出调试，可切回正常编译/下载流程。

## 九、编译验证、烧录与提交件清单

### 9.1 编译验证结果

三个工程均已通过编译验证（命令行 arm-none-eabi-gcc 14.2.1；寄存器/库版 -Os、HAL 版 -O0，与各工程 Makefile 一致），并生成 6 个可直接烧录的 hex 文件：

| 版本 | 任务 | text | data | bss | hex 大小 |
| --- | --- | --- | --- | --- | --- |
| 寄存器 | 任务1 | 548 B | 0 | 1536 B | 1615 B |
| 寄存器 | 任务2 | 904 B | 0 | 1544 B | 2626 B |
| 标准外设库 | 任务1 | 3108 B | 0 | 1536 B | 8815 B |
| 标准外设库 | 任务2 | 4344 B | 0 | 1544 B | 12301 B |
| HAL 库 | 任务1 | 5856 B | 12 B | 1612 B | 16584 B |
| HAL 库 | 任务2 | 8008 B | 12 B | 1620 B | 22643 B |

构建一致性校验（可选做法）：用 `arm-none-eabi-objdump -s -j .isr_vector app.elf` 检查中断向量表，**偏移 0xB0 处（IRQ28 = TIM2）** 应指向 `TIM2_IRQHandler` 的地址 +1（Thumb 位）——本工程验证结果 0x08000139，与符号表一致，说明定时器中断已正确挂上向量表。

> 说明：Keil 编译的绝对尺寸与上表略有出入（优化等级与库版本不同），功能一致。

### 9.2 烧录方法

**方法一（首选）：Keil + ST-Link 一键下载**

1. 用杜邦线连接 ST-Link 与板底 4 针 SWD 口：`3V3→3V3`、`GND→GND`、`SWDIO→SWDIO`、`SWCLK→SWCLK`；
2. Options for Target → Debug → 选 **ST-Link Debugger** → Settings，SW Device 列表中应识别到器件（如 `0x1BA01477`）；
3. 连接正常后按 **F8（Download）** 烧录，复位后观察现象。

**方法二（备用）：串口 ISP（ST-Link 不可用时）**

1. **BOOT0 跳线拨到 1**（BOOT1/PB2 保持 0），按一下复位键进入系统存储器 bootloader；
2. USB-TTL 接线：**TXD→PA9（RX）**、**RXD→PA10（TX）**、**GND↔GND**；**不要接 3.3 V**——板子另用 micro-USB 或 3V3 排针供电（USB-TTL 的 3.3 V 输出带载能力不足）；
3. 打开 **STM32CubeProgrammer**，接口选 **UART**，选择 COM 口，波特率先用 115200（失败可降为 57600/9600）；
4. 打开 `hex/TimerPWM_Taskx_xxx.hex` → Download；
5. 烧录完成后 **BOOT0 拨回 0**，按复位键，程序开始运行。
   也可用命令行：`STM32_Programmer_CLI.exe -c port=COM3 -w TimerPWM_Task1_Reg.hex -v -rst`

### 9.3 提交件清单

| 提交件 | 说明 |
| --- | --- |
| 工程源码 ×3 | `STM32_TimerPWM`（寄存器）、`STM32_TimerPWM_StdPeriph`、`STM32_TimerPWM_HAL`，均为 Keil MDK5 可直接打开的完整工程（含 Makefile 备用） |
| 烧录文件 ×6 | `hex/TimerPWM_Task{1,2}_{Reg,StdPeriph,HAL}.hex` |
| 实验报告 | 本文档 Markdown 源码 + PDF 版 |
| 作业博客 | 博客版文稿（同名 `_博客版.md`）同步发布并提交链接 |
| GitHub | 全部源码与报告上传仓库（建议目录即本包结构） |

GitHub 上传参考命令：

```bash
git init
git add .
git commit -m "STM32F103C8T6 定时器PWM实验：计数方式2s亮灭 + PWM呼吸灯（寄存器/StdPeriph/HAL三版本）"
git branch -M main
git remote add origin https://github.com/<你的用户名>/<仓库名>.git
git push -u origin main
```

## 十、三种实现方式对比

| 对比维度 | 寄存器方式 | 标准外设库 | HAL 库（CubeMX 风格） |
| --- | --- | --- | --- |
| 时钟使能 | `RCC->APB1ENR |= TIM2EN` | `RCC_APB1PeriphClockCmd(...)` | MSP 中 `__HAL_RCC_TIM2_CLK_ENABLE()` |
| 时基配置 | 逐个写 PSC/ARR/CNT/EGR | `TIM_TimeBaseInit(结构体)` | `htim2.Init.* + HAL_TIM_Base_Init()` |
| PWM 通道 | 写 CCMR1/CCER/CCR1 | `TIM_OC1Init + TIM_OC1PreloadConfig` | `HAL_TIM_PWM_ConfigChannel + HAL_TIM_PWM_Start` |
| 中断处理 | 直接写 `TIM2_IRQHandler` | `TIM_GetITStatus/ClearITPendingBit` | `HAL_TIM_IRQHandler` + `HAL_TIM_PeriodElapsedCallback` 回调 |
| 任务1代码体积 | 548 B | 3108 B | 5856 B |
| 可读性 | 低（需对照手册逐位理解） | 中（函数语义清晰） | 高（回调 + 结构体，流程自然） |
| 可移植性 | 差（换芯片要重写） | 中（同系列外设通用） | 好（换芯片仅改 MSP/句柄，主逻辑复用） |
| 学习价值 | 理解硬件本质（本章重点） | 理解外设驱动封装层次 | 理解现代工程化开发方式 |

**结论**：对初学者，寄存器方式最能建立"从手册到代码"的映射；工程实践中用 HAL/CubeMX 效率最高；标准库是两者之间的过渡。本实验通过三种方式实现同一功能，完整覆盖了"抽象逐层升高"的过程。

## 十一、常见问题与排查

| 问题 | 原因排查 | 处理 |
| --- | --- | --- |
| 任务1 LED 不亮且不灭 | 引脚模式或时钟错误；TIM2 时钟漏开（APB1ENR）| 检查 `APB1ENR` 是否置位 TIM2EN；确认 PA0 配置为推挽/复用推挽输出 |
| 亮灭周期不是 2 s | ① 时钟不是 72 MHz（PLL 未配置成功，跑了 8 MHz 内部时钟）② PSC/ARR 算错 | 检查 Clock_Config72M 是否被调用；重新核算 72M/(PSC+1)/(ARR+1) |
| 任务2 LED 常亮或常灭 | PWM 极性反了 / CCR 写错寄存器 / 未开 CC1E | 检查 CCER.CC1E；确认写的是 CCR1；极性 CC1P=0 |
| 呼吸效果"一跳一跳" | 占空比更新步进太大或更新周期 > 30 ms | 保持每 10 ms 调一级；加大点数更平滑 |
| LED 呼吸时闪烁（能看到 1 kHz 载波） | 载波频率太低（如 100 Hz） | 提高 PWM 频率至 1 kHz 以上（本设计 1 kHz） |
| Keil 看不到波形 | ① 未选 Use Simulator ② Parameter 无 -pSTM32F103C8 ③ 没按 F5 运行 ④ 信号名拼错 | 按第八章逐步复查；信号名尝试 `PORTA.0`、`TIM2_CCR1` |
| 编译报"未定义 TIM_..." | 标准库：conf.h 未加 `stm32f10x_tim.h` 或工程未加 `stm32f10x_tim.c`；HAL：`HAL_TIM_MODULE_ENABLED` 被注释 | 按第六章文件清单检查 |
| 串口 ISP 烧录超时 | TX/RX 未交叉；BOOT0 未置 1；3.3V 抢供 | TXD→PA9、RXD→PA10 交叉；BOOT0=1 后复位；拔掉 USB-TTL 的 3.3V |
| 烧录后程序不运行（串口方式）| BOOT0 忘记拨回 0 | BOOT0 拨回 0 再复位 |

## 十二、实验总结与体会

1. **定时器 = 硬件计数器 + 比较器**。本实验两个任务的本质是同一个 TIM2 的两种用法：任务1只用"计数器 + 更新中断"（计数溢出翻转引脚），任务2启用"输出比较 + PWM 模式"（硬件自动生成方波，CPU 只在需要时改 CCR）。把这两者对照起来，通用定时器的结构图（时基单元、比较器、输出控制）就完全理解了。
2. **PSC 与 ARR 决定"节奏"，CCR 决定"亮度"**。任务1的 2 s 周期来自 72 MHz÷7200÷10000=1 Hz 的溢出频率；任务2的 1 kHz 载波来自 72 MHz÷72÷1000，而呼吸的 1.5 s 周期来自"150 点 × 每 10 ms 更新一次"。参数计算是本次实验最需要动手核算的部分（也最容易出错——尤其注意 APB1 分频≠1 时定时器时钟是 PCLK1 的 2 倍）。
3. **中断并不"重"，关键在节流**。任务2让中断以 1 kHz 运行，但真正改写 CCR 的动作每 10 ms 才发生一次；中断服务程序保持极短（两次自增 + 一次查表写寄存器），这是嵌入式中断设计的通用原则。
4. **"GPIO 管脚复用"的一次直观体验**：同一条 PA0 线，任务1里由 CPU 写 ODR 控制，任务2里交给 TIM2_CH1 硬件控制——区别只在于 CRL 的 4 个配置位（0x3 → 0xB，通用推挽 → 复用推挽）。
5. **Keil 软件仿真 + 逻辑分析仪是非常好的"零成本示波器"**：不接硬件就能验证 PWM 频率、占空比、中断周期，非常适合写报告前的自查；对照真机现象则完成了从"波形"到"视觉亮度"的闭环。
6. 通过寄存器/标准库/HAL 三种方式实现同一功能，直观体会到"抽象层次"的意义：**寄存器代码 548 B 但最费脑；HAL 代码 5856 B 但三天后仍然一眼看懂**。课程实验用寄存器打基础，项目开发用 HAL 提效率，这个路线在此次实验中得到了验证。

## 十三、参考资料

1. 《STM32中文参考手册_V10》（对应 RM0008）：第 6 章 RCC、第 8 章 GPIO、第 14/15 章 通用定时器 TIMx；
2. 《STM32F103x8/xB 数据手册》：引脚定义表（PA0/TIM2_CH1 复用）、存储器映射表；
3. 《ARM Cortex-M3 权威指南》：NVIC 中断向量表与优先级；
4. 任务书参考博客（学长经验）：
   - STM32CubeMX & Keil——STM32F103C8T6：PWM控制LED灯：https://blog.csdn.net/Morzart/article/details/134123197
   - https://blog.csdn.net/Xkccsdn147/article/details/134191212
   - https://blog.csdn.net/weixin_52345779/article/details/135313203
5. STM32F10x 标准外设库 v3.5.0 源码与使用手册（UM0427）；
6. Keil MDK 帮助文档：µVision User's Guide — Logic Analyzer 章节。