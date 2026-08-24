---
title: "ADC 软件触发例程学习笔记"
created: "2026-08-10"
tags: ["QX", "ADC", "driverlib", "例程", "嵌入式控制"]
---
# ADC 软件触发例程学习笔记

关联笔记：[[机器人方向4周学习计划]]、[[CPU Timer 学习笔记]]、[[SCI 串口例程学习笔记]]、[[ADC 学习路线与底层阅读方法]]、[[ADC ePWM 定时触发与中断采样例程]]

例程位置：

```text
f280049revb_evb_examples-master/f280049revb_evb_examples-master/examples_core0/examples/adc/adc_ex1_soc_software
```

本笔记整理今天围绕 `adc_ex1_soc_software` 例程提出的问题。这个例程的核心不是“ADC 中断服务函数”，而是：软件触发一组 ADC SOC，轮询等待转换完成标志，再从结果寄存器读取采样值。

## 这个例程学什么

`adc_ex1_soc_software` 是 ADC 入门例程，重点是建立 ADC 采样的基本模型：

```text
配置 ADC 模块
-> 配置 SOC 采样任务
-> 软件强制触发 SOC
-> 等待转换完成标志
-> 读取结果寄存器
```

这个例程还没有进入电机控制常用的固定周期采样。后面要看 `adc_ex2_soc_epwm`，那里才是：

```text
ePWM 定时触发 ADC -> ADC 中断里读结果
```

这条路线对应 [[机器人方向4周学习计划]] 第 2 周的目标：ADC + EPWM。

与下一例程的关键差别是“谁发起 SOC”：

```text
adc_ex1_soc_software：CPU 执行 ADC_forceMultipleSOC()，再轮询 ADCINT 标志。
adc_ex2_soc_epwm：ePWM1 定时产生 SOCA，ADC 完成后真正进入 CPU ISR。
```

`ex1` 适合理解 SOC、通道、RESULT 和完成标志；`ex2` 才是控制系统中更常用的“PWM 定时采样”闭环。完整时序见 [[ADC ePWM 定时触发与中断采样例程]]。

## myADC0_init 做了什么

`board.c` 里的 `myADC0_init()` 初始化的是 ADCA。名字 `myADC0` 只是工程生成出来的抽象名，在 `board.h` 里对应：

```c
#define myADC0_BASE        ADCA_BASE
#define myADC0_RESULT_BASE ADCARESULT_BASE
```

所以可以直接理解为：

```text
myADC0 = ADCA
myADC0_RESULT_BASE = ADCA 的结果寄存器基地址
```

`myADC0_init()` 分成几步：

1. 打开 ADCA 外设时钟。
2. 设置 ADC 参考电压、ADC 时钟分频、转换完成脉冲时机。
3. 使能 ADC 转换核心，并延时等待 ADC 稳定。
4. 配置 SOC0 采 A0。
5. 配置 SOC1 采 A1。
6. 配置 ADCINT1 由 SOC1 完成置位。

关键代码：

```c
ADC_setVREF(myADC0_BASE, ADC_REFERENCE_EXTERNAL, ADC_REFERENCE_3_3V);
ADC_setPrescaler(myADC0_BASE, ADC_CLK_DIV_2_0);
ADC_setInterruptPulseMode(myADC0_BASE, ADC_PULSE_END_OF_CONV);
ADC_enableConverter(myADC0_BASE);
DEVICE_DELAY_US(5000);
```

这段是 ADC 模块本体初始化。

SOC 配置是核心：

```c
ADC_setupSOC(myADC0_BASE, ADC_SOC_NUMBER0,
             ADC_TRIGGER_SW_ONLY,
             ADC_CH_ADCIN0,
             8U);

ADC_setupSOC(myADC0_BASE, ADC_SOC_NUMBER1,
             ADC_TRIGGER_SW_ONLY,
             ADC_CH_ADCIN1,
             8U);
```

含义是：

```text
ADCA SOC0：软件触发，采 ADCIN0，也就是 A0
ADCA SOC1：软件触发，采 ADCIN1，也就是 A1
```

一句话记住：SOC 是一次“采样转换任务”，不是采样结果。

## SOC、通道、结果寄存器的关系

今天最容易混的是两个编号：

