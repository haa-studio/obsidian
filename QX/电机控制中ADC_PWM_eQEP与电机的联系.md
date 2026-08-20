---
type: knowledge-note
topic: motor-control
status: learning
updated: 2026-08-20
tags:
  - 电机控制
  - PMSM
  - FOC
  - ADC
  - PWM
  - eQEP
  - QX049
  - 运动控制
---

# 电机控制中 ADC、PWM、eQEP 与电机的联系

## 一句话理解

在典型 PMSM/伺服电机控制中：

> PWM 负责给电机施加电压，ADC 负责测量电机实际电流，eQEP 负责获取转子位置和速度。

这三个外设不是孤立的知识点，而是同一个实时闭环中的三个关键环节。

~~~mermaid
flowchart LR
    A["目标转速/目标转矩"] --> B["控制算法"]
    B --> C["PWM 占空比"]
    C --> D["栅极驱动器"]
    D --> E["三相逆变器"]
    E --> F["PMSM 电机"]
    F --> G["电流"]
    F --> H["编码器位置"]
    G --> I["ADC"]
    H --> J["eQEP"]
    I --> B
    J --> B
    I --> K["过流保护"]
    K --> C
~~~

## 1. PWM 如何连接到电机

MCU 的 PWM 引脚不能直接驱动大功率电机。真实的电机驱动链路通常是：

~~~text
MCU ePWM
    ↓
栅极驱动器
    ↓
MOSFET / IGBT 三相逆变桥
    ↓
电机 U/V/W 三相绕组
~~~

PWM 不是直接输出平滑模拟电压，而是高速开关功率管。改变占空比后，电机绕组的电感对开关电压进行滤波，电机感受到与占空比相关的平均电压。

| PWM 参数 | 对电机控制的影响 |
|---|---|
| 占空比 | 改变相电压平均值 |
| 三相占空比关系 | 决定空间电压矢量方向和大小 |
| PWM 频率 | 影响电流纹波、开关损耗、噪声和控制周期 |
| 三相同步关系 | 影响旋转磁场时序 |
| 死区时间 | 防止同一桥臂上下管同时导通 |
| Trip Zone | 故障时快速将 PWM 置为安全状态 |

必须区分：

> ePWM 输出的是控制信号，不是直接供给电机绕组的功率。

QX049 能输出 ePWM，不代表已经完成电机驱动。真实系统还需要栅极驱动器、功率管、母线电容、电流采样电路、保护电路和电机功率级。

## 2. ADC 如何连接到电机

ADC 在电机控制中的主要用途通常是测量：

- 电机相电流；
- 直流母线电流；
- 母线电压；
- 功率管温度；
- 电机或驱动器温度。

常见电流采样链路：

~~~text
电机相电流
    ↓
采样电阻 Shunt
    ↓
运算放大器
    ↓
ADC 输入
    ↓
ADC RESULT
    ↓
FOC 电流环
~~~

例如测量两相电流 Ia、Ib，第三相可以利用三相电流和为零近似计算：

~~~text
Ic = -(Ia + Ib)
~~~

ADC 采集到的电流会进入 FOC：

~~~text
三相电流
    ↓
Clarke 变换
    ↓
αβ 静止坐标系
    ↓
Park 变换
    ↓
d/q 旋转坐标系
    ↓
Id、Iq
~~~

- Id 主要对应励磁方向；
- Iq 主要对应转矩方向；
- 对表贴式 PMSM，常见目标是 Id_ref 约等于 0；
- 增大 Iq，通常会增大电磁转矩。

可以这样理解：

> ADC 测得的电流告诉控制器，电机当前实际产生了多大的电磁作用。

PWM 表示“我想施加多少电压”，ADC 反馈“实际电流变成了多少”。没有 ADC，控制器无法可靠地构成电流闭环。

## 3. 为什么 ADC 要由 PWM 触发

逆变器开关会造成电压尖峰、电流纹波和电磁干扰。如果 ADC 在任意时间采样，可能采到开关瞬间的异常值。

实际系统通常采用：

~~~text
ePWM 计数到指定位置
    ↓
产生 SOCA / SOCB
    ↓
ADC 开始采样
    ↓
转换完成
    ↓
ADCINT
    ↓
执行电流环控制
~~~

你的 adc_ex2_soc_epwm 例程对应：

~~~text
ePWM1 SOCA
    ↓
ADCA SOC0
    ↓
ADC RESULT0
    ↓
ADCINT1
    ↓
ADC ISR
~~~

采样点通常安排在 PWM 周期中相对稳定的位置，例如 PWM 中点，或避开功率管刚刚开关和采样电路尚未稳定的时间。

