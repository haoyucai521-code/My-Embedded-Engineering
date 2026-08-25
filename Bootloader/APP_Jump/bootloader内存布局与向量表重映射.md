# STM32F070 Bootloader + App 架构与内存布局

> 项目:prod-mkyf-cy041e
> 平台:STM32F070x6(Cortex-M0,32KB Flash / 6KB RAM)
> 工程:EWARM_BOOTLOADER(IAR)、EWARM_MAIN(IAR)
> 记录日期:2026-08-12(该日期的架构快照,后续结构变动不保证同步)

---

## 1. 总体结构

- **bootloader 工程**:仅 STM32 配置(`startup_stm32f070x6.s` + `stm32f070x6.icf`),无 APM32 变体;代码以 LL 库为主,混入少量 HAL(`stm32f0xx_hal.h`)
- **main(app)工程**:双平台配置(STM32 + APM32E030),STM32 下有两份链接脚本:
  - `stm32f070x6_flash.icf` —— 独立版 app(无 bootloader,调试用)
  - `stm32f070x6_flash_boot_app.icf` —— **量产版**,配合 bootloader 使用
- **烧录顺序**:先烧 bootloader,再烧 app;量产可直接烧 main 工程产物(内含 bootloader,见第 4 节)

## 2. Flash / RAM 分区对比(icf)

| 项 | bootloader icf | main(boot_app)icf |
|---|---|---|
| 中断向量表 `.intvec` | 0x08000000 | **0x08001000** |
| ROM 范围 | 0x08000000–0x08007FFF | 0x08001000–0x08007FFF |
| BOOT_region | 无 | 0x08000000–0x08001000(前 4KB) |
| RAM 起始 | 0x20000000 | **0x200000D0**(让出前 208 字节) |
| CSTACK / HEAP | 0x600 / 0 | 0x600 / 0 |
| 特有段 | 无 | `.bootloader` 段放入 BOOT_region |

```
Flash(32KB)                          RAM(6KB)
┌────────────────────┐ 0x08000000    ┌────────────────────┐ 0x20000000
│ bootloader  (4KB)  │               │ app 向量表副本 208B │ ← app 运行时拷入
├────────────────────┤ 0x08001000    ├────────────────────┤ 0x200000D0
│                    │               │                    │
│ app(向量表+代码)   │               │ app .data/.bss/栈  │
│                    │               │                    │
└────────────────────┘ 0x08007FFF    └────────────────────┘ 0x200017FF
```

## 3. 核心知识点:Cortex-M0 无 VTOR 与向量表重映射

### 3.1 问题根源

- Cortex-M0 **没有 `SCB->VTOR` 寄存器**,CPU 取向量表的地址**硬编码为 0x00000000**(M3/M4 才有 VTOR 可改)
- 0x00000000 处**没有物理存储器**,它是一个"映射窗口",背后接哪块存储由 `SYSCFG->CFGR1` 的 `MEM_MODE[1:0]` 决定:

| MEM_MODE | 0x00000000 映射到 |
|---|---|
| 00 | 主 Flash(从 0x08000000 整块映射) |
| 01 | 系统存储器(出厂 ISP) |
| 11 | SRAM(从 0x20000000 整块映射) |

- 上电时 BOOT0 接地 → MEM_MODE=00 → 0x0 映射到 Flash 开头 → 执行 **bootloader 的向量表**
- bootloader 跳转 app **只是函数跳转,不触发复位、不改 MEM_MODE**,所以跳转后 0x0 窗口仍指向 bootloader 的向量表
- 此时 app 的中断来了,CPU 会跳去执行 bootloader 的中断入口 → 中断丢失或跑飞

### 3.2 为什么必须把向量表拷贝到 SRAM

目标:让 CPU 从 0x0 读到 app 的向量表。逐条排除:

