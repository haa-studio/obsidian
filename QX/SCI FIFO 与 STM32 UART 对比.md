---
title: "SCI FIFO 与 STM32 UART 对比"
created: "2026-08-08"
tags: ["SCI", "FIFO", "UART", "STM32", "F280049"]
---

# SCI FIFO 与 STM32 UART 对比

## FIFO 是什么

FIFO 是 First In First Out（先进先出）硬件队列。SCI 内部的 FIFO 用于暂存连续到达或等待发出的串口字节。

```text
串口线上的字节
  -> SCI 接收硬件
  -> RX FIFO（硬件队列）
  -> CPU / 接收中断读取
  -> recv_frame[]（软件中拼接命令的数组）
```

`RX FIFO` 和 `recv_frame[]` 完全不同：前者解决“字节到达太快，CPU 暂时来不及读取”，后者解决“如何把多个字节拼成一条命令”。

## 当前 SCI 回显示例的 FIFO 配置

```c
SCI_enableFIFO(SCI_BASE);
SCI_resetRxFIFO(SCI_BASE);
SCI_setFIFOInterruptLevel(SCI_BASE, SCI_FIFO_TX0, SCI_FIFO_RX1);
SCI_enableInterrupt(SCI_BASE, SCI_INT_RXFF | SCI_INT_RXERR);
```

含义：

```text
打开 FIFO
清空接收 FIFO
接收 FIFO 中有 1 个字节时触发中断
允许 FIFO 接收中断和接收错误中断
```

`SCI_FIFO_RX1` 虽然打开了 FIFO，但效果仍接近逐字节中断：

```text
收到 h -> FIFO 有 1 字节 -> 中断 -> CPU 读走 h
```

若设置为 `SCI_FIFO_RX2`，则累计到两个字节才中断：

```text
收到 h -> FIFO 有 1 字节，不中断
收到 e -> FIFO 有 2 字节，进入中断
CPU 一次读走 h 和 e
```

阈值更高可减少中断次数，但会增加每批数据被处理前的等待时间。命令行交互常从 `RX1` 开始；高速连续数据可考虑更高阈值或 DMA。

发送方向同理：CPU 或 DMA 先把数据写入 `TX FIFO`，SCI 硬件再按波特率逐字节发送。

## 与 STM32 UART 的关系

不能说 STM32 一定没有 FIFO，取决于具体芯片系列和 UART 模式。

| 模型                            | 接收中断的典型条件                   | 连续数据的缓冲能力                         |
| ----------------------------- | --------------------------- | --------------------------------- |
| 传统 STM32 USART（许多 F1/F4 常见用法） | `RXNE`，接收数据寄存器非空            | 较小，CPU 必须及时读取，否则可能 `ORE` 溢出       |
| 支持 FIFO 的较新 STM32             | RX FIFO 达到阈值                | 可配置 FIFO 阈值，工作理念接近 SCI FIFO       |
| F280049 SCI FIFO 模式           | `SCI_INT_RXFF`，RX FIFO 达到阈值 | 可按 `SCI_FIFO_RX1`、`RX2` 等阈值批量处理中断 |

传统 STM32 的典型流程：

```text
收到 1 字节 -> RXNE 置位 -> 进入中断 -> CPU 读取 DR/RDR -> RXNE 清除
```

SCI FIFO 模式的典型流程：

```text
连续收到多个字节 -> 存入 RX FIFO -> 达到设定阈值 -> 中断 -> CPU 排空 FIFO
```

无论 STM32 是否有 FIFO，UART 内部都有发送移位寄存器，用于把一个字节逐位推到 TX 引脚。FIFO/数据寄存器的作用是让 CPU 能提前交付更多待发送或待处理的数据。

## 中断处理要点

接收中断中应当：

1. 先检查并处理帧错误、校验错误、溢出等 `SCI_INT_RXERR`。
2. 循环读取 FIFO，直到没有可读字节。
3. 清除 `SCI_INT_RXFF` 标志。
4. 若系统的 PIE 中断流程要求，确认 Group 9 ACK 已正确处理；若只收一次数据就不再进入 ISR，应重点检查这一点。

相关：[[SCI 串口例程学习笔记]]、[[F280049 时钟树]]