- TBPRD 影响 PWM 周期和 ADC 触发频率；
- CMPA 或其他比较事件影响采样点位置；
- 采样点影响电流反馈质量；
- 电流反馈质量影响 FOC 电流环稳定性。

这就是 PWM 同步 ADC 在电机控制中的真实意义。

## 3.1 QX049 `adc_ex10_multiple_soc_epwm` 源码逐行理解

对应文件：

`C:\Users\OSS\Desktop\f280049revb_evb_examples-master\examples_core0\examples\adc\adc_ex10_multiple_soc_epwm\board.c`

### A. `SYSCLK`、`ADCCLK` 和采样窗口不是同一个概念

这个例程的设备初始化在 `device.h` 中将系统时钟配置为：

```text
DEVICE_SYSCLK_FREQ = 100 MHz
```

`SYSCLK` 是芯片的系统主时钟。它可以作为很多外设的时钟来源，但“外设使用的时钟”不一定等于 SYSCLK。ADC 内部有自己的时钟预分频器：

```c
ADC_setPrescaler(myADC0_BASE, ADC_CLK_DIV_2_0);
ADC_setPrescaler(myADC1_BASE, ADC_CLK_DIV_2_0);
```

这里的含义是：ADC 内核时钟 `ADCCLK` 由 ADC 输入时钟经过 `/2` 得到。若当前 ADC 输入时钟就是 100 MHz 的 SYSCLK，则本例中：

```text
ADCCLK = 100 MHz / 2 = 50 MHz
```

因此应该记成：

```text
SYSCLK：系统时钟，当前约 100 MHz
ADCCLK：ADC 内核转换时钟，由 ADC 预分频器得到，当前约 50 MHz
```

但 `ADC_setupSOC(..., 8U)` 的最后一个参数，例程注释写的是“8 SYSCLK cycles”，它表示采样保持窗口的配置单位是 SYSCLK 周期，不是说 ADC 内核没有分频，也不是 8 个 ADCCLK 周期。要判断具体转换耗时，还要结合数据手册规定的采样窗口、转换周期和 ADC 内核时钟。

### B. 为什么 `ADC_INT_NUMBER1` 的中断源是 `ADC_SOC_NUMBER2`

源码中的三行配置是：

```c
ADC_setupSOC(myADC0_BASE, ADC_SOC_NUMBER0, ADC_TRIGGER_EPWM1_SOCA,
             ADC_CH_ADCIN0, 8U);
ADC_setupSOC(myADC0_BASE, ADC_SOC_NUMBER1, ADC_TRIGGER_EPWM1_SOCA,
             ADC_CH_ADCIN1, 8U);
ADC_setupSOC(myADC0_BASE, ADC_SOC_NUMBER2, ADC_TRIGGER_EPWM1_SOCA,
             ADC_CH_ADCIN2, 8U);
```

一次 `EPWM1 SOCA` 到来时，SOC0、SOC1、SOC2 都收到触发请求，分别采样 ADCIN0、ADCIN1、ADCIN2。SOC 编号从 0 开始，所以 `SOC2` 是本批次的第 3 个采样槽位。

随后：

```c
ADC_setInterruptSource(myADC0_BASE, ADC_INT_NUMBER1, ADC_SOC_NUMBER2);
```

这句话不是“让 INT1 变成 2”，而是建立一条映射：

```text
SOC2 转换完成（EOC） -> ADC 内部中断线 INT1 置位
```

两个编号属于不同对象：

| 名称 | 对象 | 本例含义 |
|---|---|---|
| `ADC_SOC_NUMBER2` | ADC 的 SOC 槽位编号 | 第 3 个转换，采样 ADCIN2 |
| `ADC_INT_NUMBER1` | ADC 模块内部中断线编号 | ADCINT1 |
| `INT_ADCA1` | PIE/CPU 层面的中断入口 | ADCA 的 ADCINT1 ISR 向量 |

选择 SOC2 作为 INT1 的源，主要是为了让前面三个通道都转换完成后再进入一次 ISR。ISR 中才可以成批读取：

```c
adcAResult0 = ADC_readResult(..., ADC_SOC_NUMBER0);
adcAResult1 = ADC_readResult(..., ADC_SOC_NUMBER1);
adcAResult2 = ADC_readResult(..., ADC_SOC_NUMBER2);
```

如果把中断源改成 SOC0，ISR 可能在 SOC1、SOC2 还没有完成时就进入，读取到的后两个结果可能还是上一批数据。

### C. “触发转换”和“触发中断”是什么关系

