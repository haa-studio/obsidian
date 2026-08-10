---
title: "SCI 串口例程学习笔记"
created: "2026-08-07 17:53"
tags: ["codex", "chat"]
---
# SCI 串口例程学习笔记

# SCI 串口例程学习笔记

## 这篇笔记解决的问题

本笔记整理 `examples_core0/examples/sci` 目录下的 SCI 串口例程，目标是理解这套开发板上的串口如何打印、发送、接收、中断处理、FIFO 收发以及 DMA 收发。

SCI 可以先理解为 UART 串口。它常用于 `printf` 调试、接收上位机命令、输出运行状态、参数配置，以及与其他控制板通信。

机器人控制板调试早期，SCI 非常重要。后面可以用串口发送命令：

```text
start
stop
duty 50
target 1000
status
```

## SCI 目录下的例程

目录位置：

`f280049revb_evb_examples-master/f280049revb_evb_examples-master/examples_core0/examples/sci`

包含 7 个例程：

```text
sci_puts_printf
sci_send_polling_recv_interrupt
sci_send_polling_recv_interrupt_232
sci_send_polling_recv_interrupt_485
sci_interrupts_fifo
sci_send_dma_recv_interrupt
sci_send_recv_dma
```

它们是一条从简单到复杂的学习路线：

```text
printf 输出 -> 轮询发送 -> 中断接收 -> FIFO 收发 -> DMA 发送 -> DMA 收发完整数据帧
```

## 1. sci_puts_printf

位置：`examples_core0/examples/sci/sci_puts_printf/main.c`

这个例程最适合先跑。它演示把 `printf` / `puts` 重定向到 SCI 串口，然后周期打印内容。

关键初始化：

```c
StdOutInit(&ScibRegs, 921600, 12, GPIO_12_SCIB_TX);
```

含义：

```text
使用 SCIB
TX 引脚是 GPIO12
波特率 921600
只发送，不接收
```

它重写了：

```c
int putchar(int c)
```

`printf()` 最终会逐字符调用 `putchar()`。`putchar()` 内部逻辑是：等待 TX 发送缓冲区为空，然后把一个字符写入 `SCITXBUF`。

关键代码：

```c
while (!dbg_uart->SCICTL2.bit.TXEMPTY)
    ;
dbg_uart->SCITXBUF.all = c;
```

主循环会周期输出：

```text
hello world
loop0, PI=3.141592
hello world
loop1, PI=3.141592
```

学习重点：SCI 发送、`printf` 重定向、波特率设置、TX GPIO 复用、轮询发送。

## 2. sci_send_polling_recv_interrupt

位置：`examples_core0/examples/sci/sci_send_polling_recv_interrupt`

这是最重要的基础收发例程。它的功能是：板子先打印欢迎信息；用户通过串口助手输入字符；板子收到一行后，把内容回显回来。

### 关键配置

在 `board.h` 中：

```c
#define SCI_BASE SCIB_BASE
#define SCI_TX_PIN_NUM 12
#define SCI_RX_PIN_NUM 13
#define SCI_BAUD_RATE 115200
#define SCI_CONFIG_WLEN SCI_CONFIG_WLEN_8
#define SCI_CONFIG_STOP SCI_CONFIG_STOP_ONE
#define SCI_CONFIG_PAR  SCI_CONFIG_PAR_NONE
```

含义：

```text
串口模块：SCIB
TX：GPIO12
RX：GPIO13
波特率：115200
格式：8 位数据，1 位停止位，无校验，也就是 8N1
```

串口助手也要设置成：`115200`、`8 data bits`、`None parity`、`1 stop bit`、`No flow control`。

### main 初始化流程

```c
Device_init();
Device_initGPIO();
Interrupt_initModule();
Interrupt_initVectorTable();
Board_init();
```

含义：

- `Device_init()`：初始化芯片时钟和基础外设。
- `Device_initGPIO()`：初始化 GPIO。
- `Interrupt_initModule()`：初始化中断控制器。
- `Interrupt_initVectorTable()`：初始化中断向量表。
- `Board_init()`：配置引脚、SCI 和中断。

