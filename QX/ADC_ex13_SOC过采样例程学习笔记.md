---
type: experiment-note
topic: ADC
status: learning
updated: 2026-08-20
tags:
  - QX049
  - ADC
  - SOC
  - ePWM
  - 过采样
  - 噪声分析
  - 电机控制
---

# `adc_ex13_soc_oversampling` 例程学习笔记

## 1. 例程要解决什么问题

这个例程演示的是：

> 用多个 SOC 对同一个 ADC 通道连续采样，再用软件求平均，降低随机噪声对结果的影响。

它不是 ADC 硬件自动平均，也不是数字滤波器，而是：

```text
同一个 ePWM 触发
    ↓
SOC2、SOC3、SOC4、SOC5 都采 A2
    ↓
得到 4 个 ADC 原始结果
    ↓
软件相加后除以 4
    ↓
得到 A2 的过采样结果
```

本例同时保留两个普通通道：

```text
SOC0 -> A0：普通采样
SOC1 -> A1：普通采样
SOC2 -> A2：过采样第 1 次
SOC3 -> A2：过采样第 2 次
SOC4 -> A2：过采样第 3 次
SOC5 -> A2：过采样第 4 次
```

对应源码目录：

`C:\Users\OSS\Desktop\f280049revb_evb_examples-master\examples_core0\examples\adc\adc_ex13_soc_oversampling`

## 2. 和 `adc_ex10_multiple_soc_epwm` 的区别

| 例程 | SOC 配置 | 主要目的 |
|---|---|---|
| `adc_ex10_multiple_soc_epwm` | SOC0/1/2 采三个不同通道 | 多通道同步采样 |
| `adc_ex13_soc_oversampling` | SOC0/1 采 A0/A1，SOC2~5 都采 A2 | 对 A2 进行 4 倍过采样 |

两者的共同点是：

```text
ePWM1 SOCA -> 多个 SOC -> 最后一个 SOC 完成 -> ADCINT1 -> ISR
```

`ex10` 的 ADCINT1 源是 SOC2，`ex13` 的 ADCINT1 源是 SOC5：

```c
ADC_setInterruptSource(myADC0_BASE,
                       ADC_INT_NUMBER1,
                       ADC_SOC_NUMBER5);
```

这是因为 `SOC5` 是这一批转换的最后一个 SOC。等 SOC5 完成时，SOC0~SOC4 的结果也已经产生，可以一次性读取整批数据。

## 3. `board.c` 的配置逻辑

### 3.1 ADC 时钟和采样窗口

```c
ADC_setPrescaler(myADC0_BASE, ADC_CLK_DIV_2_0);
```

本工程 `SYSCLK` 配置为约 100 MHz，因此 ADC 内核时钟约为：

```text
ADCCLK = SYSCLK / 2 = 50 MHz
```

每个 SOC 使用：

```c
ADC_SAMPLE_WINDOW_10
```

在当前 SDK 的 `adc.h` 中：

```c
ADC_SAMPLE_WINDOW_10 = 8U
```

这里的 `8U` 是写入 `ACQPS` 寄存器的编码值，硬件将该编码解释为 10 个采样窗口周期。因此 `board.c` 中复制过来的“8 SYSCLK cycles”注释是旧注释，不能按字面理解。这个参数的本质仍是 ADC 采样保持窗口，实际时序要结合芯片手册的 `ACQPS` 编码和 SYSCLK 周期确认。

学习时应同时记录：

1. 宏名对应的实际采样窗口周期数；
2. `ACQPS` 使用的时钟单位；
3. 输入信号源阻抗是否足够低，采样电容是否能够充电稳定。

采样窗口过短，可能带来输入建立不足和增益误差；窗口加长会增加一批 SOC 的总耗时，降低可用采样率。

### 3.2 所有 SOC 使用同一个 ePWM 触发源

```c
ADC_setupSOC(myADC0_BASE,
             ADC_SOC_NUMBER2,
             ADC_TRIGGER_EPWM1_SOCA,
             ADC_CH_ADCIN2,
             ADC_SAMPLE_WINDOW_10);
```