```text
ADC 输入通道编号：A0、A1、C2、C3
ADC SOC 编号：SOC0、SOC1、SOC2...
ADC 结果寄存器编号：RESULT0、RESULT1、RESULT2...
```

在这个例程中，结果按 SOC 编号保存，不按输入通道编号保存。

对应关系是：

```text
ADCA SOC0 采 A0 -> ADCARESULT0
ADCA SOC1 采 A1 -> ADCARESULT1

ADCC SOC0 采 C2 -> ADCCRESULT0
ADCC SOC1 采 C3 -> ADCCRESULT1
```

所以不是：

```text
C2 -> ADCCRESULT2
C3 -> ADCCRESULT3
```

而是：

```text
C2 被 ADCC 的 SOC0 采，所以结果在 ADCCRESULT0
C3 被 ADCC 的 SOC1 采，所以结果在 ADCCRESULT1
```

记忆规则：

```text
ADC_CH_ADCINx 决定采哪个引脚。
ADC_SOC_NUMBERx 决定结果放到 RESULTx。
```

主函数里的读取也验证了这一点：

```c
myADC0Result0 = ADC_readResult(myADC0_RESULT_BASE, ADC_SOC_NUMBER0);
myADC0Result1 = ADC_readResult(myADC0_RESULT_BASE, ADC_SOC_NUMBER1);
myADC1Result0 = ADC_readResult(myADC1_RESULT_BASE, ADC_SOC_NUMBER0);
myADC1Result1 = ADC_readResult(myADC1_RESULT_BASE, ADC_SOC_NUMBER1);
```

## 为什么配置 ADC interrupt，却没有 ISR

这个例程没有真正使用 CPU 中断服务函数。它配置 ADC interrupt 的目的，是把 ADCINT1 当作“转换完成标志”来轮询。

`board.c` 中：

```c
ADC_setInterruptSource(myADC0_BASE, ADC_INT_NUMBER1, ADC_SOC_NUMBER1);
ADC_clearInterruptStatus(myADC0_BASE, ADC_INT_NUMBER1);
ADC_disableContinuousMode(myADC0_BASE, ADC_INT_NUMBER1);
ADC_enableInterrupt(myADC0_BASE, ADC_INT_NUMBER1);
```

含义是：

```text
当 SOC1 转换完成后，置位 ADCINT1 标志。
```

主函数里这样等待：

```c
while (ADC_getInterruptStatus(myADC0_BASE, ADC_INT_NUMBER1) == false)
{ }
ADC_clearInterruptStatus(myADC0_BASE, ADC_INT_NUMBER1);
```

所以这里的 ADCINT1 没有接到真正的 CPU ISR。它只是一个可查询的完成标志。

这个例程没有做：

```c
Interrupt_register(..., adcISR);
Interrupt_enable(...);
```

因此 CPU 不会跳到中断服务函数。

ADCINT 可以有三种用法：

```text
1. 当完成标志，主循环轮询。
2. 接到 PIE/CPU，真正进入 ISR。
3. 触发后续 SOC，形成 ADC 自触发链。
```

本例用的是第 1 种。

## 为什么用 SOC1 作为完成标志来源

ADCA 一次触发两个 SOC：

```c
ADC_forceMultipleSOC(myADC0_BASE, ADC_FORCE_SOC0 | ADC_FORCE_SOC1);
```

SOC0 采 A0，SOC1 采 A1。SOC1 是这一组里的最后一个任务，所以让 SOC1 完成后置位 ADCINT1，基本就代表这一组 ADCA 转换完成。

流程是：

```text
软件触发 SOC0/SOC1
-> ADC 转换 A0/A1
-> SOC1 完成
-> ADCINT1 标志置位
-> 主循环轮询发现完成
-> 清标志
-> 读取 RESULT0/RESULT1
```

## SOCPRIORITY 两句配置是否冲突

今天发现了一个很重要的问题：

```c
AdcaRegs.ADCSOCPRICTL.bit.SOCPRIORITY = 2;
```

和：

```c
ADC_setSOCPriority(myADC0_BASE, ADC_PRI_ALL_ROUND_ROBIN);
```

它们配置的是同一个寄存器字段。前一句写 `2`，后一句通过 driverlib 写 `0`。