`Board_init()` 内部又分成：

```c
PinMux_init();
SCI_init();
INTERRUPT_init();
```

### PinMux_init

```c
GPIO_MuxConfig(
    SCI_TX_PIN_NUM,
    SERIAL_TX_GPIO_CONFIG(SCI_TX_PIN_NUM),
    GPIO_PIN_TYPE_STD,
    GPIO_QUAL_ASYNC);

GPIO_MuxConfig(
    SCI_RX_PIN_NUM,
    SERIAL_RX_GPIO_CONFIG(SCI_RX_PIN_NUM),
    GPIO_PIN_TYPE_STD,
    GPIO_QUAL_ASYNC);
```

本质是把 `GPIO12` 切换成 `SCIB_TX`，把 `GPIO13` 切换成 `SCIB_RX`。RX 使用 `GPIO_QUAL_ASYNC`，因为串口是异步通信，不适合普通 GPIO 同步滤波。

### SCI_init

核心配置：

```c
SCI_setConfig(SCI_BASE, DEVICE_LSPCLK_FREQ, SCI_BAUD_RATE,
    SCI_CONFIG_WLEN | SCI_CONFIG_STOP | SCI_CONFIG_PAR);
```

它配置 SCI 输入时钟、波特率和 8N1 数据格式。

随后：

```c
SCI_enableFIFO(SCI_BASE);
SCI_resetRxFIFO(SCI_BASE);
SCI_setFIFOInterruptLevel(SCI_BASE, SCI_FIFO_TX0, SCI_FIFO_RX1);
SCI_enableInterrupt(SCI_BASE, SCI_INT_RXFF | SCI_INT_RXERR);
```

含义：打开 FIFO；复位接收 FIFO；RX FIFO 中有 1 个字节就触发中断；打开 RX FIFO 中断和 RX 错误中断。

### INTERRUPT_init

```c
Interrupt_register(INT_SCIB_RX, &scirxISR);
Interrupt_enable(INT_SCIB_RX);
```

含义：当 SCIB 接收到数据触发 RX 中断时，CPU 跳转到 `scirxISR()`。

### scirxISR

中断函数先读中断状态：

```c
sci_int_flag = SCI_getInterruptStatus(SCI_BASE);
```

先检查接收错误：

```c
if ((sci_int_flag & SCI_INT_RXERR) == SCI_INT_RXERR)
{
    rx_err_cnt++;
    SCI_performSoftwareReset(SCI_BASE);
    SCI_clearInterruptStatus(SCI_BASE, SCI_INT_RXERR);
}
```

常见错误包括帧错误、校验错误、溢出错误和 break 检测。

如果 RX FIFO 触发：

```c
if ((sci_int_flag & SCI_INT_RXFF) == SCI_INT_RXFF)
{
    while (SCI_isDataAvailableNonFIFO(SCI_BASE))
    {
        recv_char = SCI_readCharNonBlocking(SCI_BASE);
        recv_data_handle(recv_char);
    }
    SCI_clearInterruptStatus(SCI_BASE, SCI_INT_RXFF);
}
```

逻辑：只要还有数据，就一直读；每读一个字符，交给 `recv_data_handle()`；最后清 RXFF 中断标志。

### recv_data_handle

```c
void recv_data_handle(char recv_char)
{
    recv_frame[recv_cnt++] = recv_char;
    if (recv_char == '\n')
    {
        msg = "  You sent: \0";
        SCI_writeCharArray(SCI_BASE, (const uint8_t *)msg, strlen(msg));
        SCI_writeCharArray(SCI_BASE, (const uint8_t *)recv_frame, recv_cnt);
        msg = (const char *)"\r\nEnter characters: \0";
        SCI_writeCharArray(SCI_BASE, (const uint8_t *)msg, strlen(msg));
        recv_cnt = 0;
    }
}
```

它是一个按行接收的简单状态机：每收到一个字符，就存进 `recv_frame`；如果收到 `\n`，说明一行结束，然后把这一行回显给串口助手。

工程注意点：`recv_frame` 是 256 字节，但示例没有检查 `recv_cnt` 是否越界。真实项目必须加边界保护。