SOC2~SOC5 的触发源全部是 `EPWM1 SOCA`，所以每次 ePWM SOCA 到来时，四个 SOC 都会排队采样 A2。

注意：这不是四个 ADC 同时采样，而是同一个 ADC 内核依次完成四个 SOC。四次采样之间存在时间间隔，因此它更准确地表示“短时间窗口内的重复采样平均”。

### 3.3 SOC 优先级

```c
ADC_setSOCPriority(myADC0_BASE, ADC_PRI_ALL_ROUND_ROBIN);
```

表示 SOC 采用轮询方式服务。由于本例 SOC0~SOC5 都由同一个 SOCA 触发，ADC 会按硬件仲裁规则依次处理这些转换。

### 3.4 ADCINT1 的作用

```c
ADC_setInterruptSource(myADC0_BASE,
                       ADC_INT_NUMBER1,
                       ADC_SOC_NUMBER5);
ADC_enableInterrupt(myADC0_BASE, ADC_INT_NUMBER1);
```

含义是：

```text
SOC5 EOC -> ADCINT1 标志 -> ADCA1 ISR
```

`ADC_INT_NUMBER1` 是 ADC 内部中断线 1，`ADC_SOC_NUMBER5` 是第 6 个 SOC 槽位，两个编号属于不同体系。

## 4. `main.c` 的 ePWM 触发时序

例程中：

```c
EPWM_setADCTriggerSource(EPWM1_BASE,
                         EPWM_SOC_A,
                         EPWM_SOC_TBCTR_U_CMPA);
EPWM_setADCTriggerEventPrescale(EPWM1_BASE,
                                EPWM_SOC_A,
                                1);
EPWM_setCounterCompareValue(EPWM1_BASE,
                            EPWM_COUNTER_COMPARE_A,
                            1000);
EPWM_setTimeBasePeriod(EPWM1_BASE, 1999);
```

SOCA 在 ePWM 向上计数到 `CMPA` 时产生，每个 PWM 周期产生 1 次 SOCA。例程注释给出的参考关系是：

```text
100 MHz ePWM 时钟、TBPRD=1999 -> 约 50 kHz 触发
50 MHz ePWM 时钟 -> 约 25 kHz 触发
```

最终必须实测确认 ePWM 输入时钟和 TBCLK 分频，不能只照抄注释。

重要时序关系：

```text
PWM 触发频率 = 一批 SOC 的启动频率
ADC 内核时钟 = 每个 SOC 内部采样/转换的执行速度
一批 SOC 总耗时 < 下一次 SOCA 周期
```

如果一批 6 个 SOC 的采样和转换尚未完成，下一次 SOCA 又到来，就可能出现 SOC 排队、溢出或采样结果时序不符合预期。

## 5. ISR 如何完成平均

ISR 中先读取 A0 和 A1：

```c
adcAResult0 = ADC_readResult(myADC0_RESULT_BASE, ADC_SOC_NUMBER0);
adcAResult1 = ADC_readResult(myADC0_RESULT_BASE, ADC_SOC_NUMBER1);
```

然后读取 A2 的四次结果：

```c
adcAResult2 = (ADC_readResult(..., ADC_SOC_NUMBER2)
              + ADC_readResult(..., ADC_SOC_NUMBER3)
              + ADC_readResult(..., ADC_SOC_NUMBER4)
              + ADC_readResult(..., ADC_SOC_NUMBER5)) >> 2;
```

`>> 2` 等价于除以 4：

```text
average = (x2 + x3 + x4 + x5) / 4
```

这里四个 `uint16_t` 结果相加时，建议在自己改写代码时显式使用 `uint32_t` 累加器，避免后续提高采样次数时发生整数溢出：

```c
uint32_t sum = ADC_readResult(..., ADC_SOC_NUMBER2)
             + ADC_readResult(..., ADC_SOC_NUMBER3)
             + ADC_readResult(..., ADC_SOC_NUMBER4)
             + ADC_readResult(..., ADC_SOC_NUMBER5);
uint16_t average = (uint16_t)(sum >> 2);
```

