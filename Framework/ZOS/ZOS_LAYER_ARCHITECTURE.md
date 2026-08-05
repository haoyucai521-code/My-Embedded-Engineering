# ZOS 三层抽象架构

## 一句话定位

**HAL 是合同，Driver 是共享逻辑，Platform 是芯片翻译。** 三层的核心目标只有一个：**加新芯片只改 Platform，Driver 和应用代码零改动。**

---

## 整体结构

```
应用层 (你的 Main.c / Fan.c / Battery.c)
  ↓ 只依赖 Driver 层接口，不 include 任何芯片 SDK 头文件
╔══════════════════════════════════════════════════╗
║              Driver 层 — 共享逻辑                  ║
║  写一次，所有芯片复用。                             ║
║  内容：缓冲区管理、队列、状态机、事件推导、背压控制    ║
║  入口：ZOSDriverInit / ZOSDriverHandle             ║
╚══════════════════════════════════════════════════╝
  ↓ #include "hal/ZOSHalXxx.h"（唯一的平台依赖）
───────────────────────────────────────────────────
              HAL 层 — 接口合同
  只有 .h，没有 .c
  定义：每个外设模块必须实现哪些函数、什么签名
  Driver 层只依赖这一套头文件，永远不出现芯片宏和 SDK 类型
───────────────────────────────────────────────────
  ↓ 编译时通过 Makefile 选一个 platform 目录链接
╔══════════════════════════════════════════════════╗
║            Platform 层 — 芯片翻译                  ║
║  每颗芯片一份，用芯片 SDK / 寄存器填写 HAL 合同。    ║
║  不包含任何可跨芯片复用的逻辑。                      ║
║  om6626 / esp32 / stm32 / linux / windows / ...    ║
╚══════════════════════════════════════════════════╝
  ↓
芯片硬件
```

---

## 每一层的职责

### HAL 层 — "合同"

**内容**：位于 `lib-zos/src/hal/`，全是 `.h` 头文件，只有函数声明没有实现。

```c
// ZOSHalGpio.h — 一份简洁的合同
void ZOSHalGpioInit(void);
int  ZOSHalGpioOpen(ZOSHalGpio_t gpio, ZOSGpioConf_t *conf);
int  ZOSHalGpioSet(ZOSHalGpio_t gpio, int value);
int  ZOSHalGpioGet(ZOSHalGpio_t gpio);
```

**存在理由**：让 Driver 层和所有 Platform 之间的依赖从 N×M 变成 1×M。

- 没有 HAL：Driver 层用 `#ifdef` 切换 14 颗芯片 → 14 个 driver × 10 个外设 = 不可维护
- 有 HAL：Driver 层只 include HAL 头文件，永不出现在芯片逻辑中。加新芯片 = 实现这份合同，Driver 零改动

**关键约束**：
- HAL 目录没有 `.c` 文件，没有实现，不能有芯片相关宏
- HAL 用 `ZOSHalGpio_t` 这种框架自有类型，不暴露芯片 SDK 的类型

---

### Driver 层 — "共享逻辑"

**内容**：位于 `lib-zos/src/driver/`，是写一次在所有芯片上复用的框架代码。有三类职责：

#### 类型 1：框架级能力（最核心）

Platform 只提供原子操作（读写一个字节），Driver 用这些原子操作拼出完整能力。

**示例 — UART 缓冲区管理**（[ZOSUart.c](lib-zos/src/driver/ZOSUart.c)）：

```
Platform 只负责：从硬件 FIFO 读完数据 → 调 _ZOSUartHalInputData(uart, data, len)

Driver 负责：
  - 环形 buffer（ZOSBufferCreate / Read / Write）
  - 状态跟踪（open / full）
  - 主循环回调分发（ZOSUartHandle → recvHandler）
  - buffer-full 告警 + 限频日志
```

数据流：`Platform(中断) → _ZOSUartHalInputData → Driver(buffer) → ZOSUartHandle(主循环) → recvHandler(应用层)`

Platform 层不感知 buffer 的存在。

**示例 — BLE Write 队列**（[ZOSBle.c](lib-zos/src/driver/ZOSBle.c)）：

```
Platform 只负责：BLE 协议栈事件 → 调 _ZOSBleWrite(char, data, len)

Driver 负责：
  - 侵入式链表队列（中断上下文塞入，主循环取出）
  - 自旋锁保护
  - 背压控制（队列超阈值 ZOSSleep 延迟）
  - 回调分发（遍历 service/char 找到 onWrite 回调）
```

**示例 — BLE 连接状态到事件推导**：

```
Platform 只负责：ZOSHalBleConnected() → 返回 true/false

Driver 负责：
  - 保存上一次状态 g_lastConnState
  - 每轮比较 current vs last
  - 状态变化时触发 onConnected / onDisconnected 回调
```

