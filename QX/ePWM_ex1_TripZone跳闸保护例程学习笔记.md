---
type: experiment-note
topic: ePWM-protection
status: learning
updated: 2026-08-24
tags:
  - QX049
  - ePWM
  - Trip-Zone
  - TZ
  - OST
  - CBC
  - 电机控制
  - 过流保护
---

# `epwm_ex1_trip_zone` 跳闸保护例程学习笔记

## 1. 这个例程演示什么

例程目录：

`C:\Users\OSS\Desktop\f280049revb_evb_examples-master\examples_core0\examples\epwm\epwm_ex1_trip_zone`

它演示 ePWM 的 Trip Zone（TZ，跳闸区）子模块：当外部故障信号到来时，不等待 CPU 处理，ePWM 硬件直接把 PWM 输出强制到预先配置的状态。

本例的连接关系是：

```text
GPIO12（外部故障输入）
        ↓
Input X-BAR 的 INPUT1
        ↓
ePWM 的 TZ1 输入
        ↓
TZ 子模块
        ├── ePWM1A：OST 一次性跳闸
        └── ePWM2A：CBC 逐周期跳闸
```

README 要求：

- GPIO0：观察 ePWM1A；
- GPIO2：观察 ePWM2A；
- GPIO12：故障输入，拉低触发；
- GPIO11：观察 ePWM2 的 TZ ISR，每次事件翻转一次。

## 2. 先理解：Trip Zone 不是普通软件中断

普通软件保护通常是：

```text
ADC/比较器发现过流
    ↓
置位标志或产生 CPU 中断
    ↓
CPU 进入 ISR
    ↓
软件修改 PWM 寄存器
```

这种方式受中断延迟、代码执行时间和 CPU 是否被占用影响。

Trip Zone 的硬件保护路径是：

```text
故障信号
    ↓
TZ1 硬件输入
    ↓
ePWM 输出逻辑立即强制动作
    ↓
同时可选地产生 TZ ISR，供软件记录和处理
```

因此要区分：

> TZ 输出动作是硬件保护本体；TZ ISR 主要用于计数、记录、上报和恢复管理，不应该依赖 ISR 才完成第一时间关断。

## 3. GPIO12 如何进入 TZ1

`board.c` 中：

```c
GPIO_setPinConfig(GPIO_12_GPIO12);
GPIO_setPadConfig(myGPIO12, GPIO_PIN_TYPE_STD | GPIO_PIN_TYPE_PULLUP);
GPIO_setQualificationMode(myGPIO12, GPIO_QUAL_ASYNC);
GPIO_setDirectionMode(myGPIO12, GPIO_DIR_MODE_IN);
```

含义：

- GPIO12 配成普通数字输入；
- 打开内部上拉，因此未外接故障时默认为高电平；
- 使用异步输入资格，故障信号不必等待普通 GPIO 同步采样；
- 外部将 GPIO12 拉低，就形成有效故障。

然后：

```c
XBAR_setInputPin(XBAR_INPUT1, 12);
```

把 GPIO12 路由到 Input X-BAR 的 INPUT1。之后 ePWM TZ1 使用的就是这条内部路由信号。

```text
GPIO12 = 1：正常
GPIO12 = 0：TZ1 有效，触发跳闸
```

实际产品中，TZ1 也可以来自比较器、CMPSS、ADC 越限逻辑、外部故障芯片等，不一定来自 GPIO。

## 4. ePWM 正常波形配置

### ePWM1

```c
EPWM_setTimeBasePeriod(EPWM1_BASE, 12000);
EPWM_setCounterCompareValue(EPWM1_BASE,
                            EPWM_COUNTER_COMPARE_A,
                            6000);
EPWM_setTimeBaseCounterMode(EPWM1_BASE,
                            EPWM_COUNTER_MODE_UP_DOWN);
```

AQ 配置为：

```text
TBCTR 上数到 CMPA：EPWM1A 置高
TBCTR 下数到 CMPA：EPWM1A 置低
```

因此正常状态下，GPIO0 输出一个约 50% 占空比的上下计数 PWM。

### ePWM2

```c
EPWM_setTimeBasePeriod(EPWM2_BASE, 6000);
EPWM_setCounterCompareValue(EPWM2_BASE,
                            EPWM_COUNTER_COMPARE_A,
                            3000);
```

同样是约 50% 占空比，但周期是 ePWM1 的一半。

两个 ePWM 都设置了：

```c
EPWM_setClockPrescaler(...,
                       EPWM_CLOCK_DIVIDER_4,
                       EPWM_HSCLOCK_DIVIDER_4);
```

