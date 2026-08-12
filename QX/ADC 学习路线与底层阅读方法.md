---
title: "ADC 学习路线与底层阅读方法"
created: "2026-08-10"
tags: ["QX", "ADC", "学习路线", "driverlib", "嵌入式控制"]
---
# ADC 学习路线与底层阅读方法

关联笔记：[[ADC 软件触发例程学习笔记]]、[[ADC ePWM 定时触发与中断采样例程]]、[[F280049 时钟树]]、[[机器人方向4周学习计划]]、[[CPU Timer 学习笔记]]、[[SCI 串口例程学习笔记]]

## 总原则

不要在“只看 API”和“直接啃寄存器”之间二选一。推荐顺序是：

```text
先用 API 看懂硬件行为
-> 画出触发和数据路径
-> 修改一个参数并验证现象
-> 再用少量寄存器确认 API 的底层落点
```

API 是硬件行为的语言，寄存器是行为的实现证据。每个例程不需要逐位背完整 ADC 章节，只追与当前数据路径直接相关的字段。

## 每个例程的四遍读法

1. 用一句话说清例程解决的问题。
2. 按 `main.c -> board.h -> board.c` 追初始化、触发、等待/ISR、读取结果。
3. 画出：

   ```text
   触发源 -> SOCx -> ADCINx -> RESULTx -> ADCINT/ISR -> 应用变量
   ```

4. 只改一个可观察参数，例如通道、触发点、SOC 数量、采样窗口或保护阈值。

完成标准不是“看过代码”，而是能解释改动后为什么采样值、触发频率或中断行为会变化。

## 例程优先级

### 第一层：必须跑通并改动

#### `adc_ex1_soc_software`

建立 ADC 最小闭环：`ADC_setupSOC()` 配任务，`ADC_forceMultipleSOC()` 启动，ADCINT 标志表示完成，`ADC_readResult()` 读取结果。先只保留 A0，并用 SCI 打印 ADC 数字值和电压。

#### `adc_ex2_soc_epwm`

这是控制系统最重要的 ADC 例程。重点理解 `ePWM1 SOCA -> SOC0 -> ADC ISR` 的固定周期采样链，以及 PWM 计数点如何决定采样时刻。详见 [[ADC ePWM 定时触发与中断采样例程]]。

#### `adc_ex10_multiple_soc_epwm`

从单通道进入多通道控制采样：同一个 ePWM 事件触发 ADCA、ADCC 的多个 SOC，再由最后一个 SOC 表示这一组采样完成。要自己写一张“信号 -> ADC 模块 -> SOC -> RESULT -> ISR”表。

#### `adc_ex8_ppb_limits`

学习硬件保护。理解 PPB 如何关联一个 SOC、设置 `TRIPHI/TRIPLO`，并在超限时产生事件中断。之后把示例的 LSB 阈值换算成真实过压/过流阈值。

#### `adc_ex14_trigger_source`

作为触发源对比实验阅读：软件触发适合调试，CPU Timer 适合普通周期任务，ePWM 适合控制同步采样。不必一开始深挖所有实现细节。

### 第二层：按项目需求学习

#### `adc_ex7_ppb_offset`

传感器零点校正。电流采样存在偏置时很有用，重点是 PPB 偏移后的结果与原始结果的区别。

#### `adc_ex13_soc_oversampling`

对同一通道配置多个 SOC 并求平均，适合噪声较大的慢变量。要同时记录采样次数、平均效果和额外耗时。

#### `adc_ex11_burst_mode_epwm`

一个 ePWM 触发一串 SOC，并通过 SOC 优先级让高频信号比低频信号更常被采样。等你确实需要“电流高频、温度低频”时再深入。

#### `adc_ex1_soc_software_differential`

只有使用差分传感器或差分输入时才重点学习。差分模式下通道 0 代表 `ADCIN0(+) - ADCIN1(-)`，一次 SOC 产生一个差值结果。

### 第三层：暂时只理解用途

#### `adc_ex4_soc_software_sync`

通过 GPIO/X-BAR 同步 ADCA 和 ADCC。了解其用途即可，控制项目通常优先采用 ePWM 同步。

#### `adc_ex5_soc_continuous`

ADCINT 触发后续 SOC 的连续采样链，目标是追求最高采样率，不是固定控制周期的首选。

#### `adc_ex12_burst_mode_oversampling`

突发模式下大量 SOC 的过采样，和 `ex13` 目标相近但资源和时序更复杂，后学。

#### `adc_ex9_ppb_delay`

分析多个 ePWM 异步触发造成的采样延迟，属于性能优化和时序测量专题。

## API 与寄存器的对应范围

### `ex1` 先看这些 API

```text
ADC_setVREF()
ADC_setPrescaler()
ADC_setInterruptPulseMode()
ADC_enableConverter()
ADC_setupSOC()
ADC_setInterruptSource()
ADC_forceMultipleSOC()
ADC_getInterruptStatus()
ADC_readResult()
```

只需对应查看：

```text
ADCCTL1.INTPULSEPOS       中断脉冲位置
ADCSOCxCTL.CHSEL          SOC 选择的输入
ADCSOCxCTL.TRIGSEL        SOC 触发源
ADCSOCxCTL.ACQPS          采样窗口
ADCINTSEL1N2.INT1SEL      哪个 SOC 置位 ADCINT1
ADCRESULTx                SOCx 的转换结果
ADCBURSTCTL.BURSTEN       是否启用突发模式
```

### `ex2` 再增加 ePWM 字段

```text
TBCTR/TBPRD/CMPA          PWM 计数、周期、比较点
ETSEL.SOCASEL/SOCAEN     ADC SOC A 的触发事件和使能
ETPS.SOCAPRD              每几个事件产生一次 SOC A
```

### `ex8` 再增加 PPB 字段

```text
PPB 关联的 SOC
PPB 偏移校正
TRIPHI/TRIPLO             上下限
PPB 事件状态与中断使能
```

## 什么时候必须查寄存器

遇到以下情况才向寄存器继续追：

- API 参数不清楚，例如 `ACQPS`、触发源、SOC 优先级。
- 采样值、采样频率或 ISR 时序与预期不一致。
- 需要做电流零漂、过流保护、采样延迟优化。
- driverlib 没有直接提供所需功能。

不要在同一个字段上同时使用直接寄存器写法和 driverlib API。后执行的写入会覆盖先执行的配置；配置方式应尽量统一，优先使用 driverlib。

## ADC 学到什么程度算过关

面对一个新例程，能够回答下面五个问题就够进入项目：

1. 谁触发了 SOC？
2. 每个 SOC 采哪个输入，采样窗口是多少？
3. 结果进入哪个 RESULT 寄存器？
4. 谁表示这一组转换已经完成，是轮询还是 CPU ISR？
5. 采样时刻是否与 PWM/控制周期同步？

之后再用 SCI 将结果打印出来，最后把 `ex2 + ex10 + ex8` 合成一个 PWM、ADC、保护监控小项目。