在 `adc.h` 中：

```c
ADC_PRI_ALL_ROUND_ROBIN  = 0U
ADC_PRI_SOC0_HIPRI       = 1U
ADC_PRI_THRU_SOC1_HIPRI  = 2U
```

所以：

```text
SOCPRIORITY = 2 表示 SOC0-SOC1 高优先级。
ADC_PRI_ALL_ROUND_ROBIN = 0 表示所有 SOC 轮询。
```

而 `ADC_setSOCPriority()` 的实现会写同一个字段，因此后写的生效。这个例程最终实际是：

```text
所有 SOC 都是 round-robin。
```

第 85 行更像是模板残留或代码生成器留下的直接寄存器写法。对这个简单例程影响不大，因为只用 SOC0/SOC1，而且是软件一次性触发。

工程习惯上更推荐统一使用 driverlib。如果想让 SOC0/SOC1 高优先级，应写成：

```c
ADC_setSOCPriority(myADC0_BASE, ADC_PRI_THRU_SOC1_HIPRI);
```

不要同时直接写寄存器又调用 driverlib 配同一字段，否则容易出现覆盖。

## 看这个例程的正确顺序

不要从所有寄存器位开始硬啃。推荐按这个顺序：

1. 先看 `board.h`，确认 `myADC0`、`myADC1` 分别对应哪个 ADC 模块。
2. 再看 `myADC0_init()`，只抓 ADCA 配了几个 SOC、每个 SOC 采哪个通道、由谁触发。
3. 再看 `myADC1_init()`，它和 ADCA 类似，只是采 ADCC 的 C2/C3。
4. 回到 `main.c`，看 `ADC_forceMultipleSOC()` 如何启动转换。
5. 看 `ADC_getInterruptStatus()` 如何等待完成。
6. 看 `ADC_readResult()` 如何读取结果。
7. 最后再追 driverlib 或手册里的寄存器字段。

最小闭环是：

```text
ADC_setupSOC 配任务
ADC_forceMultipleSOC 启动任务
ADC_getInterruptStatus 等任务完成
ADC_readResult 读任务结果
```

## 推荐练习

### 练习 1：只采 ADCA A0

先只保留 ADCA SOC0，观察 `myADC0Result0`。目标是把 A0 电压变化和数字量变化对应起来。

12 位 ADC，3.3V 参考下大致关系：

```text
数字值 = 输入电压 / 3.3V * 4095
电压 = 数字值 / 4095 * 3.3V
```

例如：

```text
0V    -> 约 0
1.65V -> 约 2048
3.3V  -> 约 4095
```

### 练习 2：把结果通过 SCI 打印

参考 [[SCI 串口例程学习笔记]]，把 ADC 结果打印成：

```text
adc = 2048
voltage = 1.65V
```

这一步会把 ADC 和串口调试接起来。

### 练习 3：再看 ePWM 触发 ADC

看完软件触发后，再看：

```text
examples_core0/examples/adc/adc_ex2_soc_epwm
```

重点比较：

```text
adc_ex1：ADC_TRIGGER_SW_ONLY
adc_ex2：ADC_TRIGGER_EPWM1_SOCA
```

这就是从“软件手动采样”进入“固定周期控制采样”的关键一步。

## 一句话总结

`adc_ex1_soc_software` 真正教的是 ADC 的基本工作链：

```text
SOC 定义采样任务，软件触发任务，ADCINT 标志表示任务完成，RESULTx 保存 SOCx 的转换结果。
```

读懂这条链，再去看 ePWM 触发 ADC，就不会被寄存器名字绕晕。

## 补充：差分模式下的通道含义

`adc_ex1_soc_software_differential` 只配置 `ADC_CH_ADCIN0`，却能测到 A0 和 A1，是因为它把 ADC 配置成了差分模式：

```c
AdcaRegs.ADCCTL1.bit.SELVI_HD_LS = 1;
AdccRegs.ADCCTL1.bit.SELVI_HD_LS = 1;
```

差分模式下，通道编号代表一对输入，而不是一个独立引脚：

```text
通道 0 -> ADCIN0(+) - ADCIN1(-)
通道 2 -> ADCIN2(+) - ADCIN3(-)
通道 4 -> ADCIN4(+) - ADCIN5(-)
```

