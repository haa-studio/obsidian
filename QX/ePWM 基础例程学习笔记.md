---
title: "ePWM 基础例程学习笔记"
created: "2026-08-12"
tags: ["QX", "F280049", "ePWM", "PWM", "GPIO", "同步", "Shadow", "例程", "嵌入式控制"]
---

# ePWM 基础例程学习笔记

关联笔记：[[F280049 时钟树]]、[[F280049 中断架构与 STM32 对比]]、[[ADC ePWM 定时触发与中断采样例程]]、[[CPU Timer 学习笔记]]、[[机器人方向4周学习计划]]。

例程目录：

```text
examples_core0/examples/epwm
```

重点入门例程：

```text
epwm_ex2_updown_aq
epwm_ex13_up_aq
epwm_ex3_synchronization
epwm_ex8_deadband
```

## 一句话目标

先把 ePWM 当成一个硬件状态机：时基计数器 `TBCTR` 按固定节拍运行，遇到 `ZERO`、`PRD`、`CMPA`、`CMPB` 等事件时，由 AQ、ET、DB、TZ 等子模块自动产生输出、触发 ADC、产生中断或保护动作。

```text
SYSCLK -> EPWMCLK -> TBCLK -> TBCTR
TBCTR 遇到 ZERO / PRD / CMPA / CMPB
-> AQ 决定 EPWMxA / EPWMxB 输出高低
-> ET 可产生 ePWM 中断或 SOCA/SOCB
-> SYNC/DB/TZ/DC/DMA 是后续扩展功能
```

## 学习顺序

不要按 `ex1` 到 `ex14` 顺序硬啃。`epwm_ex1_trip_zone` 一开始就是保护模块，对入门不友好。

建议顺序：

1. `epwm_ex2_updown_aq`：上下计数、AQ、动态占空比。
2. `epwm_ex13_up_aq`：向上计数，对比上下计数频率公式。
3. `epwm_ex3_synchronization`：多路 ePWM 同步与相移。
4. `epwm_ex8_deadband`：死区、互补输出、反相和输出交换。
5. 回看 `adc_ex2_soc_epwm`：理解 ePWM 作为 ADC 硬件触发源。
6. 最后按需要看 TZ/DC/DMA/Chopper/Global Load 等高级例程。

各例程定位：

| 例程 | 重点 |
| --- | --- |
| `epwm_ex2_updown_aq` | 上下计数 AQ，ISR 动态修改 CMPA/CMPB。 |
| `epwm_ex13_up_aq` | 向上计数 AQ，和上下计数作对比。 |
| `epwm_ex3_synchronization` | EPWM1 做同步源，EPWM2/3/4 按 TBPHS 相移。 |
| `epwm_ex8_deadband` | RED/FED 死区、极性、互补输出、A/B 交换。 |
| `epwm_ex1_trip_zone` | TZ 跳闸保护，GPIO/XBAR 触发强制输出。 |
| `epwm_ex4_digital_compare` | DC 数字比较事件接入 TZ。 |
| `epwm_ex5_digital_compare_event_filter` | DC 事件滤波。 |
| `epwm_ex9_dma` | DMA 自动更新 ePWM 比较值、相位和周期。 |
| `epwm_ex10_chopper` | PC 斩波子模块。 |
| `epwm_ex11_configure_signal` | 用配置结构体生成指定频率/占空比/相位 PWM。 |
| `epwm_ex12_monoshot_mode` | 外部同步触发单脉冲。 |
| `epwm_ex14_global_load_use_case` | 综合例程：同步、死区、保护、全局加载、寄存器链接。 |

## GPIO 与 ePWM 输出引脚

以 `epwm_ex2_updown_aq/board.c` 中 EPWM1 为例：

```c
GPIO_setPinConfig(myEPWM1_EPWMA_PIN_CONFIG);
GPIO_setPadConfig(myEPWM1_EPWMA_GPIO, GPIO_PIN_TYPE_STD);
GPIO_setQualificationMode(myEPWM1_EPWMA_GPIO, GPIO_QUAL_SYNC);

GPIO_setPinConfig(myEPWM1_EPWMB_PIN_CONFIG);
GPIO_setPadConfig(myEPWM1_EPWMB_GPIO, GPIO_PIN_TYPE_STD);
GPIO_setQualificationMode(myEPWM1_EPWMB_GPIO, GPIO_QUAL_SYNC);
```

