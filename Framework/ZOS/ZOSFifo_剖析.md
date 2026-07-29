# ZOS FIFO 环形缓冲区实现剖析（ZOSFifo.h）

## 背景

项目基于 ZOS 框架（Genie BT Mesh SDK），框架内置了一套 FIFO 实现：
`lib-zos/src/type/fifo/ZOSFifo.h`，全文不足 100 行，**全部用 C 宏实现**。
项目里串口、按键等异步数据都走它。本文剖析它的实现手法和使用时的真实约束，
作为后续在其它平台手写环形 buffer 的复用参考。

## 实现分析

### 1. 纯宏 + 静态数组，零动态内存

```c
#define ZOSFifoDef(type, fifo, size) \
struct fifo##_st {                   \
    unsigned short front, tail;      \
    type items[size];                \
} fifo
```

- 编译期定死容量，无 malloc、无碎片，宏展开后就是几条赋值/比较指令，无函数调用开销
- `front`/`tail` 是 16 位下标，不是指针

### 2. 绕环：到头回 0

```c
#define ZOS_FIFO_NEXT_TAIL(fifo) (tail + 1 == size ? 0 : tail + 1)
```

"环形"的全部实现就这一句：下标 +1，越界回 0。格子被反复覆盖使用，永不搬移数据。

### 3. 浪费一格区分空满

- 空：`front == tail`
- 满：`下一格(tail) == front`（即 `ZOSFifoHasSpace` 取反）

因为空满都是两指针相遇，必须牺牲一格。**SIZE=8 实际最多存 7 个**。

### 4. 读写分离的两步取出

```c
#define ZOSFifoFront(fifo) items[front]   // 只看，不动 front
#define ZOSFifoOut(fifo)   ...            // 只挪 front，不清数据
```

取数据 = 先 `ZOSFifoFront()` 读值，再 `ZOSFifoOut()` 挪下标。
分开的价值：**可以先窥视确认是完整一帧再消费；解析失败数据还在，可重试**。

### 5. InOnly：配合 DMA 的变体

```c
#define ZOSFifoInOnly(fifo)  // 只挪 tail，不写数据
```

适用场景：DMA 已把数据直接搬进 `items[tail]`，软件只需挪下标"入账"。
普通场景误用会读到垃圾。

## 使用约束（真实坑点）

1. **满了静默丢弃**：`ZOSFifoIn` 满时直接丢数据、无任何报错。
   关键数据必须自己先查 `ZOSFifoHasSpace`。
2. **无临界区保护**：该实现没有任何加锁。
   - 安全场景：**单生产单消费**（如中断 In、主循环 Out），
     各自动自己的下标，16 位下标在 32 位 MCU 上单次读写原子，基本安全。
   - 危险场景：两侧都会改同一根下标（如主循环也 In），必须关中断保护。
3. **取数据忘挪下标**：只 `Front` 不 `Out`，数据永远取不完，业务卡死。
4. **下标位宽**：`unsigned short`，SIZE 不要超过 65535；照搬到 8 位 MCU 注意溢出。

## 复用方法（串口收发标准模式）

```c
ZOSFifoDef(unsigned char, uartFifo, 64);   // 静态定义，全局

// 快的一方：中断里只放，立刻退出
void UART_IRQHandler(void) {
    unsigned char ch = 读硬件寄存器;
    ZOSFifoIn(uartFifo, ch);
}

// 慢的一方：主循环里取
while (1) {
    if (!ZOSFifoIsEmpty(uartFifo)) {
        unsigned char ch = ZOSFifoFront(uartFifo);  // 1. 读
        ZOSFifoOut(uartFifo);                        // 2. 挪下标（必做）
        解析处理(ch);
    }
}
```

这套模式可平移到任何"中断快产、循环慢消"的场景：按键事件、传感器采样、日志缓冲。

## 总结

- 环形 buffer 本质：固定数组 + 读写两个下标，下标越界回 0，永不搬数据
- ZOS 这版实现的价值不在代码量，而在取舍：**纯宏零开销、静态内存、
  看取分离、DMA 变体**，都是资源受限 MCU 上的标准手法
- 代价是**无任何保护**：单生产单消费下才安全，多写方必须自己补临界区
- 照搬到其它平台时，容量按 SIZE-1 规划，溢出策略（丢新/丢旧）按业务显式选择

---

## 附录：环形 Buffer 概念图解（个人学习笔记）

> 以下为原理扫盲，与具体框架无关。

### 纸面模型

一排格子 + 两根"手指"：**tail 管放，front 管取**，各自动各自的，到头绕回 0。

```
初始:     [0]   [1]   [2]   [3]
          空    空    空    空
          ↑
        front
        tail

放入A:    [0]   [1]   [2]   [3]
          A     空    空    空
          ↑     ↑
        front  tail          ← 放：只有 tail 动

取走A:    [0]   [1]   [2]   [3]
         (空)   B     空    空
                ↑     ↑
              front  tail    ← 取：只有 front 动

放入E(绕圈):
          [0]   [1]   [2]   [3]
          E     B     C     D    ← 0号格被重复使用
                ↑     ↑
              front  tail
```

### 三种状态

```
空：  front == tail
满：  tail的下一格 == front   （故意空一格不用）
其他： 中间状态
```

### 计数

```
tail >= front → 个数 = tail - front          （没绕圈）
tail <  front → 个数 = tail + SIZE - front   （绕圈，两段拼接）
```