因此：

```text
ADCA SOC0 + 通道 0 -> A0 - A1 -> ADCARESULT0
ADCC SOC0 + 通道 2 -> C2 - C3 -> ADCCRESULT0
```

一次差分转换涉及两个物理引脚，但只产生一个差值结果；不是为两个引脚分别生成两个结果寄存器。

还要区分三个概念：

```text
SOC -> RESULT：一对一，SOCx 的结果进入 RESULTx
SOC -> 通道：多对一，多个 SOC 可以重复采同一个通道
通道 -> 物理输入：单端时是一个引脚，差分时是一对引脚
```

例如单端模式下，SOC0 和 SOC1 都可以配置为 ADCIN0，它们分别进入 RESULT0 和 RESULT1。

## API 深入：ADC_setInterruptPulseMode

函数原型位于 `adc.h`：

```c
static inline void ADC_setInterruptPulseMode(uint32_t base,
                                             ADC_PulseMode pulseMode);
```

它决定一次 ADC 转换完成脉冲（EOC Pulse）何时产生。这个脉冲可以置位 ADCINT，也可以作为后续触发链的一部分。

常用模式有两个：

```text
ADC_PULSE_END_OF_CONV
    转换阶段结束后产生脉冲；结果已经准备好，ISR 中可直接读取。

ADC_PULSE_END_OF_ACQ_WIN
    采样窗口结束、逐次逼近转换刚开始时就产生脉冲；延迟更低，但此时结果可能还未完成。
```

底层实现修改的是 `ADCCTL1.INTPULSEPOS` 字段：先清字段，再写入枚举值，并用 `EALLOW/EDIS` 保护寄存器。

本例使用：

```c
ADC_setInterruptPulseMode(myADC0_BASE, ADC_PULSE_END_OF_CONV);
```

软件触发、低速轮询例程应优先使用转换结束模式。只有在高速控制环路中明确需要降低中断延迟时，才研究提前脉冲，并结合 `ADC_setInterruptCycleOffset()` 验证转换完成时序。

## API 深入：ADC_disableBurstMode

函数原型位于 `adc.h`：

```c
static inline void ADC_disableBurstMode(uint32_t base);
```

它清除 `ADCBURSTCTL.BURSTEN`，让 ADC 回到普通 SOC 触发模式。

```text
普通模式：每个 SOC 使用自己在 ADC_setupSOC() 中配置的触发源
突发模式：一个触发源启动一串连续 SOC，由 ADC_setBurstModeConfig() 配置数量
```

本例显式关闭突发模式，是为了保证软件触发例程的行为确定：SOC0、SOC1 只会按照各自的配置响应软件触发。过采样、多通道连续采集和高速采样调度才需要进一步学习 `ADC_enableBurstMode()` 与 `ADC_setBurstModeConfig()`。

## 本例的底层阅读边界

本例建议先看 API 和数据流，再只对照以下寄存器字段：

```text
ADCCTL1.INTPULSEPOS       中断脉冲位置
ADCSOCxCTL.CHSEL          SOC 选择的输入通道
ADCSOCxCTL.TRIGSEL        SOC 触发源
ADCSOCxCTL.ACQPS          采样窗口
ADCINTSEL1N2.INT1SEL      哪个 SOC 置位 ADCINT1
ADCRESULTx                SOCx 的转换结果
ADCBURSTCTL.BURSTEN       是否启用突发模式
```

不要在同一字段上同时使用直接寄存器写法和 driverlib API；后执行的写入会覆盖先执行的配置。

## 深入补充：ADC 完整中断链，不能只记“配置中断源”

一句最重要的话：

> `ADC_setInterruptSource()` 只完成“某个 SOC 的 EOC 接到 ADCINT”的选择；它不是 ADC 进入 CPU ISR 的全部配置。

完整硬件链路应从触发开始看：

```text
ePWM SOCA / 软件强制 / CPU Timer 等触发源
    -> SOCx 变为待转换
    -> ADC 采样保持（S+H）
    -> ADC 转换
    -> ADCRESULTx 写入结果，产生 EOCx
    -> ADCINTn 选择 EOCx 作为来源
    -> ADCINTFLG.bit.ADCINTn 置位
    -> CPU 轮询该标志，或 ADCINTn 送入 PIE
    -> PIE 向量进入 CPU ISR
```