### SCI_sendDataPolling

```c
while (!SCI_isSpaceAvailableNonFIFO(SCI_BASE))
    ;

SCI_writeCharNonBlocking(SCI_BASE, data);
```

含义：等待发送缓冲区有空间，然后写一个字节。这是轮询发送，优点是简单可靠，缺点是 CPU 会等待。

## 3. sci_interrupts_fifo

位置：`examples_core0/examples/sci/sci_interrupts_fifo/main.c`

这个例程演示标准 FIFO 中断回显。

关键配置：

```c
SCI_setConfig(SCIB_BASE, DEVICE_LSPCLK_FREQ, 921600,
    SCI_CONFIG_WLEN_8 | SCI_CONFIG_STOP_ONE | SCI_CONFIG_PAR_NONE);

SCI_setFIFOInterruptLevel(SCIB_BASE, SCI_FIFO_TX0, SCI_FIFO_RX2);
SCI_enableInterrupt(SCIB_BASE, SCI_INT_TXFF | SCI_INT_RXFF);
```

重点：RX FIFO 到 2 个字符才触发中断，TX FIFO 空时触发发送中断。

接收 ISR 一次读两个字符：

```c
receivedChar1 = SCI_readCharBlockingFIFO(SCIB_BASE);
receivedChar2 = SCI_readCharBlockingFIFO(SCIB_BASE);
```

ISR 末尾：

```c
SCI_clearInterruptStatus(SCIB_BASE, SCI_INT_RXFF);
Interrupt_clearACKGroup(INTERRUPT_ACK_GROUP9);
```

SCI 中断属于 PIE Group9，因此中断结束要 ACK Group9，允许下一次中断进入。

## 4. sci_send_polling_recv_interrupt_232 和 485

这两个例程和普通 `sci_send_polling_recv_interrupt` 逻辑类似：发送用 polling，接收用 interrupt。

它们的 `board.h` 配置是：

```c
#define SCI_BASE SCIA_BASE
#define SCI_TX_PIN_NUM 2
#define SCI_RX_PIN_NUM 3
#define SCI_BAUD_RATE 115200
```

也就是使用 `SCIA`，TX 是 `GPIO2`，RX 是 `GPIO3`，波特率 `115200`。

RS232 和 RS485 的主要区别一般在外部硬件接口。RS232 常见点对点全双工，经过 RS232 收发器；RS485 是工业差分总线，半双工常见，通常有 `DE/RE` 方向控制脚。

从当前代码看，两者软件配置差别不大。真正差异可能在开发板外部收发器、接口、跳线或硬件自动方向控制上。初学阶段先跑 SCIB 的 USB 串口，不要先卡在 232/485。

## 5. sci_send_dma_recv_interrupt

这个例程演示 DMA 发送 + FIFO 中断接收。

README 说明：配置 DMA1 用来发送数据；接收仍使用 FIFO 中断；串口发送的数据需要以 `\n` 结尾。

DMA 发送的意义是：CPU 不需要一直等待串口发送缓冲区。CPU 只告诉 DMA 从 `send_frame` 取 `len` 个字节搬到 `SCITXBUF`，之后由 DMA 自动搬运。

适合高速日志输出、周期状态帧和数据采集上传。

## 6. sci_send_recv_dma

位置：`examples_core0/examples/sci/sci_send_recv_dma`

这是 SCI 目录里最进阶的例程，演示 DMA 发送 + DMA 接收。

协议形式：

```text
第 1 个字节：长度
后面的字节：数据内容
```

核心状态机：

```c
case RECV_STATUS_IDLE:
    recv_len            = recv_frame[0];
    DmaCh2Regs.BLOCK_TS = recv_frame[0] - 1;
    DmaCh2Regs.DAR      = (u32)recv_frame;
    recv_status         = RECV_STATUS_RECV_DATA;
    DMA_startChannel(2);
    break;
```

含义：第一次 DMA2 只收 1 个字节，这个字节是后续数据长度，然后重新配置 DMA2 接收 `recv_len` 个字节。

完整数据收完后：