即总时基分频约为 `/16`。若当前 EPWMCLK 为 100 MHz，则：

```text
TBCLK = 100 MHz / 16 = 6.25 MHz
f_EPWM1 ≈ 6.25 MHz / (2 × 12000) ≈ 260.4 Hz
f_EPWM2 ≈ 6.25 MHz / (2 × 6000)  ≈ 520.8 Hz
```

这里的频率是按 EPWMCLK=100 MHz 的假设计算，最终应使用示波器实测，不能只相信注释或默认时钟假设。

## 5. `TZA_ACTION_HIGH` 到底做什么

源码：

```c
EPWM_setTripZoneAction(myEPWM1_BASE,
                       EPWM_TZ_ACTION_EVENT_TZA,
                       EPWM_TZ_ACTION_HIGH);
EPWM_setTripZoneAction(myEPWM2_BASE,
                       EPWM_TZ_ACTION_EVENT_TZA,
                       EPWM_TZ_ACTION_HIGH);
```

含义是：当 TZ 事件作用到 ePWMxA 时，强制 ePWMxA 输出高电平。

这只是本例方便用示波器观察的动作，不是通用安全状态。真实电机驱动中可能需要：

- 强制低电平；
- 高阻态；
- 关闭 PWM 输出使能；
- 由栅极驱动器的 EN/FAULT 脚统一关断。

安全电平必须根据功率级拓扑和栅极驱动器逻辑确认。不能看到 `EPWM_TZ_ACTION_HIGH` 就认为“故障时应该输出高电平”。

## 6. ePWM1：OST 一次性跳闸

配置：

```c
EPWM_enableTripZoneSignals(EPWM1_BASE,
                           EPWM_TZ_SIGNAL_OSHT1);
EPWM_enableTripZoneInterrupt(EPWM1_BASE,
                             EPWM_TZ_INTERRUPT_OST);
```

OST = One-Shot Trip，一次性跳闸。

故障过程：

```text
GPIO12 拉低
    ↓
TZ1 触发 OSHT1
    ↓
EPWM1A 立即强制高电平
    ↓
OST 标志锁存
```

即使 GPIO12 随后恢复高电平，OST 状态也不会自动恢复，PWM 仍保持跳闸动作。必须由软件明确清除 OST 标志，才允许恢复正常 PWM。

ISR 中的恢复代码被注释掉了：

```c
// EPWM_clearTripZoneFlag(EPWM1_BASE,
//                        (EPWM_TZ_INTERRUPT | EPWM_TZ_FLAG_OST));
```

所以本例的 ePWM1 设计成“触发后锁存”，适合模拟严重故障：过流、短路、驱动器故障、母线异常等。恢复前通常还应确认故障原因已经消失，并执行重新使能流程，而不是无条件清除标志。

## 7. ePWM2：CBC 逐周期跳闸

配置：

```c
EPWM_enableTripZoneSignals(EPWM2_BASE,
                           EPWM_TZ_SIGNAL_CBC1);
EPWM_enableTripZoneInterrupt(EPWM2_BASE,
                             EPWM_TZ_INTERRUPT_CBC);
```

CBC = Cycle By Cycle，逐周期跳闸。

故障保持低电平时：

```text
GPIO12 拉低
    ↓
当前 PWM 周期被强制到 TZA 配置状态
    ↓
到周期边界后，硬件按 CBC 规则清除/重新评估
    ↓
若故障仍存在，下一个周期再次跳闸
```

因此 CBC 不像 OST 那样永久锁存。故障信号撤销且硬件完成周期边界处理后，PWM 可以恢复正常输出。

它适合“需要逐周期限制，但允许故障消失后自动恢复”的场景，例如某些电流限流或周期性过流保护。

## 8. 两个 ISR 分别做什么

### ePWM1 OST ISR

```c
__interrupt void epwm1TZISR(void)
{
    epwm1TZIntCount++;
    Interrupt_clearACKGroup(INTERRUPT_ACK_GROUP2);
}
```

它只做两件事：

1. 记录 OST 事件次数；
2. 清 PIE 第 2 组应答，允许同组中断继续响应。

它没有清 OST 标志，所以 ePWM1 保持锁存保护。

### ePWM2 CBC ISR

```c
__interrupt void epwm2TZISR(void)
{
    epwm2TZIntCount++;
    GPIO_togglePin(myGPIO11);
    EPWM_clearTripZoneFlag(myEPWM2_BASE,
                           (EPWM_TZ_INTERRUPT | EPWM_TZ_FLAG_CBC));
    Interrupt_clearACKGroup(INTERRUPT_ACK_GROUP2);
}
```