### 1. SOC 配置：决定“什么时候采哪个模拟脚”

```c
ADC_setupSOC(ADCA_BASE, ADC_SOC_NUMBER1,
             ADC_TRIGGER_EPWM1_SOCA,
             ADC_CH_ADCIN2, acqps);
```

`ADC_setupSOC()` 对应 `ADCSOC1CTL` 的三个核心字段：

| 字段 | 回答的问题 |
| --- | --- |
| `TRIGSEL` | 谁启动 SOC1？例如 ePWM SOCA、CPU Timer、GPIO 或软件。 |
| `CHSEL` | SOC1 采哪个 ADCIN 模拟引脚？ |
| `ACQPS` | 采样保持窗口有多长？源阻抗越大，通常越需要较长窗口。 |

这里的 SOC 是一张“采样任务单”，不是已经完成的一次采样。触发信号到来后，SOC 才排队执行采样和转换。

### 2. EOC：SOC 转换完成的硬件脉冲

每个 SOC 都有对应的完成信号。例如 SOC1 完成时产生 `EOC1`。`EOC1` 表示这一次 SOC1 的转换结束，结果已写入 `ADCRESULT1`。

`ADC_setInterruptPulseMode()` 决定 EOC/ADCINT 脉冲出现的时刻：

```c
ADC_setInterruptPulseMode(ADCA_BASE, ADC_PULSE_END_OF_CONV);
```

本例使用转换结束模式。含义是 ADC 真正完成数字转换后才发 EOC，因此 CPU 看到完成标志时可以读取结果。早期中断模式可缩短 ISR 响应延迟，但 ISR 刚进入时结果可能尚未准备好。

### 3. EOC 到 ADCINT：这是“转换完成通知”的选择

```c
ADC_setInterruptSource(ADCA_BASE,
                       ADC_INT_NUMBER1,
                       ADC_SOC_NUMBER1);
```

这句配置 `ADCINTSEL1N2.INT1SEL = EOC1`：

```text
ADCA SOC1 完成 -> EOC1 -> ADCINT1
```

若一批触发同时启动 SOC0、SOC1、SOC2，通常应选择最后一个 SOC 的 EOC 作为 `ADCINT1` 来源：

```text
SOC0 完成：可能还有后续结果未准备好
SOC1 完成：可能还有后续结果未准备好
SOC2 完成：这一批的结果都已准备好 -> 适合作为 ADCINT1 来源
```

### 4. ADCINT 到 CPU ISR：还需要两层使能

先打开 ADC 模块内部的 ADCINT：

```c
ADC_enableInterrupt(ADCA_BASE, ADC_INT_NUMBER1); // ADCINTSEL1N2.INT1E = 1
```

再把它注册并使能到 PIE/CPU：

```c
Interrupt_register(INT_ADCA1, &adca1ISR);
Interrupt_enable(INT_ADCA1);
EINT;
ERTM;
```

只有上述链路全部连上，`EOC1` 才会让 CPU 跳进 `adca1ISR()`。

`ADCINT1` 本身有三种常见用途：

| 用途 | 是否需要注册 CPU ISR | 说明 |
| --- | --- | --- |
| 轮询完成标志 | 否 | 读取 `ADCINTFLG.ADCINT1`，适合软件触发和低速调试。 |
| 进入 CPU ISR | 是 | 使用 `INT_ADCA1` 或 `INT_ADCC1` 等 PIE 向量。 |
| 触发下一个 SOC | 否 | 用 `ADCINTSOCSELx` 建立 ADCINT 到后续 SOC 的硬件转换链。 |

`ADCINTSOCSEL1/2` 容易误解：它配置的是“ADCINT 反过来触发哪个 SOC”，不是“ADCINT 是否进入 CPU”。

### 5. 标志必须清除

若关闭连续模式：

```c
ADC_disableContinuousMode(ADCA_BASE, ADC_INT_NUMBER1);
```

则上一次 `ADCINT1` 标志未清除前，下一次 EOC1 不会正常产生新的 ADCINT1 脉冲。因此轮询代码或 ISR 都必须执行：