这不是重复配置同一个引脚，而是分别配置 `EPWM1A` 和 `EPWM1B`：

```text
GPIO0 -> EPWM1A
GPIO1 -> EPWM1B
```

三个函数的分工：

| 调用 | 作用 |
| --- | --- |
| `GPIO_setPinConfig()` | 选择引脚复用功能，让 GPIO 交给 ePWM 模块控制。 |
| `GPIO_setPadConfig(..., GPIO_PIN_TYPE_STD)` | 设置普通标准数字 pad，适合作为推挽输出。 |
| `GPIO_setQualificationMode(..., GPIO_QUAL_SYNC)` | 设置 GPIO 输入采样同步方式；对 ePWM 输出不是核心，但 SysConfig 常模板化生成。 |

`GPIO_PIN_TYPE_STD` 可理解为普通推挽数字输出。PWM 输出高低电平时需要芯片主动拉高、主动拉低，所以使用标准输出是正常的。

`GPIO_QUAL_SYNC` 主要用于输入路径：外部信号进入芯片时同步到系统时钟，减少异步输入直接进入内部逻辑带来的时序风险。对纯 ePWM 输出脚来说影响很小。

## EPWM1A 的基础波形

`epwm_ex2_updown_aq` 中 EPWM1A 的关键配置：

```c
EPWM_setTimeBasePeriod(myEPWM1_BASE, 2000);
EPWM_setTimeBaseCounter(myEPWM1_BASE, 0);
EPWM_setTimeBaseCounterMode(myEPWM1_BASE, EPWM_COUNTER_MODE_UP_DOWN);

EPWM_setCounterCompareValue(myEPWM1_BASE, EPWM_COUNTER_COMPARE_A, 50);
EPWM_setCounterCompareShadowLoadMode(myEPWM1_BASE,
                                     EPWM_COUNTER_COMPARE_A,
                                     EPWM_COMP_LOAD_ON_CNTR_ZERO);

EPWM_setActionQualifierAction(myEPWM1_BASE, EPWM_AQ_OUTPUT_A,
                              EPWM_AQ_OUTPUT_HIGH,
                              EPWM_AQ_OUTPUT_ON_TIMEBASE_UP_CMPA);
EPWM_setActionQualifierAction(myEPWM1_BASE, EPWM_AQ_OUTPUT_A,
                              EPWM_AQ_OUTPUT_LOW,
                              EPWM_AQ_OUTPUT_ON_TIMEBASE_DOWN_CMPA);
```

计数器运行方式：

```text
0 -> 1 -> ... -> 2000 -> 1999 -> ... -> 1 -> 0
```

`CMPA = 50` 表示 `TBCTR` 数到 50 时产生比较事件。AQ 动作规定：

```text
向上数到 CMPA：EPWM1A 置高
向下数到 CMPA：EPWM1A 置低
```

因此 EPWM1A 的波形可以画成：

```text
TBCTR:  0 -> 50 -------------> 2000 -------------> 50 -> 0
EPWM1A: 低    高高高高高高高高高高高高高高高高    低
```

这个例程中 `CMPA` 越小，高电平越宽；`CMPA` 越大，高电平越窄。因为它使用上下计数，并且是在上数 CMPA 置高、下数 CMPA 置低。

上下计数频率近似为：

```text
PWM 频率 = TBCLK / (2 * TBPRD)
```

向上计数频率近似为：

```text
PWM 频率 = TBCLK / (TBPRD + 1)
```

注意：`SYSCLK` 不一定等于 `TBCLK`。实际应按 [[F280049 时钟树]] 中的链路计算：

```text
SYSCLK -> EPWMCLK 全局分频 -> HSPCLKDIV/CLKDIV -> TBCLK
```

## 相位与同步事件

这两行和相位有关：

```c
EPWM_disablePhaseShiftLoad(myEPWM1_BASE);
EPWM_setPhaseShift(myEPWM1_BASE, 0);
```

在 `epwm_ex2_updown_aq` 里，它们表示 EPWM1 不使用相移：

```text
禁止收到同步事件后加载 TBPHS
相位值设为 0
```

同步事件可以理解为 ePWM 之间的对时脉冲：

```text
SYNCO：同步输出
SYNCI：同步输入
```

在 `epwm_ex3_synchronization` 里，EPWM1 做同步源：

```c
EPWM_setSyncOutPulseMode(myEPWM1_BASE,
                         EPWM_SYNC_OUT_PULSE_ON_COUNTER_ZERO);
```