它做四件事：

1. 记录 CBC 事件次数；
2. 翻转 GPIO11，方便用示波器或逻辑分析仪观察中断频率；
3. 清除 ePWM2 的 CBC/TZ 标志；
4. 清 PIE 第 2 组应答。

注意：清标志不等于强行解除外部故障。如果 GPIO12 仍然为低，故障源仍有效，CBC 仍会继续产生事件。

## 9. OST 与 CBC 对比

| 特性 | OST | CBC |
|---|---|---|
| 全称 | One-Shot Trip | Cycle By Cycle |
| 触发后 | 锁存 | 按周期处理 |
| 输入恢复高电平后 | 不会自动恢复 | 通常可在周期边界恢复 |
| 典型用途 | 短路、严重过流、驱动器故障 | 周期限流、可恢复过流 |
| 是否需要软件清除 | 需要清 OST 标志 | 需要按配置清 CBC/中断标志 |
| 本例输出 | 强制 EPWM1A 高 | 强制 EPWM2A 高 |

记忆方式：

```text
OST：一次触发，锁住不放
CBC：这一周期保护，下周期重新看故障
```

## 10. 和 ADC、电机控制的联系

你前面学习的 ADC/ePWM 链路是：

```text
PWM 触发 ADC
    ↓
ADC 采样电流
    ↓
ADC/PPB 判断是否越限
```

Trip Zone 可以接在保护链的最后：

```text
电机相电流
    ↓
采样电阻/运放
    ↓
ADC 或 CMPSS
    ↓
过流比较结果
    ↓
X-BAR / TZ1
    ↓
ePWM 硬件强制安全状态
```

完整的运动控制执行链可以画成：

```text
ePWM 输出功率开关控制
        ↓
逆变器和电机
        ↓
电流传感器 -> ADC -> FOC 电流环
        ↓                  ↓
过流比较/PPB --------> 下一周期 PWM
        ↓
Trip Zone 快速关断/保护
```

这里要区分两种作用：

- ADC ISR：用于正常控制反馈，例如 Clarke/Park、PI、SVPWM；
- Trip Zone：用于异常保护，优先保证功率级不损坏。

不要把 Trip Zone 当成普通控制回路。它不负责计算占空比，只负责在故障时覆盖 AQ 的正常输出。

## 11. 推荐上板实验

### EXP-TZ-001：观察正常 PWM

- GPIO12 保持高电平；
- 示波器观察 GPIO0、GPIO2；
- 记录两个 PWM 的频率、周期、占空比；
- 记录 GPIO11 是否保持不变。

### EXP-TZ-002：触发 OST

- 先观察 GPIO0；
- 将 GPIO12 短暂拉低后释放；
- 观察 GPIO0 是否被强制高电平并保持；
- 观察 `epwm1TZIntCount` 是否增加；
- 读取并记录 OST 标志；
- 仅在确认故障已消失后，手动取消注释清 OST 代码验证恢复。

### EXP-TZ-003：观察 CBC

- 观察 GPIO2 和 GPIO11；
- 保持 GPIO12 为低电平一段时间；
- 观察 GPIO11 周期性翻转；
- 恢复 GPIO12 为高电平；
- 对比 GPIO2 是否在故障撤销后恢复 PWM。

### EXP-TZ-004：验证安全动作

将 `EPWM_TZ_ACTION_HIGH` 依次改为符合硬件允许的其他动作，比较：

- `EPWM_TZ_ACTION_LOW`；
- `EPWM_TZ_ACTION_HIGH_Z`；
- 实际驱动器 EN/FAULT 关断方案。

不要在未确认板卡电气连接的情况下把 GPIO0/GPIO2 接到功率级进行实验。

## 12. 面试表达

可以这样介绍：

> 我在 QX049 上验证了 ePWM Trip Zone 保护链路。通过 Input X-BAR 将 GPIO12 故障输入接入 TZ1，同时配置 ePWM1 为 OST 一次性跳闸、ePWM2 为 CBC 逐周期跳闸。故障发生时由 ePWM 硬件直接覆盖正常 AQ 输出，分别强制到预设状态；ISR 只负责事件计数、标志管理和状态指示。我进一步理解了 OST 适合锁存严重故障，CBC 适合可恢复的逐周期限制，并将它们映射到电机驱动中的过流/短路保护场景。

## 13. 一句话记忆

> ePWM 正常由 TBCTR、CMPA 和 AQ 产生波形；Trip Zone 是更高优先级的硬件保护路径，故障到来时直接覆盖正常 PWM 输出。OST 锁存，CBC 逐周期恢复。