要把它们看成两条不同的链：

```text
转换启动链：EPWM1 SOCA -> SOC0/SOC1/SOC2 -> ADC 开始采样转换 -> EOC

中断通知链：SOC2 的 EOC -> ADCINT1 标志 -> ADC 模块中断使能
           -> PIE/CPU 的 INT_ADCA1 -> adcA1ISR()
```

源码中的：

```c
ADC_setupSOC(..., ADC_TRIGGER_EPWM1_SOCA, ...);
```

决定“什么事件启动转换”。它依赖 ePWM 的 SOCA 已配置并通过：

```c
EPWM_enableADCTrigger(EPWM1_BASE, EPWM_SOC_A);
```

开启。

源码中的：

```c
ADC_setInterruptSource(myADC0_BASE, ADC_INT_NUMBER1, ADC_SOC_NUMBER2);
ADC_enableInterrupt(myADC0_BASE, ADC_INT_NUMBER1);
```

决定“转换完成后是否向中断控制器发通知，以及由哪个 SOC 的完成事件通知”。主函数中的：

```c
Interrupt_enable(INT_ADCA1);
```

则是继续打开 PIE/CPU 这一层的通路。三层都打开，CPU 才会真正跳到 `adcA1ISR()`。

### D. 不开中断、不开触发，分别会发生什么

| 配置状态 | ADC 是否转换 | 结果寄存器是否更新 | 是否进入 ISR |
|---|---:|---:|---:|
| ePWM SOCA 开启，ADCINT 开启 | 是 | 是 | 是 |
| ePWM SOCA 开启，ADCINT 未开启 | 是 | 是 | 否；可轮询 EOC 或直接读结果 |
| ePWM SOCA 未开启，ADCINT 开启 | 否 | 不产生新结果 | 否；没有新的 EOC 就没有新中断 |
| 两者都未开启 | 否 | 不产生新结果 | 否 |

所以：

> ADC 中断不会主动启动一次 ADC 转换；它只是转换完成后的“通知”。真正启动转换的是 SOC 的触发源。

### E. `ADC_INT_SOC_TRIGGER_NONE` 容易误解

每个 SOC 后面的：

```c
ADC_setInterruptSOCTrigger(myADC0_BASE, ADC_SOC_NUMBER2,
                           ADC_INT_SOC_TRIGGER_NONE);
```

控制的是反向路径“ADC 中断是否反过来触发某个 SOC”。设为 `NONE` 表示不使用 ADCINT1/2 去再次启动 SOC。它不等于“关闭 ADCINT1”，也不影响下面这句将 SOC2 的 EOC 映射给 ADCINT1：

```c
ADC_setInterruptSource(myADC0_BASE, ADC_INT_NUMBER1, ADC_SOC_NUMBER2);
```

本例采用的是单向链路：

```text
ePWM SOCA -> SOC -> EOC -> ADCINT1 -> ISR
```

而不是：

```text
ADCINT1 -> 再触发下一个 SOC
```

### F. 为什么要清中断标志

本例关闭了 ADCINT 连续模式：

```c
ADC_disableContinuousMode(myADC0_BASE, ADC_INT_NUMBER1);
```

因此 ISR 中必须执行：

```c
ADC_clearInterruptStatus(ADCA_BASE, ADC_INT_NUMBER1);
```

否则中断标志可能一直保持置位，下一批转换不能按预期再次产生正常中断；如果新的 EOC 在旧标志未清除时到来，还可能出现 ADCINT overflow。最后还要：

```c
Interrupt_clearACKGroup(INTERRUPT_ACK_GROUP1);
```

释放 PIE 第 1 组，使 CPU 可以继续响应下一次 ADCA1 中断。

## 3.2 读 ADC 例程的固定顺序

以后阅读任意 ADC 例程，按以下顺序查：

1. `Device_init()` / `device.h`：确认 `SYSCLK`。
2. `ADC_setPrescaler()`：确认 `ADCCLK` 分频。
3. `ADC_setupSOC()`：确认 SOC 编号、触发源、通道、采样窗口。
4. ePWM 配置：确认 SOCA/SOCB 在什么计数事件产生、是否真正 enable。
5. `ADC_setInterruptSource()`：确认哪个 SOC 的 EOC 负责 ADCINT1/2/3/4。
6. `ADC_enableInterrupt()` 和 `Interrupt_enable()`：确认 ADC 模块、PIE/CPU 两级中断是否都打开。
7. ISR：确认读取哪些 RESULT、清哪些标志、是否处理 overflow。

本例可以压缩成一句话：

