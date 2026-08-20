---
title: "eQEP 编码器基础与例程学习笔记"
tags: ["QX", "F280049", "eQEP", "编码器", "电机控制", "例程"]
created: 2026-08-17
---

# eQEP 编码器基础与例程学习笔记

关联笔记：[[机器人方向4周学习计划]]、[[ePWM 基础例程学习笔记]]、[[CPU Timer 学习笔记]]、[[F280049 时钟树]]、[[F280049 中断架构与 STM32 对比]]。

例程位置：

```text
f280049revb_evb_examples-master/f280049revb_evb_examples-master/examples_core0/examples/eqep/eqep_ex2_pos_speed
```

## 今天的核心结论

`eqep_ex2_pos_speed` 不是完整电机控制程序，而是一个“用 ePWM 模拟正交编码器，再用 eQEP 读取位置、方向和速度”的学习例程。

学习 eQEP 时先不要从寄存器表开始背，而要先建立链路：

```text
编码器 A/B/Index 信号
-> eQEP 正交解码
-> QPOSCNT 位置计数
-> 方向 directionQEP
-> 机械角 thetaMech
-> 电角度 thetaElec
-> 速度 speedRPMFR / speedRPMPR
```

## 编码器是什么

编码器是把机械转动转换成数字脉冲的传感器。电机转动时，编码器输出方波，芯片通过数这些方波判断：

- 转了多少：位置
- 往哪转：方向
- 转得多快：速度

增量式编码器上电后只知道“从现在开始走了多少”，不天然知道绝对零点，因此常配合 Index/Z 相寻找每圈零点。

## A/B 相

A 相和 B 相是两路相差 90 度的方波，所以叫正交信号。

```text
A: _|‾|_|‾|_|‾|_
B:   _|‾|_|‾|_|‾|_
```

eQEP 根据 A/B 哪一路领先判断方向：

```text
A 领先 B：一个方向
B 领先 A：相反方向
```

正转时，位置计数器 `QPOSCNT` 增加；反转时，`QPOSCNT` 减少。

## QPOSCNT 位置计数

`QPOSCNT` 是 eQEP 的当前位置计数器，可以把它理解成“编码器当前位置数字”。

```text
有效边沿到来且方向为正 -> QPOSCNT + 1
有效边沿到来且方向为反 -> QPOSCNT - 1
```

如果一圈是 4000 count：

```text
QPOSCNT = 1000 -> 约 1/4 圈
QPOSCNT = 2000 -> 约 1/2 圈
QPOSCNT = 3999 -> 接近 1 圈
```

例程中的 `thetaMech` 本质是：

```text
thetaMech = QPOSCNT / 每圈总 count
```

`thetaElec` 本质是：

```text
thetaElec = polePairs * thetaMech
```

## 角度与计数的关系

这部分只看最本质的关系，不先纠结寄存器名。

### 1. 物理世界里到底有什么

电机轴、转子磁铁、编码器圆盘都装在同一根轴上。

- 轴真的转了多少，这是现实中的转动
- 编码器圆盘跟着转，输出脉冲
- eQEP 把脉冲数成 `QPOSCNT`
- 软件再把 `QPOSCNT` 换成机械角和电角

### 2. QPOSCNT 是什么

`QPOSCNT` 是“当前位置计数值”。

如果一圈等于 4000 count，那么：

```text
QPOSCNT = 0     -> 当前位置在一圈起点
QPOSCNT = 1000  -> 约 1/4 圈
QPOSCNT = 2000  -> 约 1/2 圈
QPOSCNT = 3000  -> 约 3/4 圈
QPOSCNT = 4000  -> 约 1 圈
```

### 3. 机械角是什么

机械角描述的是“轴在几何上转到了哪里”。

本例一圈 4000 count，所以：

```text
机械角 thetaMech = QPOSCNT / 4000 * 360°
```

也可以理解成：

```text
QPOSCNT 只是格子数
thetaMech 是把格子数换成角度后的结果
```

### 4. 电角是什么

电角描述的是“磁场在电气周期里走到了哪里”。

本例极对数是 2，所以：

```text
电角 thetaElec = 2 * thetaMech
```

也就是说：

```text
机械转 1 圈，电角转 2 圈
```

### 5. 为什么还要 offset