含义：

```text
EPWM1 的 TBCTR == 0 时，发出同步事件
```

EPWM2/3/4 收到同步事件后，如果启用了 phase load，就把自己的 `TBCTR` 加载为 `TBPHS`：

```text
EPWM2 TBPHS = 300 -> 300 * 360 / 2000 = 54 度
EPWM3 TBPHS = 600 -> 600 * 360 / 2000 = 108 度
EPWM4 TBPHS = 900 -> 900 * 360 / 2000 = 162 度
```

示意图：

```text
时间向右 ->

EPWM1 TBCTR:
0 -------- 2000 | 0 -------- 2000 | 0 -------- 2000
^                ^                ^
同步事件          同步事件          同步事件

EPWM2 收到同步事件后：
TBCTR = TBPHS = 300

EPWM2 TBCTR:
300 ---- 2000 | 0 -------- 2000 | 0 -------- ...
^
收到 EPWM1 同步事件
```

波形上就是相对错开：

```text
EPWM1A: ___████____████____████
EPWM2A: _____████____████____██
```

同步事件的典型用途：

1. 三相电机控制，让三相 PWM 保持固定相位关系。
2. 多相交错电源，例如两相差 180 度、三相差 120 度，降低纹波。
3. PWM 触发 ADC，让采样点固定在 PWM 周期中特定位置。
4. 多路 PWM 参数同步更新，避免 A/B/C 三相在不同瞬间生效。

## Shadow 影子寄存器

`shadow` 是影子寄存器，也可以理解为缓冲区。它解决的问题是：PWM 正在高速输出，如果 CPU 在周期中间直接改 `CMPA/CMPB/TBPRD/死区值`，当前周期可能被半路改坏，产生毛刺或不一致。

有 shadow 后，写入路径变成：

```text
CPU 写新值 -> Shadow 寄存器
到指定安全时刻 -> 自动加载到 Active 寄存器
Active 值才真正影响 PWM 输出
```

以 `CMPA` 为例：

```c
EPWM_setCounterCompareValue(myEPWM1_BASE, EPWM_COUNTER_COMPARE_A, 50);
EPWM_setCounterCompareShadowLoadMode(myEPWM1_BASE,
                                     EPWM_COUNTER_COMPARE_A,
                                     EPWM_COMP_LOAD_ON_CNTR_ZERO);
```

含义：

```text
CPU 修改 CMPA 时，先进入 shadow
等 TBCTR == 0 时，再加载到 active CMPA
```

没有 shadow 的风险：

```text
TBCTR:   0 ---- 50 ---- 1000 ---- 1500 ---- 2000 ---- 1500 ---- 0
PWM:     ____██████████████████████████████____
CPU改CMPA=1500       ^
当前周期中途就可能改变边沿判断
```

有 shadow 后：

```text
当前周期：
TBCTR:   0 ---- 50 ---- 1000 ---- 2000 ---- 50 ---- 0
PWM:     ____██████████████████████████████____
CPU改CMPA=1500       ^
只是写入 shadow，不影响当前周期

下一周期：
TBCTR:   0 ------------ 1500 ---- 2000 ---- 1500 ---- 0
PWM:     ________________██████________________
新 CMPA 才生效
```

`EPWM_COMP_LOAD_ON_CNTR_ZERO` 表示在 `TBCTR == 0` 这个周期边界加载，适合让每个 PWM 周期开始时统一更新占空比。

## `epwm_ex2_updown_aq`：board.c 与 main.c 的职责

这个例程把内容分在两处；它们不是重复做同一件事。

```text
board.c：把初始参数和 AQ 规则写入 ePWM 硬件寄存器
main.c ：保存运行时状态；在中断中决定 CMPA/CMPB 下一步如何变化
```

`board.c` 做的是“硬件从什么状态开始工作”。例如：

```c
EPWM_setTimeBasePeriod(myEPWM1_BASE, 2000);
EPWM_setCounterCompareValue(myEPWM1_BASE, EPWM_COUNTER_COMPARE_A, 50);
EPWM_setCounterCompareValue(myEPWM1_BASE, EPWM_COUNTER_COMPARE_B, 1950);
```

它还规定比较事件来临时输出执行什么动作：

```text
EPWM1A：上数到 CMPA 时置高；下数到 CMPA 时置低
EPWM1B：上数到 CMPB 时置高；下数到 CMPB 时置低
```