> 100 MHz SYSCLK 经 ADC `/2` 得到约 50 MHz ADCCLK；ePWM1 的 SOCA 同时启动 ADCA 的 SOC0/1/2；SOC2 最后完成后置位 ADCINT1；ADCINT1 再经 PIE/CPU 调用 ISR，ISR 批量读取三个结果并清标志。

## 4. eQEP 如何连接到电机

eQEP 不负责输出功率，也不负责测电流。它负责获取转子位置和速度。

增量式编码器通常输出：

~~~text
A 相
B 相
Index / Z 相
~~~

连接关系：

~~~text
编码器 A → eQEP A
编码器 B → eQEP B
编码器 Z → eQEP Index
~~~

A、B 两相信号相差约 90 度。根据哪一相先变化，可以判断旋转方向。

eQEP 可以提供：

- 位置计数 QPOSCNT；
- 机械角度；
- 旋转方向；
- 机械速度；
- Index 基准位置。

你的 eqep_ex2_pos_speed 例程使用 ePWM 模拟 A/B 信号，再通过跳线送回 eQEP。它先验证：

~~~text
编码器信号 → eQEP → 位置/速度数据
~~~

## 5. 机械角和电角度

FOC 不仅需要机械角，还需要电角度：

~~~text
θe = p × θm + θoffset
~~~

- θm：机械角；
- p：电机极对数；
- θe：FOC 使用的电角度；
- θoffset：编码器零位与电机磁链零位之间的偏移。

如果电角度不正确，可能出现电流分量混乱、电机抖动、电流增大但转矩不明显、电机发热、转向错误或电流环不稳定。

因此，eQEP 是 FOC 的转子位置感知器。

## 6. 一个 FOC 控制周期的完整时序

以 20 kHz 电流环为例：

~~~text
① ePWM 产生三相 PWM
        ↓
② PWM 在合适时刻触发 ADC
        ↓
③ ADC 采集 Ia、Ib、Vbus
        ↓
④ ADC 完成，触发 ADC ISR
        ↓
⑤ 读取 eQEP，得到转子位置和速度
        ↓
⑥ Clarke 变换
        ↓
⑦ Park 变换
        ↓
⑧ 得到 Id、Iq
        ↓
⑨ Id/Iq 与目标值比较
        ↓
⑩ PI 电流环计算 Vd、Vq
        ↓
⑪ 反 Park 变换
        ↓
⑫ SVPWM 计算三相占空比
        ↓
⑬ 更新 ePWM CMPA/CMPB
        ↓
⑭ 下一周期生效
~~~

核心关系：

> ePWM 产生动作，ADC 测量结果，eQEP 提供位置基准，控制算法根据反馈计算下一周期 PWM。

PWM 的 CMPA/CMPB 通常使用 Shadow Load，在合适的计数事件统一装载，避免占空比在周期中间突然改变。

## 7. 速度环、电流环和 1 ms Timer

典型伺服控制采用多环结构：

~~~text
位置环：较低频率
    ↓
速度环：中等频率
    ↓
电流环：较高频率
    ↓
PWM / 逆变器
~~~

频率只是常见量级示意：

- 电流环：10 kHz～20 kHz；
- 速度环：几百 Hz～1 kHz；
- 位置环：几十 Hz～几百 Hz。

你的 timer_1ms 例程可以对应速度环、状态机、通信上报、参数保存和健康监测等低频任务。ADC ISR 更适合执行高速电流控制。

## 8. PPB、越限和 Trip Zone

软件判断过流的路径是：

~~~text
ADC 完成
    ↓
进入 ISR
    ↓
执行判断
    ↓
软件关闭 PWM
~~~

更快的保护路径是：

~~~text
ADC / PPB 越限
    ↓
硬件事件
    ↓
Trip Zone
    ↓
PWM 立即进入安全状态
~~~

对应例程：

- adc_ex7_ppb_offset：ADC 或电流采样零点校准；
- adc_ex8_ppb_limits：ADC 越限检测；
- epwm_ex1_trip_zone：故障时 PWM 快速关断。

这三者组合起来，就是伺服驱动器的电流校准、异常监测和硬件保护基础。

## 9. QX049 例程与真实电机系统的对应关系