Platform 只回答"现在连着吗"，Driver 把它变成"刚才断开了"的事件通知。

#### 类型 2：统一编排（系统级入口）

[ZOSDriver.c](lib-zos/src/driver/ZOSDriver.c) 是框架层的大脑：

```c
void ZOSDriverInit(void) {
    ZOSHalMiscInit();              // 1. 基础设施（时间/中断）
#if ZOS_OPTION_HAL_GPIO_SWITCH
    ZOSGpioInit();                 // 2. 基础外设
#endif
#if ZOS_OPTION_HAL_UART_SWITCH
    ZOSUartInit();                 // 3. 通信外设
#endif
#if ZOS_OPTION_HAL_BLE_SWITCH
    ZOSBleInit();                  // 4. 复杂协议栈
#endif
}
```

**三个设计决策**：
1. **初始化顺序 = 依赖关系**：基础设施 → 基础外设 → 通信外设 → 复杂协议栈，不可随意调换
2. **编译期裁剪**：`ZOS_OPTION_HAL_WIFI_SWITCH 0` 时，WiFi Init/Handle 及全部代码物理删除，固件里零字节
3. **核心调度器无感知**：`ZOSEventLoop` 只认识 Driver/Task/Net 三个概念，新增外设只需在 ZOSDriver.c 加两行，核心调度器零改动

#### 类型 3：参数校验

最薄的一层，检查参数合法性（索引越界、模块是否已初始化），防止非法调用穿透到 Platform。

---

### Platform 层 — "芯片翻译"

**内容**：位于 `lib-zos/src/platform/<芯片名>/`，用芯片 SDK 或寄存器实现 HAL 合同。

```c
// platform/om6626/ZOSHalGpio.c
int ZOSHalGpioSet(ZOSHalGpio_t gpio, int value) {
    return GPIO_PinWrite(gpioLookup[gpio], value);  // 芯片 SDK 调用
}

// platform/linux/ZOSHalGpio.c
int ZOSHalGpioSet(ZOSHalGpio_t gpio, int value) {
    return write(gpio_fd, value ? "1" : "0", 1) == 1 ? 0 : -1;  // POSIX 系统调用
}
```

同一份合同，14 种完全不同的实现。

**关键约束**：
- Platform 不写任何可复用的逻辑（缓冲、队列、状态机归 Driver）
- Platform 完成硬件操作后，通过调 Driver 暴露的函数回传数据（例如 `_ZOSUartHalInputData`）
- 换芯片只需新增一个 platform 目录 + 覆盖 ZOSOptionsUser.h 的编译开关

---

## 为什么是三层，两层不行吗

### 砍掉 Driver → 应用直接调 HAL

```
app → HAL(接口) → platform(实现)
```

**死因**：UART 的缓冲区、BLE 的 write 队列、连接状态机、扫描去重——这些纯软件逻辑要么每个应用重写，要么塞进 Platform。塞进 Platform 意味着 om6626 写一遍、ESP32 写一遍、Linux 写一遍，**N 份相同的代码**。

### 砍掉 HAL → Driver 直接调 Platform

```
app → driver → platform
```

**死因**：Driver 必须知道调哪个 Platform 的哪个函数——`#ifdef MODULE_OM6626` 满天飞。**加一个新平台要改所有 Driver 文件**，M×N 爆炸。

### 三层是唯一解

Driver 和 Platform 之间唯一的耦合是 HAL 的函数签名。换芯片 = 签同一份合同，用新 SDK 实现——合同不变，签字方换人。

---

## 这套架构教给你的思维模型

### 一句话：依赖倒置

**你的代码依赖你定义的接口，不依赖别人的实现。**

```
不该做的事：Main.c → #include "om6626_sdk.h" → 换芯片全废
该做的事：  Main.c → 你定义的接口(你控制) → 别人的实现(对方控制，可替换)
```

### 抽象边界的判断标准

每加一层，问自己：**这一层有没有引入新的语义？**

- HAL 层引入新语义：把"芯片操作"翻译成"统一接口" → 值
- Driver 层的 UART buffer 引入新语义：把"字节流"变成"缓冲+回调" → 值
- Driver 层的 BLE 连接跟踪引入新语义：把"状态查询"变成"事件通知" → 值
- GPIO driver 层只做参数校验然后原样转发 → 没有新语义 → 过度封装

**没有语义提升的包装层不值得加。**

### 编译器是你的朋友

不是所有功能都靠 `if/else` 在运行时开关。能用 `#if` 在编译期裁剪的，不要留到运行时：

- om6626 没有 WiFi → `#if 0` → 整段代码不存在 → Flash 零占用
- Release 固件不需要调试日志 → `ZOS_OPTION_LOG_DISABLED 1` → 所有日志宏变成空语句

嵌入式的资源是钱。编译期裁剪 = 不用的功能不付钱。