机械零点和电角零点不是天然重合的。

- 机械零点：通常由 Index / Z 脉冲定义
- 电角零点：通常由转子磁铁对准 A 相来定义

如果这两个位置不重合，就要加一个电角 offset：

```text
thetaElec = polePairs * thetaMech + thetaOffsetElec
```

### 5.1 电角 offset 怎么求

电角 offset 不是凭空猜出来的，通常按下面流程标定：

1. 先用 `Index / Z` 建立机械零点。
2. 再通电让转子磁铁停在你定义的“电角 0°”位置。
3. 读取这时编码器的机械位置 `QPOSCNT = C0`。
4. 把 `C0` 换成机械角：

```text
thetaMech0 = C0 / 每圈count * 360°
```

5. 用“目标电角 = 0°”代入公式反推 offset：

```text
0° = polePairs * thetaMech0 + thetaOffsetElec
thetaOffsetElec = - polePairs * thetaMech0
```

6. 角度可以绕一圈等价，所以最后把结果整理到 `0° ~ 360°` 范围内。

### 5.2 用数字例子算一遍

假设：

- 一圈 = 4000 count
- 极对数 = 2
- 电角 0° 时，编码器读到 `QPOSCNT = 1500`

先把计数换成机械角：

```text
thetaMech0 = 1500 / 4000 * 360° = 135°
```

再代入公式：

```text
0° = 2 * 135° + thetaOffsetElec
thetaOffsetElec = -270°
```

`-270°` 和 `+90°` 是等价的，所以可以写成：

```text
thetaOffsetElec = 90°
```

于是运行时就变成：

```text
thetaElec = 2 * thetaMech + 90°
```

### 5.3 这个 offset 到底表示什么

它表示的是：

```text
当机械角为 0° 时，电角到底偏到哪里去了
```

或者反过来理解：

```text
电角 0° 对应的机械位置，离 Index 机械零点有多远
```

### 5.4 这件事为什么必须先知道一个“电角 0°”

因为只看编码器，你只能知道机械位置，不能知道磁铁朝向。

所以必须先人为制造一个已知的电角 0° 状态，比如：

- 给定子某一相通直流
- 让转子磁铁吸到指定位置
- 这个位置就当成电角 0°

然后再去读编码器值，才能把 offset 算出来。

### 6. 用本例举个完整例子

假设：

- 一圈 = 4000 count
- 极对数 = 2
- Z 位置定义为机械 0°
- Z 位置时，电角刚好偏了 90°

那么：

```text
QPOSCNT = 0
thetaMech = 0°
thetaElec = 90°
```

再看几个点：

| QPOSCNT | 机械角 thetaMech | 电角 thetaElec = 2 * thetaMech + 90° |
|---:|---:|---:|
| 0 | 0° | 90° |
| 500 | 45° | 180° |
| 1000 | 90° | 270° |
| 1500 | 135° | 0° |
| 2000 | 180° | 90° |

这张表的意思是：

- QPOSCNT 只告诉你轴转了多少格
- 机械角告诉你轴在几何上转到哪里
- 电角告诉你磁场走到哪里
- offset 告诉你机械零点和电角零点之间差多少

### 7. 这个例程里 `calAngle` 是什么

本例里有：

```c
p->thetaRaw = pos16bVal + p->calAngle;
```

这里的 `calAngle` 是机械角的校准偏移。

也就是说：

```text
先把编码器读数平移一下，再去算机械角
```

当前例程里 `CAL_ANGLE = 0`，所以默认没有做机械偏移补偿。

### 8. 最容易混淆的一句话

```text
Index 只是机械零点，不等于电角零点。
QPOSCNT = 0 只是软件把当前位置定义成 0，不是物理世界天然就必须是 0。
```

## 1X、2X、4X 计数

同一个 A/B 周期里有多个边沿。计数方式决定分辨率：

```text
1X：每周期数较少边沿，分辨率低
2X：数更多边沿
4X：A/B 所有边沿都利用，分辨率最高
```

本例在 `board.c` 中配置：

```c
EQEP_CONFIG_QUADRATURE | EQEP_CONFIG_1X_RESOLUTION
```

含义是使用正交 A/B 相模式，并采用 driverlib 定义的 `1X` 分辨率。

