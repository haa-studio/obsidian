---
title: "CPU Timer 学习笔记"
created: "2026-08-06 15:29"
tags: ["codex", "chat"]
---
# CPU Timer 学习笔记

# CPU Timer 学习笔记

## 这篇笔记解决的问题

代码里 `timer_1ms` 例程使用的是 `CPUTIMER0`。它的核心作用是：让 CPU Timer0 每 1ms 产生一次中断，在中断函数里累加 `time_tick`，主循环再用这个 tick 实现毫秒级延时和周期打印。

例程位置：

`f280049revb_evb_examples-master/f280049revb_evb_examples-master/examples_core0/examples/timer/timer_1ms/main.c`

## 这颗芯片里的 CPU Timer

`QXS320F280049RevB` 每个核心有 3 个 32 位 CPU Timer：

- `CPUTIMER0`
- `CPUTIMER1`
- `CPUTIMER2`

这颗芯片是双核结构，所以数据手册里会写总共有 6 个 32 位 CPU Timer，也就是每个核 3 个。

一般用途建议：

- `CPUTIMER0`：最适合做用户程序里的系统 tick，例如 1ms tick。
- `CPUTIMER1`：也可以给用户程序使用，例如另一个周期任务。
- `CPUTIMER2`：通常预留给 RTOS 或系统级用途。裸机程序也能用，但初学阶段先不用它。

## CPU Timer 的硬件模型

技术参考手册里的核心逻辑是：

```text
SYSCLK -> 预分频器 TPR/TPRH -> 32 位倒计数器 TIM -> 到 0 -> 产生 TINT 中断 -> 自动重载 PRD -> 下一轮
```

CPU Timer 是一个向下计数器。软件先把周期值写进 `PRD`，然后硬件把 `PRD` 加载到 `TIM`。`TIM` 会不断递减，直到减到 0。

当 `TIM` 到 0 时，硬件会做两件事：

1. 产生一次定时器中断脉冲 `TINT`。
2. 自动把 `PRD` 重新加载到 `TIM`，开始下一轮倒计时。

所以 CPU Timer 最常见的用途就是：固定周期进入中断函数。

## 关键寄存器

### TIM

`TIM` 是当前计数寄存器，保存 Timer 当前倒数到哪里。

例如 `PRD = 99999` 时，`TIM` 会从 `99999` 倒数到 `0`。

### PRD

`PRD` 是周期寄存器，决定 Timer 多久触发一次。

当 `TIM` 倒数到 0，硬件会把 `PRD` 重新加载到 `TIM`。

例程里的对应代码：

```c
CPUTimer_setPeriod(timer_base, period - 1);
```

### TCR

`TCR` 是 Timer 控制寄存器，里面有几个重要位：

- `TIF`：Timer interrupt flag。Timer 倒数到 0 后置位。
- `TIE`：Timer interrupt enable。置 1 后允许 Timer 产生中断请求。
- `TSS`：Timer stop status。控制 Timer 启动或停止。
- `TRB`：Timer reload bit。写 1 后把 `PRD` 重新加载到 `TIM`。
- `FREE/SOFT`：调试断点时 Timer 是否继续运行。

例程里的对应代码：

```c
CPUTimer_stopTimer(timer_base);
CPUTimer_reloadTimerCounter(timer_base);
CPUTimer_enableInterrupt(timer_base);
CPUTimer_startTimer(timer_base);
```

### TPR / TPRH

`TPR` 和 `TPRH` 是预分频相关寄存器。

手册里的规则是：

```text
TIM 每 (TDDR + 1) 个 Timer 输入时钟周期递减 1 次
```

如果预分频值是 `0`，那么每 1 个 SYSCLK 周期，`TIM` 减 1。

如果预分频值是 `99`，那么每 100 个 SYSCLK 周期，`TIM` 减 1。

例程里的对应代码：

```c
uint32_t prescaler = CPUTIMER_CLOCK_PRESCALER_1;
CPUTimer_setPreScaler(timer_base, prescaler);
```

## 定时周期怎么算

Timer 中断周期公式：

```text
中断周期 = (PRD + 1) * (Prescaler + 1) / TimerClock
```

如果反过来，想要一个目标中断频率：

```text
PRD = TimerClock / ((Prescaler + 1) * 目标中断频率) - 1
```

例程中设置 1ms tick：