`main.c` 中的 `EPWMx_MIN_CMPA/B`、`EPWMx_MAX_CMPA/B` 则不是额外写一次初值，而是软件更新算法的边界。例如：

```c
#define EPWM1_MAX_CMPA 1950U
#define EPWM1_MIN_CMPA   50U
```

它们决定 `CMPA` 扫到哪里折返；`CMPB` 同理。`main.c` 里的 `EPWMx_TIMER_TBPRD` 宏目前没有被使用，真正写入 `TBPRD=2000` 的是 `board.c`。这属于示例迁移/生成代码留下的重复定义，应避免两处参数日后改得不一致。

## `updateCompare()`：CMPA/CMPB 的往返扫描

函数位置：`epwm_ex2_updown_aq/main.c`，`updateCompare(epwmInformation *epwmInfo)`。

它**不修改 `TBPRD` 或时钟分频，所以不改变 PWM 频率**。它只移动 `CMPA/CMPB` 这两个比较点；AQ 根据比较点的位置，在相应时刻自动改变引脚电平。因此它造成的是占空比/边沿位置的缓慢变化。

可以把 `CMPA`、`CMPB` 理解为两个在区间内往返移动的滑块：

```text
CMPA: 50 -> 51 -> ... -> 1950 -> 1949 -> ... -> 50 -> ...
CMPB: 1950 -> 1949 -> ... -> 50 -> 51 -> ... -> 1950 -> ...
```

### ePWM1 的初始软件状态

`board.c` 写入初值，`initEPWM1()` 写入方向状态：

```text
硬件比较寄存器：CMPA = 50，CMPB = 1950
软件状态：      CMPA 方向 = UP，CMPB 方向 = DOWN，软件计数 = 0
扫描边界：      CMPA/CMPB 均为 50 到 1950
```

函数先读取当前值：

```c
compAValue = EPWM_getCounterCompareValue(..., EPWM_COUNTER_COMPARE_A);
compBValue = EPWM_getCounterCompareValue(..., EPWM_COUNTER_COMPARE_B);
```

接着看软件计数器是否为 10：

```c
if(epwmInfo->epwmTimerIntCount == 10U)
```

若不是 10，只执行 `epwmTimerIntCount++`，两个比较值都不变。初值为 0，因此第 1 到第 10 次 ISR 都只是计数；第 11 次 ISR 才更新比较值：

| ISR 次数 | 进入时软件计数 | 是否更新 CMP | 退出时软件计数 | CMPA | CMPB |
| ---: | ---: | --- | ---: | ---: | ---: |
| 1 | 0 | 否 | 1 | 50 | 1950 |
| 2 | 1 | 否 | 2 | 50 | 1950 |
| 10 | 9 | 否 | 10 | 50 | 1950 |
| 11 | 10 | 是 | 0 | 51 | 1949 |

所以源码注释“每 10 次中断更新”并不严格：按现有 `== 10` 写法，是每 **11 次 ISR** 更新。又因为 `board.c` 配置“每 3 个 PWM 周期触发一次 ePWM 中断”，所以一个 CMP 步进相隔 `11 × 3 = 33` 个 PWM 周期。

### 第 11 次 ISR：逐行看状态变化

进入函数时：

```text
timerIntCount = 10
CMPA = 50，方向 = UP
CMPB = 1950，方向 = DOWN
```

先清零软件计数器：

```c
epwmInfo->epwmTimerIntCount = 0U;
```

然后更新 CMPA。方向是 `UP` 且 `50 < 1950`，故执行 `++compAValue`：

```text
CMPA：50 -> 51；方向仍为 UP
```

更新 CMPB。方向是 `DOWN` 且 `1950 != 50`，故执行 `--compBValue`：

```text
CMPB：1950 -> 1949；方向仍为 DOWN
```

结束状态：

```text
timerIntCount = 0
CMPA = 51，方向 = UP
CMPB = 1949，方向 = DOWN
```

下一次真正更新时变为 `CMPA=52、CMPB=1948`。每次写入的是 shadow 比较值，配置为在下一个 `TBCTR=0` 装载到 active 寄存器，所以不会在一个 PWM 周期中途突然改变输出。

### 到边界时如何折返

假设 CMPA 已经走到最大值：

```text
更新前：CMPA = 1950，方向 = UP
```

由于不再满足 `compAValue < epwmMaxCompA`，程序不会写 `1951`，而是：