```c
case RECV_STATUS_RECV_DATA:
    DmaCh2Regs.BLOCK_TS = 0;
    DmaCh2Regs.DAR      = (u32)recv_frame;
    DMA_startChannel(2);
    recv_status = RECV_STATUS_IDLE;
    SCI_recvFrameHandle();
    break;
```

随后 `SCI_recvFrameHandle()` 把收到的数据通过 DMA 发回去。

学习重点：DMA 通道配置、SCI TX/RX DMA 触发源、长度字节协议、DMA 完成中断、状态机处理帧。

这个例程先读懂即可，初学阶段不建议马上修改。

## 推荐学习顺序

1. `sci_puts_printf`：目标是串口助手看到 `hello world` 和 `printf` 输出。
2. `sci_send_polling_recv_interrupt`：目标是输入一行，板子回显一行。
3. 把回显改成命令解析，例如 `led on`、`led off`、`status`。
4. `sci_interrupts_fifo`：理解 FIFO 阈值，为什么可以一次收多个字节再中断。
5. `sci_send_dma_recv_interrupt`：理解 DMA 发送，CPU 不用卡着等每个字节发完。
6. `sci_send_recv_dma`：理解完整 DMA 收发协议，适合高吞吐串口通信。

## 当前最适合做的练习

建议改 `sci_send_polling_recv_interrupt`，把“回显字符串”改成“串口控制 LED”。

示例思路：

```c
if (strcmp(recv_frame, "led on\n") == 0)
{
    GPIO_writePin(5, 1);
    GPIO_writePin(9, 1);
}
else if (strcmp(recv_frame, "led off\n") == 0)
{
    GPIO_writePin(5, 0);
    GPIO_writePin(9, 0);
}
```

这就是机器人控制板的最小命令接口：上位机通过串口发命令，板子解析命令，板子控制外设，板子返回状态。

后续可以扩展为：

```text
duty 50
target 1000
mode speed
start
stop
```

## 一句话总结

SCI 目录的学习主线是：从能打印调试信息，到能接收命令，再到能用 FIFO/DMA 高效收发数据。

当前最重要的两个例程是：

```text
sci_puts_printf
sci_send_polling_recv_interrupt
```

掌握它们之后，就可以做“串口控制 LED”和“串口调 PWM 占空比”，这正是控制板调试最基础也最常用的能力。

## 补充：SCIB 接收中断在 PIE 中的位置

这一节只补充 SCI 专属的中断信息；通用中断架构、优先级和 STM32 对比见 [[F280049 中断架构与 STM32 对比]]。

当前基础回显示例使用：

```text
SCIB RX → PIE Group 9 / Channel 3 → CPU INT9 → scirxISR()
```

工程中的中断初始化对应为：

```c
Interrupt_initModule();
Interrupt_initVectorTable();

Interrupt_register(INT_SCIB_RX, &scirxISR);
Interrupt_enable(INT_SCIB_RX);
```

其中 `Interrupt_register()` 将 `scirxISR()` 写入 PIE Group 9 / Channel 3 的向量表项；`Interrupt_enable()` 开启该通道及其上层中断通路。SCI 模块还必须单独使能本地中断源：

```c
SCI_enableInterrupt(SCI_BASE, SCI_INT_RXFF | SCI_INT_RXERR);
```

接收 ISR 的正确收尾顺序为：读空 RX FIFO，清 `SCI_INT_RXFF`，再确认 PIE Group 9：

```c
SCI_clearInterruptStatus(SCI_BASE, SCI_INT_RXFF);
Interrupt_clearACKGroup(INTERRUPT_ACK_GROUP9);
```

`sci_interrupts_fifo` 例程已经展示了上述两个步骤。基础 `sci_send_polling_recv_interrupt` 的 `scirxISR()` 没有 Group 9 ACK，应作为学习和实机调试时的检查项。

注意：当前仓库 `libs/driverlib/interrupt.h` 中的 `Interrupt_clearACKGroup()` 是空实现。标准实现应向 PIE 的 ACK 寄存器写入组掩码；在修复或确认该驱动层前，即使调用该函数也可能没有完成真实 ACK。若出现“只进入第一次 SCI 中断”，先检查此处。
