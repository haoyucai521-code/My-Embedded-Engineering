# 中断优先级导致的 bit-bang 串口乱码

> 文档日期：2026-07-20

## 环境

- **芯片**：APM32E030 (Geehy, Cortex-M0+ 内核)
- **日志方式**：软件模拟串口 (bit-banged UART) @ PA14, 9600bps, TMR3 驱动
- **RTOS**：ZOS
- **工具链**：IAR EWARM
- **涉及工程**：某项目显示板固件

---

## 现象

运行中偶尔跑飞（系统异常），UART 日志输出严重乱码：

```
[14:15:09.102] ======OOOOb'\xba\x151\r=5\x15\x81TO ZOS ============
[14:15:09.197] bype ordeb'r  = LE\r\nint size   \x90O'b'\xa2j
[14:15:09.291] <b'000000.008> Ii\xbd\xb9troL: LMD TMR OK\n
[14:15:09.384] <000000b'.012> I/ZOSCore\x9d'b'\x9a\xb1eep\x00open mode=2 timeout=5000
```

单字节错误 → 字符错位 → 整行不可读，典型的串口 bit 时序问题。

---

## 排查过程

### 1. 确认日志路径

追溯 `putchar → ZOSSimUartWrite`，确认**日志走的是软件 bit-banged 模拟串口，不是硬件 UART**。

```
ZOSHalPrintf → printf → putchar → ZOSSimUartWrite (PA14, TMR3 驱动)
```

### 2. 分析模拟串口

发送 1 字节需要 10 次 TMR3 中断（10 × 105µs = 1.05ms）。TMR3 ISR 每次执行约 5-10µs。

### 3. 找冲突源

LED 扫描也用了硬件定时器（TMR14, 250Hz）。ISR 执行时间 50-150µs，含大量 GPIO 操作和关全局中断的临界区。

### 4. 定位根因：中断优先级

代码中所有定时器统一设为优先级 1。但 **Cortex-M0+ NVIC 同级优先级不嵌套**——TMR14 ISR 执行时 TMR3 被挂起等待。TMR14 ISR 执行 50-150µs，而 TMR3 的 bit 周期只有 105µs，**LED ISR 轻松吃掉 1-1.5 个 bit 周期**。

### 5. 时序抖动如何导致乱码

```
正常 9600bps 时序:  每个bit 104µs
接收端UART采样点:   每个bit中心 (~52/156/260...µs)

抖动场景:
TMR3 应该在 t=0/105/210/315... 触发
TMR14 在 t=3900 触发, ISR 执行 120µs
TMR3 在 t=3990 的触发被延迟到 t=4020
→ 该 bit 宽度从 105µs 变成 135µs (偏差 28.6%)

接收端采样窗口 ~52µs，28.6% 抖动远超 2-5% 容限
→ 采样到错误 bit → 字节错误 → 乱码
```

### 6. 跑飞的原因

两个同级 ISR 轮流占据 CPU，`LedScanTimerCallback` 内 `ZOSHalInterruptEnable(false)` 关全局中断，连 SysTick 都屏蔽。主循环得不到执行 → 业务逻辑停滞 → 看门狗饿死 → 跑飞。

---

## 根因

**Cortex-M0+ NVIC 限制 + 中断优先级配置错误。** 两个定时器同级无法抢占，LED ISR 阻塞 bit-bang ISR，导致串口 bit 宽度畸变，同时主循环被饿死。

核心教训：**M0+ 没有 PRIGROUP，同级不嵌套。** 这个限制在 M3/M4 上不存在，是平台移植时最容易忽略的差异。

---

## 修复方案

| 措施 | 效果 |
|------|------|
| TMR14 优先级 1→2 | TMR3(优先级1) 可抢占 TMR14，模拟串口时序零干扰 |
| 频率 250→500Hz | 弥补优先级降低可能导致的微闪，实际体验更好 |
| 临界区缩短 | 关全局中断 ~15µs → <1µs (仅 memcpy 8 字节) |
| 稳态跳过 | 同色/全灭 + 状态无变化 → 直接 return，95%+ 调用零开销 |
| 异色路径优化 | 只关上一个 LED 而非全部 4 个，GPIO 操作减少 40% |