```c
epwmInfo->epwmCompADirection = EPWM_CMP_DOWN;
--compAValue;  // 1950 -> 1949
```

因此更新后是：

```text
CMPA = 1949，方向 = DOWN
```

在最小值也一样：`CMPA=50、方向=DOWN` 的下一次更新会变成 `CMPA=51、方向=UP`。CMPB 使用完全相同的逻辑，只是它有自己的方向和边界。

### 初始 ePWM1 波形与 CMP 的关系

初始 `CMPA=50`、`CMPB=1950` 时，一个完整的上下计数周期为：

```text
TBCTR:    0 -> 50 -------------> 1950 -> 2000 -> 1950 -------------> 50 -> 0
EPWM1A:   低    [上数 CMPA：置高]                         [下数 CMPA：置低] 低
EPWM1B:   低                         [上数 CMPB：置高] [下数 CMPB：置低]    低
```

用简化波形表示：

```text
时间  ---------------------------------------------------------------------->
TBCTR       /\
           /  \
          /    \
         0      2000      0

EPWM1A  ___|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾|___
           U-CMPA=50                              D-CMPA=50

EPWM1B  ______________________________________|‾‾|________________________
                                           U-CMPB=1950 D-CMPB=1950
```

因此：

```text
CMPA 小（如 50）   -> EPWM1A 很早置高、很晚置低 -> 高电平很宽
CMPA 大（如 1950） -> EPWM1A 很晚置高、很早置低 -> 高电平很窄

CMPB 小            -> EPWM1B 高电平很宽
CMPB 大            -> EPWM1B 高电平很窄
```

对 ePWM1 而言，`updateCompare()` 让 A 的高电平逐渐变窄、B 的高电平逐渐变宽；走到边界后再反向。该例程没有启用死区模块，因此 ePWM1A/B 不应直接视为可安全驱动半桥的一对互补 PWM。

## 死区 RED/FED 的 shadow

`board.c` 里还有：

```c
EPWM_setRisingEdgeDelayCountShadowLoadMode(myEPWM1_BASE,
                                           EPWM_RED_LOAD_ON_CNTR_ZERO);
EPWM_disableRisingEdgeDelayCountShadowLoadMode(myEPWM1_BASE);

EPWM_setFallingEdgeDelayCountShadowLoadMode(myEPWM1_BASE,
                                            EPWM_FED_LOAD_ON_CNTR_ZERO);
EPWM_disableFallingEdgeDelayCountShadowLoadMode(myEPWM1_BASE);
```

`RED` 是 Rising Edge Delay，上升沿延迟。`FED` 是 Falling Edge Delay，下降沿延迟。它们属于死区模块。

但在 `epwm_ex2_updown_aq` 里，EPWM1 主要演示 AQ 动态占空比，没有真正使用死区；所以这些死区 shadow 配置基本是 SysConfig 模板化生成的默认配置。真正学习死区应看 `epwm_ex8_deadband`。

## 最小记忆框架

```text
TBPRD：周期
TBCTR：正在跑的计数器
CMPA/CMPB：比较点，也就是边沿位置
AQ：遇到事件后输出高、低、翻转或不变
Shadow：先存新值，到安全时刻再生效
SYNC：多路 ePWM 对时
TBPHS：收到同步事件后装载到 TBCTR 的相位值
DB RED/FED：上升沿/下降沿死区延迟
TZ/DC：保护与数字比较
ET：ePWM 中断、ADC SOCA/SOCB 触发
```

## 建议练习

1. 只观察 `EPWM1A/GPIO0`，确认 `CMPA` 改变时占空比变化。
2. 把 `CMPA` 从 50 改成 500、1000、1500，画出高电平宽度变化。
3. 对比 `epwm_ex2_updown_aq` 和 `epwm_ex13_up_aq`，确认上下计数与向上计数频率不同。
4. 看 `epwm_ex3_synchronization`，只关注 `SYNCO/SYNCI/TBPHS` 三个词。
5. 看 `epwm_ex8_deadband`，用 EPWM1 作为参考，对比 EPWM2/3/4/5/6 的死区输出。

## 一句话总结

学 ePWM 时不要先背所有子模块名。先抓住这条主线：

```text
TBCTR 计数 -> 遇到 CMPA/CMPB/ZERO/PRD -> AQ 改变输出 -> Shadow 保证修改在安全时刻生效 -> SYNC/DB/TZ/ADC 触发在这条主线上扩展
```
