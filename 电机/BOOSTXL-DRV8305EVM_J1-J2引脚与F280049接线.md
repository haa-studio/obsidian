# BOOSTXL-DRV8305EVM J1/J2 引脚与 F280049 接线

> [!info] 适用范围
> 本笔记整理 BOOSTXL-DRV8305EVM 的 J1/J2 排针定义，并结合当前 `NewSensorFOC2` 工程中的 PWM、ADC、eQEP 配置，给出 F280049 的候选接线关系。
>
> 排针编号视角：看板子丝印面，元件朝上，从上往下数 1~20。J1 为右侧排针，J2 为左侧排针。

## 1. 三套编号必须分开

| 编号类型 | 示例 | 含义 |
|---|---|---|
| J1/J2 物理编号 | J1-5 | 排针上的第 5 个位置 |
| DRV8305 芯片管脚号 | Pin 9 | 芯片封装上的管脚编号 |
| F280049 软件信号 | GPIO0 / ADCA4 | DSP 代码使用的 GPIO 或 ADC 通道 |

例如：DRV8305 芯片 Pin 9 是 `SCS`，但它接到排针的物理位置是 J1-5，不能把 Pin 9 当成 J1-9。

## 2. J1 右侧 20Pin

| J1 引脚 | 信号             | DRV8305 芯片管脚 | 功能                     | F280049 当前工程候选连接                       |
| ----: | -------------- | -----------: | ---------------------- | -------------------------------------- |
|     1 | 3.3V           |            - | 板载 3.3V 供电输出           | 3.3V 电源                                |
|     2 | GND            |            - | 功率/数字地                 | GND                                    |
|     3 | `INH_A`        |            2 | A 相上桥 PWM 输入           | `GPIO0 / EPWM1A`                       |
|     4 | `INL_A`        |            3 | A 相下桥 PWM 输入           | `GPIO1 / EPWM1B`                       |
|     5 | `SCS` / `nSCS` |            9 | SPI 片选，低有效             | SPI GPIO，当前工程未配置                       |
|     6 | `INH_B`        |            4 | B 相上桥 PWM 输入           | `GPIO2 / EPWM2A`                       |
|     7 | `INL_B`        |            5 | B 相下桥 PWM 输入           | `GPIO3 / EPWM2B`                       |
|     8 | `SCLK`         |           12 | SPI 时钟                 | SPI GPIO，当前工程未配置                       |
|     9 | `SDI`          |           10 | SPI MOSI，MCU 到 DRV8305 | SPI GPIO，当前工程未配置                       |
|    10 | `INH_C`        |            6 | C 相上桥 PWM 输入           | `GPIO4 / EPWM3A`                       |
|    11 | `INL_C`        |            7 | C 相下桥 PWM 输入           | `GPIO5 / EPWM3B`                       |
|    12 | `SDO`          |           11 | SPI MISO，DRV8305 到 MCU | SPI GPIO，当前工程未配置                       |
|    13 | `nFAULT`       |            8 | 故障开漏输出，故障时拉低           | 建议接输入 GPIO；工程宏预留 `GPIO35`，需核对实际 PCB    |
|    14 | `PWRGD`        |           13 | 电源正常指示                 | 输入 GPIO，当前工程未使用                        |
|    15 | `ISEN_A`       |           16 | A 相电流采样输出              | `ADCA4 / ADCRESULT0`                   |
|    16 | `ISEN_B`       |           17 | B 相电流采样输出              | `ADCB0 / ADCRESULT0`                   |
|    17 | `ISEN_C`       |           18 | C 相电流采样输出              | `ADCC0 / ADCRESULT0`；当前软件用 Ia、Ib 重构 Ic |
|    18 | `EN_GATE`      |            1 | 栅极驱动总使能，高电平开启          | 必须由 DSP GPIO 拉高；当前代码尚未明确分配             |
|    19 | `WAKE`         |           47 | 芯片唤醒，板上通常已上拉           | 一般无需控制                                 |
|    20 | GND            |            - | 数字地                    | GND                                    |

## 3. J2 左侧 20Pin

| J2 引脚 | 信号          | 功能             | F280049 当前工程候选连接     |
| ----: | ----------- | -------------- | -------------------- |
|     1 | 3.3V        | 参考电压/逻辑电源      | 3.3V                 |
|     2 | GND         | 地              | GND                  |
|     3 | `VSEN_PVDD` | 母线 PVDD 电压分压采样 | `ADCA8 / ADCRESULT2` |
|     4 | `VSEN_A`    | A 相相电压分压采样     | `ADCA5 / ADCRESULT1` |
|     5 | `VSEN_B`    | B 相相电压分压采样     | `ADCB4 / ADCRESULT1` |
|     6 | `VSEN_C`    | C 相相电压分压采样     | `ADCC5 / ADCRESULT1` |
|  7~20 | NC          | 空脚，无连接         | 不接                   |

当前工程在 `Libraries/drivers/src/adc.c` 中的 ADC 配置与上表一致：

```text
ADCA SOC0 = ADCIN4  -> ISEN_A
ADCB SOC0 = ADCIN0  -> ISEN_B
ADCC SOC0 = ADCIN0  -> ISEN_C
ADCA SOC1 = ADCIN5  -> VSEN_A
ADCB SOC1 = ADCIN4  -> VSEN_B
ADCC SOC1 = ADCIN5  -> VSEN_C
ADCA SOC2 = ADCIN8  -> VSEN_PVDD
```