### 优先级修改

应用层直接调平台 CMSIS API，不动 lib-zos：

```c
// 放在 ZOSHardTimerStart 之后，确保不被 HAL 层覆盖
ZOSHardTimerOpen(ZOS_HAL_HARD_TIMER_14, &timerConf);
ZOSHardTimerStart(ZOS_HAL_HARD_TIMER_14);
NVIC_SetPriority(TMR14_IRQn, 2);  // 从默认的 1 降为 2
```

### 临界区优化

```c
// 之前: 临界区包了拷贝+分析 (~15µs)
ZOSHalInterruptEnable(false);
for (...) { 拷贝 + 判断同色/异色 + 计数 }
ZOSHalInterruptEnable(interruptState);

// 之后: 临界区只做原子拷贝 (<1µs)
ZOSHalInterruptEnable(false);
memcpy(ledSnapshot, g_ledScan, sizeof(ledSnapshot));  // 8 bytes
ZOSHalInterruptEnable(interruptState);
// 分析逻辑移出临界区
```

### 最终中断优先级矩阵

| 中断源 | 逻辑优先级 | 说明 |
|--------|-----------|------|
| WWDT | 0 (最高) | 系统异常恢复 |
| SysTick | 0 | 系统时基 1ms |
| TMR3 (模拟串口) | **1** | bit-bang 时序敏感，9600bps @ PA14 |
| TMR14 (LED扫描) | **2** | 允许 TMR3 抢占，500Hz |
| USART1 | 2 | 硬件 UART |

---

## 踩坑记录

### 坑1：Cortex-M0+ 同级优先级不嵌套

APM32E030 完全保留 ARMv6-M 的 NVIC 限制。在 STM32 Cortex-M3/M4 上可以通过 `PRIGROUP` 拆分抢占/响应优先级实现嵌套，但 M0/M0+ 根本没有这个寄存器。原代码注释"其他定时器同样降为优先级 1 避免延迟 TMR3"的逻辑在 M3 上是对的，在 M0+ 上恰好适得其反——同级意味着互相阻塞。

### 坑2：lib-zos 不应为了一个项目暴露通用接口

最初试图给 `ZOSHardTimerConf_t` 加 `priority` 字段。但 lib-zos 是多项目共享子模组，每个平台的定时器枚举值不同，暴露通用字段意义不大。最终方案：应用层 `NVIC_SetPriority(TMR14_IRQn, 2)` 一行覆盖。

### 坑3：轮询 UART 阻塞

`ZOSHalUartWrite` 是轮询阻塞的，115200bps 下每字节阻塞 ~87µs。长帧连续阻塞可达 ms 级。同级 ISR 在此期间无法抢占，需注意帧长控制。

### 坑4：bit-bang 对时序要求极其严格

9600bps 下 bit 周期 104µs，容限 ±2-5%（约 ±5µs）。任何一个 bit 错误都会破坏整个字节，起始位错误导致帧同步丢失，产生连续乱码。**bit-bang UART 的 ISR 优先级必须是所有定时器中最高的。**

---

## 验证方法

1. 把 PA14（模拟串口 TX）接到 USB-TTL 模块，9600bps 8N1
2. 上电，连续打 1000+ 条日志，对比收发字节一致性
3. 用示波器抓 PA14 波形，确认 bit 宽度稳定在 104µs ± 5µs 以内
4. 长时间跑（1h+），确认不再出现跑飞
5. 不同 LED 显示模式切换，肉眼确认无频闪

---

## 相关记录

- [APM32E030 (Cortex-M0+) 平台开发笔记](../../MCU/APM32E030/Platform_Notes.md)
