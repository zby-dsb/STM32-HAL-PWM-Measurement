# STM32-HAL-PWM-Measurement

> **库类型：STM32 HAL 库（硬件抽象层）** —— 与此前仓库中基于标准外设库（SPL / Standard Peripheral Library）的工程相互独立、互不通用。本仓库属于「HAL 库」系列。

基于 STM32CubeMX + HAL 库的 **PWM 参数测量**实验：用一个定时器产生 PWM 信号，另一个定时器工作在**输入捕获（PWM 输入模式）**测量其周期、脉宽与占空比，结果通过串口打印。

## 硬件与引脚

| 外设 | 引脚 | 方向 | 说明 |
|---|---|---|---|
| TIM3_CH1 | PA6 | 输出 | 产生被测 PWM 信号 |
| TIM1_CH1 / CH2 | PA8 | 输入 | 输入捕获（PWM 输入模式，同一 TI1 引脚进两路） |
| USART1_TX / RX | PA9 / PA10 | 输出 | 串口打印测量结果（115200 等波特率由 CubeMX 配置） |
| 调试 | PA13 / PA14 | — | SWD（Serial Wire） |

芯片：**STM32F103xB**（如 STM32F103C8T6「蓝 Pill」）。

> 接线：把 **PA6 与 PA8 用杜邦线短接**，即可用本板自产的 PWM 做测量对象，无需外部信号源。

## 定时器参数

| 定时器 | 角色 | 预分频 | 周期 / 脉冲 | 结果 |
|---|---|---|---|---|
| TIM3 | PWM 输出（被测） | 7 | Period=999，Pulse=200 | 8 MHz / 8 / 1000 = **1 kHz**，占空比 **20 %** |
| TIM1 | 输入捕获（测量） | 7 | — | 计数时钟 1 MHz → **1 µs 分辨率** |

TIM1 通道配置：
- `CH1`：捕获**上升沿**（`TIM_INPUTCHANNELPOLARITY_RISING`）
- `CH2`：捕获**下降沿**（`TIM_INPUTCHANNELPOLARITY_FALLING`），间接映射到 TI1
- 即 CubeMX 里的 **PWM Input mode**：一个引脚进两路，硬件自动复位计数并锁存周期与脉宽。

## 程序流程

1. `HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1)` 启动被测 PWM 输出。
2. 每次测量循环：
   1. `__HAL_TIM_CLEAR_FLAG(&htim1, TIM_FLAG_CC1)` 清除捕获标志；
   2. `HAL_TIM_IC_Start()` 分别启动 CH1、CH2 输入捕获；
   3. 等待第一次 `CC1` 标志（第一个上升沿，作为周期起点）后再次清除；
   4. 等待第二次 `CC1` 标志（下一个上升沿，即一个完整周期结束）；
   5. `HAL_TIM_IC_Stop()` 停止捕获，避免数据被下一次覆盖；
   6. 读取 `CCR1`（周期计数）与 `CCR2`（高电平脉宽计数），按 1 µs 换算：
      ```c
      float period     = ccr1 * 1e-6f;      // 秒
      float pulseWidth = ccr2 * 1e-6f;      // 秒
      float duty       = pulseWidth / period;
      UART_Printf("脉宽= %.1fus,周期 = %.1fus,占空比 = %.1f%%", ...);
      ```
   7. `HAL_Delay(1000)` 每秒测量并打印一次。

预期输出（PA6 接 PA8、TIM3 输出 1 kHz / 20 % 时）：

```
脉宽= 200.0us,周期 = 1000.0us,占空比 = 20.0%
```

## 自定义串口打印

工程内实现了轻量 `UART_Printf()`（`Core/Src/main.c`），用 `vsprintf` 格式化后调用 `HAL_UART_Transmit()` 发送，避免引入完整 `printf` 重定向与半主机（semihosting）依赖。

## 编译与烧录

本工程使用 CMake + arm-none-eabi-gcc（与同系列 HAL 仓库同一套工具链）：

```bash
cmake --preset <你的预设名>      # 见 CMakePresets.json
cmake --build build
# 用 ST-Link / DAPLink 烧录 build/*.elf / *.bin
```

## 学习要点（HAL vs 标准外设库）

- **PWM 输入模式是「一脚两路」**：同一个 TI1 引脚同时进 CH1（上升沿）和 CH2（下降沿），硬件在上升沿自动复位计数器，所以 `CCR1` 直接就是周期、`CCR2` 直接就是高电平时间——不需要像普通输入捕获那样手动存两次边沿再相减。
- **捕获通道要成对 Start/Stop**：HAL 里 `HAL_TIM_IC_Start()` 对 CH1、CH2 各调一次；本例每次测量后 `HAL_TIM_IC_Stop()`，保证读到的是完整、未被覆盖的一帧数据。
- **等标志位用宏**：`__HAL_TIM_GET_FLAG()` / `__HAL_TIM_CLEAR_FLAG()` 是 HAL 统一的标志操作宏，对应标准外设库的 `TIM_GetFlagStatus()` / `TIM_ClearFlag()`。
- **注意溢出**：TIM1 是 16 位，1 µs 分辨率下最长约 65 ms；测更低频率的 PWM 需要加大预分频或做溢出计数扩展。

## 目录结构

```
Core/       用户代码（main.c、gpio、tim、usart、中断服务 stm32f1xx_it.c 等）
Drivers/    HAL 库与 CMSIS（由 CubeMX 生成，随工程提交）
```

> 编译产物 `build/` 已写入 `.gitignore`，不会进仓库。