```c
#define SYS_TICKS_PER_SECOND    1000

uint32_t SystemClock = SysCtl_getClock(DEVICE_OSCSRC_FREQ);
uint32_t prescaler   = CPUTIMER_CLOCK_PRESCALER_1;
uint32_t period      = SystemClock / ((prescaler + 1) * SYS_TICKS_PER_SECOND);

CPUTimer_setPeriod(timer_base, period - 1);
```

假设系统时钟是 100MHz，目标中断频率是 1000Hz，也就是 1ms 一次：

```text
PRD = 100,000,000 / 1000 - 1
PRD = 99,999
```

这样 Timer 从 `99999` 倒数到 `0`，一共经历 100000 个系统时钟周期，也就是 1ms。

## `timer_1ms` 例程流程

主函数的核心流程：

```c
Interrupt_initVectorTable();
Device_init();
StdOutInit(&ScibRegs, 115200, 12, GPIO_12_SCIB_TX);
TIMER0_init();
EINT;
ERTM;
```

含义：

- `Interrupt_initVectorTable()`：初始化中断向量表。
- `Device_init()`：初始化芯片系统基础状态和时钟。
- `StdOutInit(...)`：把 `printf` 重定向到 `SCIB` 串口输出。
- `TIMER0_init()`：配置 Timer0。
- `EINT`：打开全局中断。
- `ERTM`：打开实时调试相关中断能力。

## `TIMER_init()` 做了什么

例程里的核心代码：

```c
CPUTimer_stopTimer(timer_base);
CPUTimer_setPeriod(timer_base, period - 1);
CPUTimer_setPreScaler(timer_base, prescaler);
CPUTimer_reloadTimerCounter(timer_base);
CPUTimer_enableInterrupt(timer_base);
CPUTimer_startTimer(timer_base);
```

对应硬件动作：

1. 停止 Timer，避免配置过程中计数器乱跑。
2. 设置 `PRD` 周期值。
3. 设置预分频。
4. 触发重载，把 `PRD` 加载到 `TIM`。
5. 允许 Timer 到 0 后产生中断。
6. 启动 Timer。

## 中断函数做了什么

例程里的中断函数：

```c
static volatile uint32_t time_tick;

__interrupt void timer0_isr(void)
{
    time_tick++;
}
```

因为 Timer0 被配置成每 1ms 中断一次，所以 `time_tick` 每 1ms 加 1。

`volatile` 很重要。它告诉编译器：这个变量可能在中断里被修改，不要把它优化掉。

## `delay_ms()` 的原理

例程里的延时函数：

```c
void delay_ms(volatile uint32_t ms)
{
    volatile uint32_t tc = time_tick;
    while (time_tick - tc < ms) {

    }
}
```

它先记录当前 `time_tick`，然后等待 `time_tick` 增加指定数量。

因为当前例程里 1 tick = 1ms，所以：

```c
delay_ms(1000);
```

就是等待 `time_tick` 增加 1000 次，也就是大约 1000ms。

注意：如果把 `SYS_TICKS_PER_SECOND` 改成 2000，那么 1 tick 就是 0.5ms，此时 `delay_ms(1000)` 实际只会延时约 500ms。`delay_ms()` 这个名字成立的前提是 tick 真的等于 1ms。

## 主循环现象

例程主循环：

```c
while (1)
{
    printf(" 1 \r\n");
    delay_ms(1000);
    printf(" 2 \r\n");
    delay_ms(1000);
    printf(" 3 \r\n");
}
```

串口现象大概是：

```text
1
等待 1 秒
2
等待 1 秒
3
1
等待 1 秒
...
```

这里 `3` 后面没有 `delay_ms(1000)`，所以 `3` 和下一轮的 `1` 可能挨得很近。更合理的练习写法：

```c
while (1)
{
    printf(" 1 \r\n");
    delay_ms(1000);
    printf(" 2 \r\n");
    delay_ms(1000);
    printf(" 3 \r\n");
    delay_ms(1000);
}
```

## 为什么插上烧录线就能看到串口打印

例程中 `printf` 被映射到了 `SCIB_TX`：

```c
StdOutInit(&ScibRegs, 115200, 12, GPIO_12_SCIB_TX);
```

这表示芯片通过 `GPIO12 / SCIB_TX` 输出串口数据。

