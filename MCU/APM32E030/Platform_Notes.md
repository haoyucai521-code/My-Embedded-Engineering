# APM32E030 (Cortex-M0+) 平台开发笔记

> 文档日期：2026-07-20

## 平台速查

| 关键信息 | 说明 |
|----------|------|
| 芯片 | APM32E030 (Geehy) |
| 内核 | Cortex-M0+ (ARMv6-M) |
| NVIC 优先级位宽 | **2 bit (4 级: 0/1/2/3)**，不可扩展 |
| 同级中断嵌套 | **不支持** (ARMv6-M 硬限制) |
| PRIGROUP 寄存器 | **不存在** (M3/M4 才有) |
| 中断咬尾 | 支持，但无法解决同级阻塞 |
| bit-bang UART | 软件模拟串口，TMR3 驱动，时序要求 ±5µs 以内 |
| RTOS | ZOS |
| 工具链 | IAR EWARM |

---

## Cortex-M0+ NVIC 硬限制

### 同级优先级不嵌套

Cortex-M0+ 只支持 2 bit 优先级，共 4 级。**没有** `AIRCR.PRIGROUP` 寄存器，无法像 M3/M4 那样拆分抢占/响应优先级。

**两个同为优先级 N 的中断无法相互抢占。** 如果 ISR_A 正在执行时 ISR_B 触发，ISR_B 被挂起等待 ISR_A 结束。

### 这意味着什么

如果有两个定时器 ISR：
- TMR3（bit-bang UART，bit 周期 105µs，时序敏感）
- TMR14（LED 扫描，ISR 执行 50-150µs）

都设为优先级 1 → TMR14 ISR 执行时 TMR3 被阻塞 → bit 宽度畸变 → 乱码。

**正确做法**：时序敏感的 ISR 设更高抢占优先级（数值更小），让它能抢占耗时长的 ISR。

---

## bit-bang 模拟串口

### 原理

- 驱动引脚：PA14
- 波特率：9600bps，bit 周期 ≈ 104µs
- 定时器：TMR3，`intervalUs = 1000000/9600 + 1 ≈ 105µs`
- 发送 1 字节需要 10 次 TMR3 中断（10 × 105µs = 1.05ms）
- ISR 每次执行约 5-10µs

### 时序要求

UART 接收端在 bit 中心采样，容限 **±2-5%（约 ±5µs）**。TMR3 ISR 被延迟 30µs 即可导致误码。任何一个 bit 错误都会破坏整个字节，起始位错误导致帧同步丢失，产生连续乱码。

### 结论

**bit-bang UART 的中断优先级必须高于所有可能长时间阻塞 CPU 的 ISR。**

---

## 中断优先级最佳实践

以某项目实际配置为例：

| 中断源 | IPR 值 | 逻辑优先级 | 说明 |
|--------|--------|-----------|------|
| WWDT | `0x00` | 0 (最高) | 系统异常恢复 |
| SysTick | `0x00` | 0 | 系统时基 1ms |
| TMR3 (模拟串口) | `0x40` | **1** | bit-bang 时序敏感 |
| TMR14 (LED 扫描) | `0x80` | **2** | 允许 TMR3 抢占 |
| USART1 | `0x80` | 2 | 硬件 UART |

> APM32E030 IPR 寄存器：逻辑值 0/1/2/3 映射到 0x00/0x40/0x80/0xC0（位移 6 bit）。

---

## lib-zos HAL 行为

### 定时器优先级

`ZOSHalHardTimer.c` 中所有定时器默认统一优先级。应用层如需调整，在 `ZOSHardTimerStart` 之后直接调 CMSIS API 覆盖：

```c
ZOSHardTimerOpen(ZOS_HAL_HARD_TIMER_14, &timerConf);
ZOSHardTimerStart(ZOS_HAL_HARD_TIMER_14);
NVIC_SetPriority(TMR14_IRQn, 2);  // 覆盖 HAL 层默认优先级
```

放在 `ZOSHardTimerStart` 之后，确保不被 HAL 覆盖（`ZOSHalHardTimerStart` 不操作 NVIC）。

### 硬件 UART 发送是轮询阻塞的

```c
for (i = 0; i < len; i++) {
    USART_TxData(uartDef->lowUart, data[i]);
    while (!USART_ReadStatusFlag(uartDef->lowUart, USART_FLAG_TXBE));
}
```

115200bps 下每字节阻塞 ~87µs。长帧连续阻塞可达 ms 级。同级 ISR 在此期间无法抢占，需注意。

---

## 常见坑

| 问题 | 触发条件 | 规避方案 |
|------|----------|----------|
| 同级 ISR 互锁导致时序畸变 | 两个同级定时器同时活跃 | 时序敏感的 ISR 设更高抢占优先级 |
| bit-bang UART 乱码 | TMR3 ISR 被其他 ISR 阻塞 >30µs | 确保 TMR3 优先级高于其他定时器 |
| `NVIC_SetPriority` 被 HAL 覆盖 | 在 `ZOSHardTimerStart` 之前调用 | 放在 `Start` 之后 |
| 轮询 UART 阻塞 ISR | 长帧发送期间同级 ISR 无法响应 | 控制帧长，或调低 UART 优先级 |
| IRK 重启后丢失 | Bond 信息保存不完整 | 确保 `irk.id_info.irk` 被完整持久化 |

---

## 待补充

- [ ] 芯片具体型号与封装
- [ ] 主频与时钟树
- [ ] Flash/RAM 布局
- [ ] 启动流程
- [ ] 低功耗模式
- [ ] 烧录与调试工具链
- [ ] 其他外设使用情况

---

## 相关记录

- [中断优先级导致的 bit-bang 串口乱码调试](../../Debug/Hardware/ISR_Priority_Log_Garbled.md)
- [OM6626 平台开发笔记](../OM6626/Platform_Notes.md)