## 4. 当前工程中的 PWM、ADC、编码器关系

### PWM

- `EPWM1A/B`：U 相，对应 J1-3/J1-4
- `EPWM2A/B`：V 相，对应 J1-6/J1-7
- `EPWM3A/B`：W 相，对应 J1-10/J1-11
- PWM 频率：16 kHz
- 中心对齐，上下计数
- 死区：约 1.1 us
- PWM 初始状态由 `Init_EnMotorPwm(FALSE)` 强制关闭

### ADC

- ADC 由 EPWM1 SOCA 触发
- `ADCA SOC2` 完成后触发 `ADCA1` 中断
- ADC 中断中执行 FOC 快环
- 电流采样启动时先进行约 50 ms 偏置校准
- 正常运行时软件主要使用 A、B 两相电流，并通过 `Ic = -(Ia + Ib)` 重构 C 相

### 编码器

当前工程使用 eQEP1：

- A 相：GPIO6
- B 相：GPIO7
- Index：GPIO31

编码器实际线数、1X/2X/4X 解码方式，以及软件中假设的 4000 count/rev 必须根据实际编码器确认。

## 5. 必须特别处理的信号

### `EN_GATE`，J1-18

这是 DRV8305 的总栅极使能。即使六路 PWM 正确，如果 `EN_GATE` 没有拉高，DRV8305 也不会输出栅极驱动。

建议初始化顺序：

```text
1. DSP GPIO 配置为输出
2. 输出低电平
3. 初始化 PWM、ADC、故障输入和控制变量
4. 确认无故障、母线电压正常
5. 再将 EN_GATE 拉高
6. 最后解除 PWM 强制关闭
```

当前 `drive_board_init()` 中 GPIO8/9/10 被用于 BLDC A0/A1/A2 选择，不能直接假定 GPIO8 是 `EN_GATE`。必须根据主控板原理图确认实际 GPIO。

### `nFAULT`，J1-13

- 开漏、故障时拉低；主控端需要上拉或确认板上已有上拉。
- 建议配置为异步输入，并在 PWM 输出前读取。
- 当前工程虽然定义了 `PM_nFAULT_GPIO = 35`，但仍需核对实际连接和初始化代码。

### SPI

J1-5/J1-8/J1-9/J1-12 是 DRV8305 SPI 配置/诊断接口。当前工程没有看到完整 SPI 初始化，因此不要假定 DRV8305 已通过 SPI 完成配置。首次调试时至少读取并记录 DRV8305 状态寄存器。

## 6. 电机和功率端子

- 电机三相：接板上方蓝色端子 `MOTA/MOTB/MOTC`，通常标作 J4。
- 功率母线：接 `PVDD/GND`，通常标作 J3。
- J1/J2 只是控制和采样接口，不承载电机功率。

## 7. 上电前检查清单

- [ ] 已确认 J1/J2 的 1~20 脚方向，没有上下反接
- [ ] F280049 的 GPIO0~5 与 J1 PWM 信号一一对应
- [ ] `EN_GATE` 已分配明确 GPIO，并能被软件拉低/拉高
- [ ] `nFAULT` 已接入 DSP 输入，且有有效上拉
- [ ] ADC 通道与实际排针/PCB 走线一致
- [ ] ADC 满量程、电流增益、母线分压参数与原理图一致
- [ ] 编码器 A/B/Index 接线和方向已确认
- [ ] PWM 初始为关闭状态
- [ ] 已确认过流比较器、PWM Trip Zone 和硬件关断链路有效
- [ ] 已修正或确认软件中的母线电压采样值，不使用测试硬编码
- [ ] 低压限流测试通过后，才接入额定母线

## 8. 与当前 FOC 工程对应的主要文件

| 主题 | 文件 |
|---|---|
| 主程序和 ADC 中断 | `NewSensorFOC2/NewSensorFOC_core0/main_core0.c` |
| FOC 快环、慢环 | `NewSensorFOC2/NewSensorFOC_core0/Motor_Code/FOC/motor_source/motor_control.c` |
| PWM 初始化 | `NewSensorFOC2/NewSensorFOC_core0/Libraries/drivers/src/pwm.c` |
| ADC 通道配置 | `NewSensorFOC2/NewSensorFOC_core0/Libraries/drivers/src/adc.c` |
| PWM 占空比输出 | `NewSensorFOC2/NewSensorFOC_core0/Motor_Code/App/src/motor_pwm.c` |
| eQEP 初始化 | `NewSensorFOC2/NewSensorFOC_core0/Libraries/drivers/src/eqep.c` |
| 电机参数 | `NewSensorFOC2/NewSensorFOC_core0/Motor_Code/FOC/motor_source/user_interface.c` |
| 保护和硬件宏 | `NewSensorFOC2/NewSensorFOC_core0/Motor_Code/App/inc/motor_sys_config_basic.h` |

## 9. 参考资料

- `资料/MDBU003A_SCH.PDF`
- `资料/BOOSTXL-DRV8305EVM Design Files (Rev. A) slvc626a.zip`
- `资料/InstaSPIN-FOC 和InstaSPIN-MOTION中文版.pdf`

> [!warning] 说明
> BOOSTXL-DRV8305EVM 的排针定义和当前主控板实际走线仍应以原理图、PCB 和万用表实测为准。特别是 `EN_GATE`、`nFAULT`、SPI 以及地线分区，不能仅凭软件 GPIO 名称推断。