## 6. 过采样到底降低什么

设每次采样值为：

```text
x[k] = 真实信号 + 随机噪声[k]
```

4 次平均后：

```text
x_avg = (x[0] + x[1] + x[2] + x[3]) / 4
```

理想条件下，随机噪声的标准差约降低为原来的：

```text
1 / sqrt(4) = 1 / 2
```

也就是说，4 倍过采样的主要收益是降低不相关随机噪声，而不是消除所有误差。

它不能自动消除：

- ADC 固有偏移误差；
- 增益误差；
- 参考源误差；
- 地线噪声引入的系统性误差；
- 时钟抖动导致的采样时刻误差；
- 高频开关噪声与真实信号相关的干扰。

这和你实习中观察到的“增益误差、偏移误差、地线噪声、时钟抖动”要分开分析：过采样主要针对随机噪声，不能把它当成万能校准手段。

## 7. 和电机控制的联系

这个例程可以类比电机控制中的电流采样，但不能直接等同于 FOC 电流环。

### 可以迁移的部分

```text
ePWM 定时触发 ADC
    ↓
连续获取同一采样量
    ↓
统计/滤波
    ↓
得到更稳定的反馈值
```

例如，电流传感器输出经过运放后接到 ADC，短时间内采多次，可以降低部分随机噪声，使电流反馈更平稳。

### 不能直接照搬的部分

FOC 电流环要求固定、低延迟、与 PWM 开关时刻严格对齐。过采样次数越多：

- 一批 SOC 总转换时间越长；
- ISR 读取和计算开销增加；
- 反馈延迟增大；
- 可能限制 PWM/电流环频率；
- 电流在四次采样之间可能已经发生变化。

因此真实电机控制常见做法是：

```text
PWM 中点单次采样 + 硬件/软件校准
```

只有在采样噪声较大、控制带宽允许、且经过时序验证时，才考虑多次过采样或专门的数字滤波。

## 8. 这个例程的推荐实验

### EXP-ADC-013-01：确认四次结果是否接近

- 给 A2 接稳定直流电压；
- 观察 SOC2、SOC3、SOC4、SOC5 原始结果；
- 记录四个值和平均值；
- 比较平均值与单次采样的波动。

### EXP-ADC-013-02：人为增加噪声

- 不改变程序结构；
- 让输入悬空或接入较长导线，观察噪声变化；
- 比较单次结果和 4 次平均结果的标准差；
- 不要把悬空输入的结果当作 ADC 精度结论。

### EXP-ADC-013-03：改变过采样次数

将 A2 的 SOC 数量改为 2、4、6 或 8 次，并记录：

|  次数 | 平均值标准差 | 一批转换耗时 | ISR 计算耗时 | 结论  |
| --: | -----: | -----: | -------: | --- |
|   1 |        |        |          | 基准  |
|   2 |        |        |          |     |
|   4 |        |        |          |     |
|   8 |        |        |          |     |

### EXP-ADC-013-04：验证实时性边界

逐渐提高 ePWM 触发频率，观察：

- ADCINT overflow 是否出现；
- SOC 是否发生排队或溢出；
- ISR 是否来不及清标志；
- 结果是否出现跳变或丢批次。

## 9. 面试表达

可以这样介绍这个例程：

> 我在 QX049 上使用 ePWM1 SOCA 作为 ADC 触发源，配置 SOC0、SOC1 采集两个普通通道，SOC2~SOC5 对同一通道进行 4 次重复采样，并将 SOC5 的转换完成事件映射到 ADCINT1。在 ISR 中读取四个结果求平均，用于观察过采样对随机噪声的抑制效果。同时记录了过采样带来的转换时间、ISR 负载和反馈延迟代价。这个方法可以迁移到电机电流采样，但 FOC 中需要优先保证 PWM 同步、采样时刻确定性和控制环低延迟。

## 10. 一句话记忆

> `ex10` 解决“同一时刻采多个通道”，`ex13` 解决“同一通道短时间采多次并平均”；它能降低部分随机噪声，但会增加 ADC 转换时间、ISR 负载和控制反馈延迟。