```c
ADC_clearInterruptStatus(ADCA_BASE, ADC_INT_NUMBER1);
```

真实 ISR 还应检查 `ADCINTOVF`。它说明新的 EOC 到来时，旧的 ADCINT 标志仍未被清除，控制循环可能已来不及处理上一批结果。

```c
if(ADC_getInterruptOverflowStatus(ADCA_BASE, ADC_INT_NUMBER1))
{
    ADC_clearInterruptOverflowStatus(ADCA_BASE, ADC_INT_NUMBER1);
    ADC_clearInterruptStatus(ADCA_BASE, ADC_INT_NUMBER1);
}
```

## 深入补充：PPB 后处理模块

关联例程：`adc_ex7_ppb_offset`。

PPB（Post-Processing Block，后处理模块）不是启动 ADC 的模块。SOC 已经完成转换之后，PPB 才对绑定 SOC 的数字结果做硬件处理。每个 ADC 模块有 PPB1 到 PPB4。

```text
模拟电压
    -> SOCx 采样和转换
    -> ADC 原始数字码 D_raw
    -> PPB 的 OFFCAL 校准
    -> ADCRESULTx = D_cal
    -> PPB 的 OFFREF 参考相减
    -> ADCPPBxRESULT = D_err（有符号）
    -> 高限/低限/过零检测、事件、ePWM 保护或事件中断
```

### 1. 先将 PPB 绑定到一个 SOC

```c
ADC_setupPPB(ADCA_BASE, ADC_PPB_NUMBER1, ADC_SOC_NUMBER1);
```

含义：`ADCA PPB1` 只处理 `ADCA SOC1` 的转换结果。一个 PPB 同时只能绑定一个 SOC；多个 PPB 可以绑定同一个 SOC。

注意：多个 PPB 指向同一个 SOC 时，真正影响 `ADCRESULTx` 的 `OFFCAL` 是编号最高的 PPB 的 `OFFCAL`。这也是所有 PPB 默认关联 SOC0 时容易误配置的原因。

### 2. OFFCAL：校准固定硬件偏移，结果会写入 ADCRESULTx

配置：

```c
ADC_setPPBCalibrationOffset(ADCA_BASE, ADC_PPB_NUMBER1, offcal);
```

硬件公式：

```text
D_cal = saturate(D_raw - OFFCAL, 0, 4095)
ADCRESULTx = D_cal
```

`OFFCAL` 是 10 位有符号校准值。ADC 为 12 位时，校准后的 `ADCRESULTx` 会饱和：低于 0 写 0，高于 4095 写 4095。

现实电流采样例子：零电流时理想 ADC 码应为 2048，但测得 2078。

```text
D_raw = 2078
期望零点 = 2048
OFFCAL = 2078 - 2048 = +30

D_cal = 2078 - 30 = 2048
```

若零电流实际测得 2028：

```text
OFFCAL = 2028 - 2048 = -20
D_cal = 2028 - (-20) = 2048
```

因此只需记住：硬件固定执行“原始结果减 OFFCAL”。`OFFCAL` 用于消除传感器、运放、PCB 和 ADC 通道共同造成的固定零点误差。

### 3. OFFREF：计算“实际值相对于参考值”的有符号误差

配置：

```c
ADC_setPPBReferenceOffset(ADCA_BASE, ADC_PPB_NUMBER1, offref);
ADC_disablePPBTwosComplement(ADCA_BASE, ADC_PPB_NUMBER1);
```

硬件公式：

```text
D_err = D_cal - OFFREF
ADCPPBxRESULT = D_err
```

`ADCPPBxRESULT` 是符号扩展的 32 位结果，不饱和。因此它适合直接表示双向电流或控制误差：

| 物理情况 | 原始值 | OFFCAL 校正后 | `OFFREF = 2048` 时 PPBRESULT |
| --- | ---: | ---: | ---: |
| 零电流 | 2078 | 2048 | 0 |
| 正电流 | 2250 | 2220 | +172 |
| 负电流 | 1900 | 1870 | -178 |

若使能二进制补码：

```c
ADC_enablePPBTwosComplement(ADCA_BASE, ADC_PPB_NUMBER1);
```

