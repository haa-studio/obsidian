---
title: "ADC ePWM 定时触发与中断采样例程"
created: "2026-08-11"
tags: ["QX", "F280049", "ADC", "ePWM", "中断", "例程", "嵌入式控制"]
---

# ADC ePWM 定时触发与中断采样例程

关联笔记：[[ADC 软件触发例程学习笔记]]、[[ADC 学习路线与底层阅读方法]]、[[F280049 时钟树]]、[[F280049 时钟配置与 Device_init]]、[[F280049 中断架构与 STM32 对比]]。

例程位置：

```text
examples_core0/examples/adc/adc_ex2_soc_epwm
```

## 一句话目标

本例用 ePWM1 做硬件节拍器，定时触发 ADCA 对 A0 采样。每次转换完成后，ADC 中断服务函数把结果存入数组；收满 256 点后，停止采样并在调试断点暂停。

```text
ePWM1 的 TBCTR 计数
-> TBCTR == CMPA 且正在向上计数
-> ePWM1 SOCA
-> ADCA SOC0
-> 采 ADCIN0 / A0
-> ADCARESULT0
-> ADCINT1
-> adcA1ISR()
-> myADC0Results[256]
```

## 先区分三个容易混淆的名字

```text
ADCIN0 / A0          物理模拟输入通道
ADC SOC0             ADC 内部的第 0 个采样转换任务
ePWM SOCA            ePWM 输出给 ADC 的 A 路启动转换触发脉冲
```

连接关系是：

```text
ePWM1 SOCA -> ADCA SOC0 -> ADCIN0(A0) -> ADCARESULT0
```

所以 `EPWM_SOC_A` 不是 `ADC_SOC_NUMBER0`；前者是 ePWM 产生的触发脉冲，后者是 ADC 接到该脉冲后执行的任务。

## 文件分工

| 文件 | 作用 |
| --- | --- |
| `main.c` | 建立主流程，启停 ePWM，等待 256 点采样完成；实现 ADC ISR。 |
| `board.c` | 初始化 ADCA、SOC0、ADCINT1，并注册 ISR。 |
| `board.h` | 为 ADCA、结果寄存器和中断定义工程别名。 |

## 阶段 1：系统、PIE 与 ADC 初始化

`main()` 先执行：

```c
Device_init();
Device_initGPIO();
Interrupt_initModule();
Interrupt_initVectorTable();
Board_init();
```

当前工程的 `Device_init()` 将外部 10 MHz 晶振经 PLL 配为 `SYSCLK = 100 MHz`。它建立系统时钟，但不等于已经完整配置了 ePWM 的 `TBCLK`，时基频率要另见 [[F280049 时钟树]]。

`Board_init()` 中的实际 ADCA 配置为：

```c
ADC_setVREF(ADCA_BASE, ADC_REFERENCE_INTERNAL, ADC_REFERENCE_1_65V);
ADC_setPrescaler(ADCA_BASE, ADC_CLK_DIV_2_0);
ADC_setInterruptPulseMode(ADCA_BASE, ADC_PULSE_END_OF_CONV);
ADC_enableConverter(ADCA_BASE);
DEVICE_DELAY_US(5000);

ADC_setupSOC(ADCA_BASE, ADC_SOC_NUMBER0,
             ADC_TRIGGER_EPWM1_SOCA,
             ADC_CH_ADCIN0, 8U);
```

其意义：

```text
参考电压：内部 1.65 V
被采输入：A0 / ADCIN0
采样任务：ADCA SOC0
触发源：ePWM1 SOCA
采样窗口：8 个 SYSCLK 周期；当前约为 80 ns
中断脉冲：转换真正结束后才产生，故 ISR 可安全读结果
```

这是 12 位 ADC。若输入范围与参考配置匹配，理想换算可近似写成：

```text
Vin ≈ RESULT × 1.65 V / 4095
```

`ADCINT1` 被选择为由 `SOC0` 转换完成置位，并注册到 `adcA1ISR()`：

