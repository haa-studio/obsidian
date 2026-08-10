---
title: "ADC 软件触发例程学习笔记"
created: "2026-08-10"
tags: ["QX", "ADC", "driverlib", "例程", "嵌入式控制"]
---
# ADC 软件触发例程学习笔记

关联笔记：[[机器人方向4周学习计划]]、[[CPU Timer 学习笔记]]、[[SCI 串口例程学习笔记]]

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