则方向反过来：

```text
ADCPPBxRESULT = OFFREF - D_cal
```

即从“实际值 - 参考值”变为“参考值 - 实际值”。是否反相必须由控制算法的误差符号定义决定。

### 4. PPB 高限、低限、过零事件

PPB 可以针对处理后的 `ADCPPBxRESULT` 自动判断：

```text
高于 TRIPHI -> 高限事件
低于 TRIPLO -> 低限事件
达到/跨越零条件 -> 零事件
```

事件标志在 `ADCEVTSTAT` 中，软件用 `ADCEVTCLR` 清除；也可以配置在下一次 PPB 结果装载时自动清除。事件有两条独立输出路径：

```text
PPB 事件 -> X-Bar / ePWM Trip Zone：用于硬件过流关 PWM
PPB 事件 -> ADCA_EVT / ADCC_EVT：用于 CPU 记录故障
```

要让事件送往 ePWM/X-Bar，使用：

```c
ADC_enablePPBEvent(ADCA_BASE, ADC_PPB_NUMBER1,
                   ADC_EVT_TRIPHI | ADC_EVT_TRIPLO);
```

要让事件送往 CPU 的 ADC 事件中断，使用：

```c
ADC_enablePPBEventInterrupt(ADCA_BASE, ADC_PPB_NUMBER1,
                            ADC_EVT_TRIPHI | ADC_EVT_TRIPLO);
```

同一个 ADC 模块的所有 PPB 共享一个 `ADCA_EVT` 或 `ADCC_EVT` 入口。进入 ISR 后必须读取 `ADCEVTSTAT` 或 `ADC_getPPBEventStatus()`，判断是哪个 PPB、哪个事件触发。

### 5. PPB 采样延迟时间戳

`ADCPPBxSTAMP.DLYSTAMP` 记录关联 SOC 从“触发到来”到“实际开始采样”之间等待的 SYSCLK 周期数。

它用于发现多个 SOC/多个控制环同时请求同一 ADC 时的排队延迟。高速电机控制中，采样时刻晚于预定位置会引入测量误差。注意：手册说明，软件直接触发与 PPB 关联的 SOC 时，这项延迟捕捉功能不工作。

### 6. `adc_ex7_ppb_offset` 的真实数据流

| ADC | SOC0 | SOC1 | PPB1 配置 | 观察变量 |
| --- | --- | --- | --- | --- |
| ADCA（A2） | 原样采 A2 | 采 A2，EOC1 置 ADCINT1 | `OFFCAL=+100`，`OFFREF=0` | `myADC0Result=ADCRESULT0`；`myADC0PPBResult=ADCPPB1RESULT` |
| ADCC（C2） | 原样采 C2 | 采 C2，EOC1 置 ADCINT1 | `OFFCAL=-100`，`OFFREF=0` | `myADC1Result=ADCRESULT0`；`myADC1PPBResult=ADCPPB1RESULT` |

若 A2、C2 的 SOC1 原始码都为 2500：

```text
ADCA：D_cal = 2500 - (+100) = 2400，PPBRESULT = 2400
ADCC：D_cal = 2500 - (-100) = 2600，PPBRESULT = 2600
```

本例中 `myADC0Result` 来自 SOC0，而 `myADC0PPBResult` 来自 SOC1 的 PPB1；二者是连续两次独立采样，只在输入电压稳定时才会近似相差 100。另一个注意点：`ADC_readPPBResult()` 返回有符号 32 位值。若未来使用 `OFFREF` 产生负误差，接收 PPB 结果的变量应使用 `int32_t`，不要继续使用 `uint16_t`。

### 7. 本例没有启用 PPB 事件或 CPU ISR

`adc_ex7_ppb_offset` 仅用 ADCINT1 作为主循环轮询的“转换完成标志”。它没有 `Interrupt_register(INT_ADCA1, ...)`，因此不会进入 ADCA ISR。它也关闭了：

```c
ADC_disablePPBEvent(...);
ADC_disablePPBEventInterrupt(...);
```

所以本例的目标仅仅是验证：硬件可以自动对 ADCA 结果减 100、对 ADCC 结果加 100；它不演示 PPB 越限保护和 PPB 事件中断。