```c
ADC_setInterruptSource(ADCA_BASE, ADC_INT_NUMBER1, ADC_SOC_NUMBER0);
Interrupt_register(INT_ADCA1, &adcA1ISR);
Interrupt_enable(INT_ADCA1);
```

## 阶段 2：安全配置 ePWM1

在调用 `initEPWM()` 前，程序先执行：

```c
SysCtl_disablePeripheral(SYSCTL_PERIPH_CLK_TBCLKSYNC);
initEPWM();
SysCtl_enablePeripheral(SYSCTL_PERIPH_CLK_TBCLKSYNC);
```

`TBCLKSYNC = 0` 会暂停各 ePWM 的时间基准计数时钟，让程序可以安全设置比较值、周期和触发条件，而不会在“配置到一半”时发生计数或意外 SOCA。它不同于关闭 ePWM1 模块时钟：此时寄存器仍可配置。

`initEPWM()` 的有效配置是：

```c
EPWM_setADCTriggerSource(EPWM1_BASE, EPWM_SOC_A,
                         EPWM_SOC_TBCTR_U_CMPA);
EPWM_setADCTriggerEventPrescale(EPWM1_BASE, EPWM_SOC_A, 1);
EPWM_setCounterCompareValue(EPWM1_BASE, EPWM_COUNTER_COMPARE_A, 1000);
EPWM_setTimeBasePeriod(EPWM1_BASE, 1999);
EPWM_setClockPrescaler(EPWM1_BASE,
                       EPWM_CLOCK_DIVIDER_1,
                       EPWM_HSCLOCK_DIVIDER_1);
EPWM_setTimeBaseCounterMode(EPWM1_BASE, EPWM_COUNTER_MODE_STOP_FREEZE);
```

逐项翻译：

```text
TBCTR 向上数到 CMPA 时，选择为 SOCA 的源。
每遇到 1 次该事件都产生一次 SOCA，不跳过事件。
CMPA = 1000：周期内第 1000 个 TBCLK 附近触发 ADC。
TBPRD = 1999：向上计数周期共有 2000 个 TBCLK（0 到 1999）。
HSPCLKDIV = /1，CLKDIV = /1：TBCLK = EPWMCLK。
初始化结束时冻结 TBCTR：准备好但暂不采样。
```

注意：本例没有配置 ePWM1A/ePWM1B 的引脚复用和输出动作；ePWM1 在这里主要是内部定时触发器，并非要从引脚输出 PWM 波形。

## 阶段 3：开始一次 256 点采样

主循环中：

```c
EPWM_enableADCTrigger(EPWM1_BASE, EPWM_SOC_A);
EPWM_setTimeBaseCounterMode(EPWM1_BASE, EPWM_COUNTER_MODE_UP);
```

从此刻起，单个周期内的硬件时序是：

```text
TBCTR = 0
-> 向上计数
-> TBCTR = 1000：ePWM1 发出 SOCA
-> ADCA SOC0 打开采样保持窗口，采 A0
-> ADC 完成 12 位转换，写入 ADCARESULT0
-> ADCINT1 请求 PIE/CPU 中断
-> 进入 adcA1ISR()
-> TBCTR 继续到 1999，再回到 0
```

在这条链路中，CPU 不负责每次启动转换，因此采样时刻由 ePWM 硬件保持固定。这正是它适合 PWM、电流环和电机控制采样的原因。

## 阶段 4：ISR 保存结果

```c
myADC0Results[index1++] =
    ADC_readResult(myADC0_RESULT_BASE, ADC_SOC_NUMBER0);
```

由于读取的是 SOC0，结果来自 `ADCARESULT0`。数组大小为 256：

```text
第 1 次转换  -> myADC0Results[0]
第 2 次转换  -> myADC0Results[1]
...
第 256 次转换 -> myADC0Results[255]
```

第 256 点保存后，ISR 将 `index1` 回零、置位 `bufferFull`。`bufferFull` 被声明为 `volatile`，因为它由 ISR 修改、由主循环轮询；若不加 `volatile`，编译器可能错误地把主循环优化成永远不重新读取该变量。