| QX049 例程 | 真实系统对应部分 | 当前缺少 |
|---|---|---|
| adc_ex2_soc_epwm | PWM 同步采集相电流 | 采样电阻、运放、功率级 |
| adc_ex10_multiple_soc_epwm | 同步采集多相电流和母线电压 | 真实电机电流源 |
| adc_ex13_soc_oversampling | 降低采样噪声 | 真实功率开关噪声 |
| adc_ex7_ppb_offset | 电流采样零点校准 | 实际电流传感器 |
| adc_ex8_ppb_limits | 过流检测 | 真实过流事件 |
| epwm_ex3_synchronization | 三相 PWM 同步 | 三相逆变桥 |
| epwm_ex8_deadband | 上下桥臂防直通 | 栅极驱动器和功率管 |
| epwm_ex1_trip_zone | 过流、欠压或故障关断 | 真实硬件故障信号 |
| eqep_ex2_pos_speed | 编码器位置和速度反馈 | 真实电机编码器 |
| sci 例程 | 上位机调参与状态监控 | 完整驱动器参数 |
| timer_1ms | 速度环、状态机、故障管理 | 完整控制任务 |
| dma_output_spwm_by_load_mode | 自动更新调制波 | 不能代替 FOC |

## 10. 你当前 QX049 项目的准确定位

项目名称建议：

> QX049 基于 PWM 同步采样的运动控制外设验证平台

可以验证：

- ePWM 频率、占空比和同步；
- ADC 触发、采样和转换时序；
- 多通道采样和过采样；
- PPB 偏移校准和越限检测；
- ePWM 死区和 Trip Zone；
- eQEP 位置、方向和速度读取；
- SCI 参数配置和状态监控；
- 1 ms 任务调度；
- 控制算法映射到实时中断骨架的时序。

暂时不能证明：

- 真实三相逆变器驱动；
- 真实电机启动和运行；
- 实物电流闭环；
- 实物位置闭环；
- 完整伺服执行器性能。

缺少的硬件主要包括电机、三相逆变器、栅极驱动器、电流采样、编码器、直流母线和功率保护电路。

## 11. 推荐学习顺序

### 第一阶段：外设时序

1. epwm_ex2：PWM 频率、占空比和计数模式。
2. adc_ex2：PWM 在固定时刻触发 ADC。
3. adc_ex10：多通道同步采样。
4. adc_ex13：采样噪声和过采样代价。

### 第二阶段：保护

5. adc_ex7：电流传感器零点校准。
6. adc_ex8：ADC 越限。
7. epwm_ex1：故障触发 PWM 关断。
8. epwm_ex8：上下桥臂死区。

### 第三阶段：位置反馈

9. eqep_ex2：A/B 正交编码器、方向、位置和速度。
10. 推导机械角到电角度的转换。
11. 在 Simulink 中加入角度、速度和电流反馈。

### 第四阶段：完整控制链

12. 完成 Simulink PMSM FOC。
13. 画出 QX049 的 ADC ISR 控制骨架。
14. 用固定测试数据替代真实相电流，验证控制计算周期。
15. 将保护路径映射为 PPB/Trip Zone。

## 12. 每个实验必须回答的问题

| 问题 | 应记录内容 |
|---|---|
| PWM 在什么时候触发 ADC？ | SOCA/SOCB 来源和计数事件 |
| ADC 采到了什么？ | 通道、采样窗口、结果寄存器 |
| 采样是否稳定？ | 均值、标准差、最大值、最小值 |
| 采样时刻是否合适？ | 与 PWM 波形的相对位置 |
| 出现异常如何处理？ | PPB、ADCINT、Trip Zone 或软件保护 |
| eQEP 得到了什么？ | 位置计数、方向、机械速度、电角度 |
| 对电机驱动器有什么意义？ | 电流采样、位置反馈、调制或保护 |

## 13. 最终记忆版本

> PWM 决定我想给电机施加什么电压，ADC 测量电机实际产生的电流，eQEP 告诉我转子当前的位置；控制算法利用 ADC 和 eQEP 的反馈，计算下一周期应该输出什么 PWM。

加上安全保护：

> ADC/PPB 发现异常电流，Trip Zone 立即关闭 PWM，避免功率级和电机损坏。

最终链路：

~~~text
电机电流 → ADC → Clarke/Park → PI → SVPWM → PWM → 逆变器 → 电机
                         ↑                         ↓
                   eQEP 位置 ← 编码器 ← 电机转子
~~~

## 14. 关联笔记

- [[个人信息总表]]
- [[QX049官方历程驱动的三个月执行计划_v2]]

## 15. 下一步实验

- [ ] 跑通 adc_ex2_soc_epwm 原版
- [ ] 记录 PWM 频率、ADC 通道、SOC、ADCINT 和 ISR
- [ ] 只修改 TBPRD，测量 ADC 触发频率变化
- [ ] 只修改 CMPA，观察采样点变化
- [ ] 把实验结果写入单独的 EXP-001 页面