## Index / Z 相

Index 通常每转一圈出现一次，也叫 Z 相、零位、原点脉冲。

用途：给增量编码器提供每圈参考点。

本例配置：

```c
EQEP_POSITION_RESET_IDX
```

含义：每次检测到 Index 事件，就复位位置计数器 `QPOSCNT`。

这适合把一圈内的位置限制在从 Index 开始的相对位置。

## Strobe

Strobe 是额外触发输入，不是每个编码器都必须使用。

它常用于外部事件锁存当前位置，例如限位、同步信号或传感器触发：

```text
Strobe 事件发生 -> 把当时的 QPOSCNT 锁存下来
```

本例配置了 strobe 来源和锁存方式，但主线学习可以先忽略它，先掌握 A/B、Index、QPOSCNT。

### 输入反相什么时候使用？

输入反相用于把外部信号的实际极性和 eQEP 内部的有效边沿对齐，不是为了改变锁存值。

常见情况：

- 外部模块输出的是低有效脉冲：平时为高，事件到来时拉低。
- 信号经过反相器、光耦、比较器或开集电极电路后，逻辑电平被翻转。
- 希望把外部下降沿转换成 eQEP 看到的逻辑上升沿，从而继续使用 `RISING_STROBE` 或 `RISING_INDEX` 配置。
- A/B 某一路极性反了，导致计数方向不正确时，可用 `invertQEPA` 或 `invertQEPB` 修正。

例如：

```text
外部 strobe：高 ───┐____┌───
                    下降沿

invertStrobe = true
eQEP 看到的：低 ───┘‾‾‾‾└───
                    上升沿
```

反相后，eQEP 仍然锁存当前的 `QPOSCNT`；变化的只是“哪个物理边沿被当成有效边沿”。

## Index 门控

门控就是给某个信号加“允许条件”。

普通 Index：

```text
Index 来了就有效
```

门控 Index：

```text
Index 来了还不够，必须满足 strobe 等额外条件才有效
```

本例配置：

```c
EQEP_CONFIG_IGATE_DISABLE
```

含义：不启用 Index 门控。

## myEQEP0_init 的配置含义

函数位置：

```text
examples_core0/examples/eqep/eqep_ex2_pos_speed/board.c
```

核心配置：

```c
EQEP_setStrobeSource(myEQEP0_BASE, EQEP_STROBE_FROM_GPIO);
EQEP_setInputPolarity(myEQEP0_BASE, false, false, false, false);
EQEP_setDecoderConfig(myEQEP0_BASE,
    EQEP_CONFIG_QUADRATURE |
    EQEP_CONFIG_1X_RESOLUTION |
    EQEP_CONFIG_NO_SWAP |
    EQEP_CONFIG_IGATE_DISABLE);
EQEP_setPositionCounterConfig(myEQEP0_BASE, EQEP_POSITION_RESET_IDX, 4294967295U);
EQEP_setPosition(myEQEP0_BASE, 0U);
EQEP_enableUnitTimer(myEQEP0_BASE, 1000000U);
EQEP_setLatchMode(myEQEP0_BASE,
    EQEP_LATCH_UNIT_TIME_OUT |
    EQEP_LATCH_RISING_STROBE |
    EQEP_LATCH_RISING_INDEX);
EQEP_setCaptureConfig(myEQEP0_BASE,
    EQEP_CAPTURE_CLK_DIV_64,
    EQEP_UNIT_POS_EVNT_DIV_32);
EQEP_enableModule(myEQEP0_BASE);
EQEP_enableCapture(myEQEP0_BASE);
```

配置结果：

```text
输入模式：QEPA/QEPB 正交编码器模式
输入极性：A/B/Index/Strobe 都不反相
A/B 交换：不交换
Index 门控：关闭
位置计数：QPOSCNT 从 0 开始
位置复位：Index 事件复位
单位定时：每 1000000 个 SYSCLK 触发一次
位置锁存：unit timeout、strobe 上升沿、index 上升沿
低速捕获：CAPCLK = SYSCLK / 64，每 32 个 QCLK 捕获一次时间
```

如果 `SYSCLK = 100 MHz`，则：

```text
1000000 / 100 MHz = 10 ms
```

这就是固定时间测速 `speedRPMFR` 的时间窗口。