ISR 末尾还会：

```text
清 ADCINT1 标志
若检测到中断溢出则清溢出和中断标志
向 PIE 的 ACK Group 1 应答
```

这些操作保证下一次 ADC 转换完成仍可正确进入 ISR。

## 阶段 5：停机、观察、再次采样

主循环等待：

```c
while (bufferFull == 0)
{
}
```

收满 256 点后：

```c
EPWM_disableADCTrigger(EPWM1_BASE, EPWM_SOC_A);
EPWM_setTimeBaseCounterMode(EPWM1_BASE, EPWM_COUNTER_MODE_STOP_FREEZE);
ESTOP0;
```

于是 ADC 不再有新的 SOCA，ePWM 计数也冻结；数组内容稳定，可在调试器 Watch 窗口查看。再次运行后，程序会继续执行并采集下一批数据。

若应用要求“每次开始采样的第一个点都在精确相位”，还应在启动前主动将 `TBCTR` 清零；本例冻结后再次运行可能从上次冻结的计数值继续。

## 采样频率与采样点位置

向上计数且 `SOCA` 每周期一次时：

```text
Fs = TBCLK / (TBPRD + 1)
   = TBCLK / 2000
```

`CMPA` 决定的是周期内的采样相位：

```text
t_sample = CMPA / TBCLK
```

若 `TBCLK = 100 MHz`：

```text
采样周期 = 2000 / 100 MHz = 20 us
采样频率 = 50 kHz
触发时刻 = 1000 / 100 MHz = 周期开始后 10 us
```

但 `SYSCLK = 100 MHz` 不应直接替代 `TBCLK`。实际频率要先确认 `EPWMCLK` 全局分频，再看本例设置的 `/1 × /1` 局部分频；详见 [[F280049 时钟树]]。

## 与软件触发例程的对照

| 项目 | `adc_ex1_soc_software` | `adc_ex2_soc_epwm` |
| --- | --- | --- |
| SOC 触发者 | CPU 软件强制 | ePWM1 SOCA 硬件触发 |
| 等待方式 | 主循环轮询 ADCINT 标志 | ADC 完成后进入 ISR |
| 采样间隔 | 受软件执行与轮询影响 | 由 ePWM 硬件保持固定 |
| 适合场景 | 入门、手动调试 | PWM 同步采样、控制系统 |

## 源码核对时应以实际调用为准

本目录里有几处注释、宏或调用前后不一致，阅读时不要被它们误导：

1. `board.h` 定义了 `myADC0_SAMPLE_WINDOW_SOC0 = 80`，但 `board.c` 实际传给 `ADC_setupSOC()` 的是 `8U`；生效的是 8 个 SYSCLK 周期。
2. 代码先调用 `ADC_disableContinuousMode()`，后又调用 `ADC_enableContinuousMode()`；后者覆盖前者，最终连续模式是开启的，注释“disabled”不准确。
3. 注释说 ADC 上电等待 1 ms，实际调用 `DEVICE_DELAY_US(5000)`，即约 5 ms。
4. 注释按 ePWM 时钟 100 MHz 给出 50 kHz；实际频率仍要以 `EPWMCLK` 的全局分频为准。

## 最小记忆框架

```text
谁定时间？      ePWM1 / TBCTR
谁发触发？      ePWM1 SOCA
谁执行采样？    ADCA SOC0
采哪个引脚？    ADCIN0 / A0
结果在哪里？    ADCARESULT0
谁通知 CPU？    ADCINT1
谁搬运结果？    adcA1ISR()
数据放哪里？    myADC0Results[256]
```

## 建议练习

1. 给 A0 输入稳定电压，观察 256 个结果是否接近恒定。
2. 改变 `CMPA`，理解“采样点在周期内移动”，但采样频率不变。
3. 改变 `TBPRD`，计算并测量采样频率变化。
4. 将结果通过 SCI 输出，画出波形或导入上位机。
5. 之后继续学习 `adc_ex10_multiple_soc_epwm`，扩展到多通道同步采样。