只插烧录线就能看到打印，通常说明开发板上已经把 `GPIO12 / SCIB_TX` 接到了板载 USB 转串口或调试器串口通道。实际链路是：

```text
芯片 GPIO12 / SCIB_TX -> 板载 USB 串口 RX -> USB 线 -> 电脑虚拟 COM 口 -> 串口助手或 IDE 终端
```

所以不是绕过了 `GPIO12`，而是开发板已经通过板上走线帮你接好了。

## Timer0、Timer1、Timer2 的中断连接

资料里提到：

- `Timer0` 通过 PIE，通常是 `INT1.10`。
- `Timer1` 直接连接到 CPU `INT13`。
- `Timer2` 直接连接到 CPU `INT14`。

初学阶段可以先记住 driverlib 层面的用法：

```c
Interrupt_register(INT_TIMER0, timer0_isr);
Interrupt_enable(INT_TIMER0);
```

以后如果切换 Timer1 或 Timer2，就用对应的中断号。

## Timer2 的特殊点

Timer0 和 Timer1 基本连接到 `SYSCLK`。

Timer2 默认也连接到 `SYSCLK`，但它可以通过 `TMR2CLKCTL` 选择其他时钟源，比如：

- `INTOSC1`
- `INTOSC2`
- `XTAL`

Timer2 能选独立时钟源，所以更偏系统级用途，例如 RTOS tick、频率测量、安全诊断等。

## 在机器人控制中的用途

CPU Timer 适合做：

- 系统 tick
- 低频周期任务
- 通讯超时检测
- 状态机调度
- 速度计算周期
- 软件延时

但高频电机控制，例如 FOC 电流环，通常更常用：

```text
EPWM 触发 ADC -> ADC 中断里跑电流环
```

CPU Timer 更适合系统调度和低频任务。

一个典型裸机任务框架：

```c
volatile uint32_t tick_1ms;
volatile uint8_t task_1ms_flag;
volatile uint8_t task_10ms_flag;
volatile uint8_t task_100ms_flag;

__interrupt void timer0_isr(void)
{
    tick_1ms++;
    task_1ms_flag = 1;

    if (tick_1ms % 10 == 0)
        task_10ms_flag = 1;

    if (tick_1ms % 100 == 0)
        task_100ms_flag = 1;
}

while (1)
{
    if (task_1ms_flag)
    {
        task_1ms_flag = 0;
        read_encoder();
        calculate_speed();
    }

    if (task_10ms_flag)
    {
        task_10ms_flag = 0;
        can_process();
    }

    if (task_100ms_flag)
    {
        task_100ms_flag = 0;
        printf_status();
    }
}
```

## 建议练习

### 练习 1：跑原例程

编译下载 `timer_1ms`，打开串口，确认能看到 `1 2 3` 打印。

### 练习 2：补上第三个延时

在：

```c
printf(" 3 \r\n");
```

后面加：

```c
delay_ms(1000);
```

观察 `1、2、3` 是否都按 1 秒间隔打印。

### 练习 3：改成 500ms 打印

把主循环改成：

```c
while (1)
{
    printf("tick\r\n");
    delay_ms(500);
}
```

### 练习 4：改变 tick 频率

把：

```c
#define SYS_TICKS_PER_SECOND 1000
```

改成：

```c
#define SYS_TICKS_PER_SECOND 2000
```

然后观察 `delay_ms(1000)` 是否变成约 500ms。

这个实验能帮助理解 `delay_ms()` 对 1ms tick 的依赖。

### 练习 5：用 flag 代替中断里打印

不要在中断里直接 `printf`。推荐这样写：

```c
volatile uint32_t time_tick;
volatile uint8_t flag_1s;

__interrupt void timer0_isr(void)
{
    time_tick++;

    if (time_tick % 1000 == 0)
    {
        flag_1s = 1;
    }
}

while (1)
{
    if (flag_1s)
    {
        flag_1s = 0;
        printf("1 second\r\n");
    }
}
```

这样更接近真实工程。中断里只做短、快、确定的事情；慢操作放到主循环。

## 一句话总结

`timer_1ms` 例程真正教的不是打印数字，而是嵌入式控制程序的基础骨架：

```text
硬件 Timer -> 固定周期中断 -> 系统 tick -> 周期任务/延时/控制循环
```

这套结构后面会继续用在 ADC 采样、PWM 控制、编码器测速、CAN 通讯超时和机器人控制状态机里。
