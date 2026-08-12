---
title: "F280049 时钟配置与 Device_init"
created: "2026-08-08"
tags: ["F280049", "时钟", "Device_init"]
---

# F280049 时钟配置与 Device_init

## `Device_init()` 的职责

`Device_init()` 先建立全芯片的时钟环境：选择时钟源、设置 PLL 与分频器、设置低速外设时钟，并在需要时保护 Flash 时钟。

```c
void Device_init(void)
{
    SysCtl_setClock(DEVICE_SETCLOCK_CFG);
    SysCtl_setLowSpeedClock(LSPCLK_PRESCALE);

    if (SysCtl_getClock(DEVICE_OSCSRC_FREQ) > 100000000)
        Flash_setClkDiv(4);
}
```

它不负责配置 SCI 引脚、波特率或中断。SCI 的这些工作在各例程的 `Board_init()` 中完成。

```text
Device_init()      配置整机时钟
Device_initGPIO()  初始化 GPIO 基础状态
Board_init()       配置引脚复用、打开外设时钟、配置 SCI/FIFO/中断
```

## 配置值与频率值的区别

```c
SysCtl_setClock(DEVICE_SETCLOCK_CFG);
```

`DEVICE_SETCLOCK_CFG` 是写入时钟控制寄存器所需的位字段，包含时钟源、PLL 是否启用、倍频和各级分频，因而可以告诉芯片“怎样产生时钟”。

```c
#define DEVICE_SYSCLK_FREQ (((DEVICE_OSCSRC_FREQ * IMULT_VAL / 1) / ODIV_VAL) / 1)
```

`DEVICE_SYSCLK_FREQ` 是软件对最终系统时钟频率的计算结果，供延时、PWM、定时器和外设驱动计算分频使用；它不能传给 `SysCtl_setClock()`。

```text
DEVICE_SETCLOCK_CFG  = 写给硬件的配置说明
DEVICE_SYSCLK_FREQ   = 软件使用的频率计算结果
```

## 当前工程的配置

`libs/device/device.h` 的主要参数：

```c
#define DEVICE_OSCSRC_FREQ 10000000U
#define IMULT_VAL          40
#define ODIV_VAL           4
#define LSPCLK_PRESCALE    SYSCTL_LSPCLK_PRESCALE_1

#define DEVICE_SETCLOCK_CFG                                        \
    (SYSCTL_OSCSRC_XTAL | SYSCTL_IMULT(IMULT_VAL) | SYSCTL_IDIV(1) \
        | SYSCTL_ODIV(ODIV_VAL) | SYSCTL_SYSDIV(1) | SYSCTL_PLL_ENABLE)
```

计算结果：

```text
外部晶振 = 10 MHz
SYSCLK = 10 MHz x 40 / 1 / 4 / 1 = 100 MHz
LSPCLK = SYSCLK / 1 = 100 MHz
```

`SYSCTL_OSCSRC_XTAL` 已经表示选择外部晶振/外部振荡器路径。不要只修改 `DEVICE_OSCSRC_FREQ` 来“假装”硬件频率改变；它必须和板上实际时钟源一致。

## 修改时钟的原则

改变外部晶振频率时，需要同时修改 `DEVICE_OSCSRC_FREQ` 和 PLL 分频参数。例如外部晶振为 20 MHz，若仍希望得到 100 MHz 系统时钟：

```c
#define DEVICE_OSCSRC_FREQ 20000000U
#define IMULT_VAL          40
#define ODIV_VAL           8
```

```text
20 MHz x 40 / 8 = 100 MHz
```

更改前必须确认：

1. 时钟源与硬件接法一致。
2. PLL 的输入、倍频和输出频率满足数据手册限制。
3. `DEVICE_SYSCLK_FREQ` 与实际配置保持同步。
4. 高系统时钟时 Flash 时钟分频满足限制。
5. 先用 `sci_puts_printf` 验证串口输出，再验证 PWM、定时器和 ADC。

## 与外设的关系

大多数例程使用共享的 `device.h` 宏，因此会自动随此配置变化。SCI/SPI 通常传入 `DEVICE_LSPCLK_FREQ`，I2C、ePWM、eQEP、CPU Timer 等常使用 `DEVICE_SYSCLK_FREQ`。

但 `DEVICE_SYSCLK_FREQ = 100 MHz` 只确定了系统时钟，不自动保证 ePWM 的 `TBCTR` 以 100 MHz 计数。ePWM 的实际时间基准还要看全局 `EPWMCLK` 分频及各模块的 `HSPCLKDIV`、`CLKDIV`；详见 [[F280049 时钟树]] 与 [[ADC ePWM 定时触发与中断采样例程]]。

少量低功耗例程会在运行中再次调用 `SysCtl_setClock()`，或者存在硬编码的时基参数；修改全局时钟后，这些例程需要单独核对。

相关：[[F280049 时钟树]]、[[SCI FIFO 与 STM32 UART 对比]]、[[SCI 串口例程学习笔记]]