## FR 与 PR 两种速度估算

本例计算两个速度：

```text
speedRPMFR：固定时间内读位置差，适合中高速
speedRPMPR：固定位置间隔内测时间，适合低速
```

FR 思路：

```text
每隔固定时间 T 读取一次位置
速度 = (当前位置 - 上次位置) / T
```

PR 思路：

```text
走固定数量的编码器边沿
测这段边沿花了多久
速度 = 固定位置量 / 时间
```

记忆方式：

```text
高速：一小段时间里能数到很多脉冲，数脉冲更合适
低速：一小段时间里可能没几个脉冲，测一个脉冲花多久更合适
```

### SPEED_SCALER 有什么用？

问题：

```text
SPEED_SCALER — 低速测速换算系数
这个有什么用呢？
我还是没懂。
```

先不要把它当成复杂公式看。它可以理解成：

```text
SPEED_SCALER = 满速 6000 rpm 时，两次捕获事件之间应该间隔多少个 QCAPCLK 滴答。
```

在本例中，低速测速 `speedRPMPR` 不是看固定时间内走了多少位置，而是反过来：

```text
走固定数量的编码器边沿
测这段边沿花了多少时间
时间越短 -> 速度越快
时间越长 -> 速度越慢
```

代码里：

```c
temp1 = EQEP_getCapturePeriodLatch(EQEP1_BASE);
p->speedPR = _IQdiv(p->speedScaler, temp1);
p->speedRPMPR = _IQmpy(p->baseRPM, p->speedPR);
```

含义是：

```text
temp1       = 实际测到的时间间隔，单位是 QCAPCLK 滴答数
speedScaler = 满速时的标准时间间隔

当前速度占满速比例 = speedScaler / temp1
当前 rpm = BASE_RPM * 当前速度占满速比例
```

用生活例子理解：

```text
假设满速 6000 rpm 时，两次“咔嚓”之间是 125 个滴答。
这个 125 就是 SPEED_SCALER。

如果实际测到 250 个滴答：
125 / 250 = 0.5
说明当前速度是满速的一半。
6000 * 0.5 = 3000 rpm

如果实际测到 1250 个滴答：
125 / 1250 = 0.1
说明当前速度是满速的 10%。
6000 * 0.1 = 600 rpm
```

一句话记忆：

```text
SPEED_SCALER 是“满速时的基准时间”。
用它除以实际测到的时间，就得到“当前速度是满速的百分之几”。
再乘以 BASE_RPM，就得到实际转速。
```

## 本例模拟信号连接

```text
GPIO0 / ePWM1A -> GPIO6 / EQEP1_A
GPIO1 / ePWM1B -> GPIO7 / EQEP1_B
GPIO4          -> GPIO9 / EQEP1_INDEX
GPIO12         -> EQEP1_STROBE
```

`ePWM1A/ePWM1B` 模拟编码器 A/B 相；GPIO 模拟 Index。

注意：当前例程中 `board.h` 定义 Index 模拟输出为 GPIO4，但 `main.c` 中脉冲写的是 `GPIO_writePin(2, ...)`。如果后续 Index 复位现象不符合预期，应优先检查这里是否需要改为 `GPIO4` 或 `myGPIO0`。

## 建议练习顺序

1. 只理解 A/B 相和方向判断。
2. 在 Watch 窗口观察 `posSpeed.directionQEP`、`posSpeed.thetaRaw`。
3. 理解 `QPOSCNT` 如何换算成 `thetaMech`。
4. 理解 `thetaElec = polePairs * thetaMech`。
5. 观察 `speedRPMFR` 和 `speedRPMPR`。
6. 修改 `PWM_CLK`，观察速度变化。
7. 尝试交换 A/B 相或使用 `EQEP_CONFIG_SWAP`，观察方向是否反转。
8. 最后再理解 strobe、Index 门控、capture prescaler。

## 合格标准

学完这个例程后，应能说清楚：

```text
1. 编码器如何输出 A/B/Index 信号。
2. eQEP 如何通过 A/B 相判断方向并更新 QPOSCNT。
3. Index 为什么可以作为每圈零点。
4. QPOSCNT 如何换算机械角和电角度。
5. FR 和 PR 两种速度估算为什么分别适合高速和低速。
```