1. **MEM_MODE 只能整块映射,无偏移参数** —— 无法让 0x0 窗口直接对准 Flash 0x08001000(app 向量表原件位置),Flash 这条路是死的
2. **系统存储器**是出厂 ISP,不能用
3. **只剩 SRAM** —— SRAM 运行期可写,窗口切到 SRAM 后,CPU 读 0x0 = 读 0x20000000
4. SRAM 是整块映射,**0x0 精确对应 0x20000000**,向量表必须逐字节放在 SRAM 最开头
5. 向量表原件在 0x08001000,数据不会自己出现在 SRAM —— **只能拷贝**

app 启动代码的标准流程:

```c
/* 1. 把 app 向量表(0x08001000)拷到 SRAM 起始处 */
memcpy((void*)0x20000000, (void*)0x08001000, 0xD0);

/* 2. 切换映射:0x00000000 窗口指向 SRAM */
SYSCFG->CFGR1 |= SYSCFG_CFGR1_MEM_MODE;
```

附带好处:向量表在 RAM 中,运行期可动态修改中断入口。

### 3.3 为什么是 208 字节(0xD0)

- STM32F070 向量表:16 个内核异常 + 32 个外设中断 = **48 入口 × 4B = 192B(0xC0)**
- 实际让出 **0xD0(208B)**,多 16B 为对齐/余量,防不同 F0 型号向量数差异
- icf 中 app 的 RAM_region 从 0x200000D0 开始,防止链接器把变量分到这片区域被 memcpy 覆盖
- 代价:app 可用 RAM 从 6144B 缩到 **5936B**

## 4. main 工程的 `.bootloader` 段(boot.bin 嵌入)

### 4.1 机制

main 工程 IAR 链接器配置(app.ewp)启用了 **Raw binary image** 输入:

```
IlinkRawBinaryFile    = boot.bin      ← EWARM_MAIN 目录下
IlinkRawBinarySymbol  = bootloader
IlinkRawBinarySegment = .bootloader
IlinkKeepSymbols      = bootloader    ← 防止被链接器回收
```

链接时把 bootloader 工程单独编译出的 `boot.bin` 原封不动嵌入 `.bootloader` 段;icf 再规定:

```
define region BOOT_region = mem:[from 0x08000000 to 0x08001000];
place in BOOT_region { readonly section .bootloader };
```

### 4.2 效果与目的

- main 工程的烧录产物中,**前 4KB 是 bootloader,0x08001000 之后是 app** —— 一次烧录两段俱全
- 产线只需烧一个文件,避免漏烧和版本错配
- app 与 bootloader 版本绑定在同一产物中

### 4.3 风险点

- ⚠️ **bootloader 源码改动重编后,必须手动把新 boot.bin 拷到 EWARM_MAIN 目录**,否则 main 产物里仍是旧 bootloader —— 最常见的坑
- boot.bin 必须 ≤ 4KB,超出时链接器报 BOOT_region 溢出(也算一种保护)

## 5. 完整启动链路

```
上电(BOOT0=0)
  → MEM_MODE=00,0x0 映射到 Flash 0x08000000
  → 执行 bootloader 向量表,bootloader 运行
      │
      ├─ 需要升级:走 UART 刷写 app 区
      └─ 正常:校验 app 后跳转 0x08001000 的 Reset_Handler
              → app 启动代码 memcpy 向量表到 0x20000000
              → 写 SYSCFG->CFGR1,MEM_MODE=11
              → 0x0 窗口切到 SRAM,app 接管全部中断
              → 进入 app main()
```

## 6. 移植 APM32E030 注意事项

> 以下为 2026-08-12 时的认知,**未验证**;APM32 bootloader 后续已建立
> (`src/bootloader_apm32/`),实战排查见 `Debug/Crash/bugfix_oad_random_reset_and_brick.md`。

- APM32E030 的内核版本(Cortex-M0 还是 M0+)与 VTOR 有无**未查证**——这直接决定
  向量表接管方案:有 VTOR 直接写 `SCB->VTOR` 即可,无 VTOR 才需要本文的 SRAM 重映射。
  以 APM32 官方手册为准,勿照搬 STM32 结论。
- 寄存器名 / SYSCFG 位定义同样以 APM32 手册为准。
