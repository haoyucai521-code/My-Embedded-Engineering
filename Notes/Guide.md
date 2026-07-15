# Embedded Knowledge Base Guide

## 1. 项目说明

这个仓库用于长期积累嵌入式软件开发经验。

主要记录：

- MCU平台经验
- 驱动开发经验
- Debug定位过程
- 硬件问题分析
- 通信协议理解
- RTOS使用经验
- 项目开发经验
- AI辅助开发流程


记录原则：

1. 记录真实开发过程中遇到的问题
2. 记录自己的分析过程
3. 记录最终解决方案
4. 记录未来可以复用的方法


维护频率：

每天10~15分钟。


---

# 2. 目录结构


```text
.
├── AI_Workflow/
├── Bootloader/
├── Debug/
├── Driver/
├── Hardware/
├── MCU/
├── Notes/
├── Projects/
├── Protocol/
├── RTOS/
└── Tools/
```


---

# 3. 各目录说明


# AI_Workflow


## 用途

记录AI辅助嵌入式开发的方法。


包含：

- AI Agent使用方式
- Prompt模板
- 工程分析流程
- 代码Review流程
- Debug辅助流程


目录建议：

```text
AI_Workflow/

├── Prompt/
├── Agent/
├── Code_Review/
├── Debug_Process/
└── Templates/
```


记录内容：

例如：

- 如何让AI分析大型工程
- 如何让AI定位Bug
- 如何让AI阅读驱动代码
- 如何生成测试代码
- 如何进行代码Review


---

# Bootloader


## 用途

记录Bootloader相关经验。


包含：

- OTA升级
- Flash管理
- APP跳转
- 固件校验
- 升级协议


目录建议：

```text
Bootloader/

├── OTA/
├── Flash/
├── APP_Jump/
├── CRC/
└── Problems/
```


重点记录：

- Flash布局设计
- 启动流程
- APP跳转条件
- 升级失败处理
- 常见问题


---

# Debug


## 用途

建立自己的问题定位数据库。


包含：

- HardFault
- 死机
- 重启
- 内存异常
- 外设异常
- 软件逻辑错误


目录建议：

```text
Debug/

├── HardFault/
├── Memory/
├── Crash/
├── Logic/
├── Hardware/
└── Tools/
```


重点：

不要只记录答案。

需要记录：

- 现象
- 排查过程
- 使用工具
- 根因
- 解决方法


---

# Driver


## 用途

记录驱动开发经验。


包含：

- GPIO
- UART
- SPI
- I2C
- PWM
- ADC
- Timer
- Sensor


目录建议：

```text
Driver/

├── GPIO/
├── UART/
├── SPI/
├── I2C/
├── PWM/
├── ADC/
└── Sensor/
```


每个驱动建议记录：


1. 工作原理

2. 初始化流程

3. API使用方式

4. 常见问题

5. 调试方法

6. 可复用代码


---

# Hardware


## 用途

记录软硬件结合问题。


包含：

- 原理图分析
- 电源问题
- 信号问题
- PCB问题
- 元器件问题


目录建议：

```text
Hardware/

├── Power/
├── Signal/
├── PCB/
├── Schematic/
└── Components/
```


记录：

- 问题现象
- 测试方法
- 分析过程
- 解决方案


---

# MCU


## 用途

记录不同MCU平台经验。


包含：

- STM32
- GD32
- ESP32
- Nordic
- 其他MCU


目录建议：

```text
MCU/

├── STM32/
├── GD32/
├── ESP32/
├── NRF52/
└── Others/
```


每个平台记录：

- 芯片特点
- 启动流程
- 时钟系统
- 外设特点
- 开发环境
- 常见坑


---

# Projects


## 用途

记录真实项目经验。


目录：

```text
Projects/

├── Project_A/
├── Project_B/
```


项目结构：

```text
Project_A/

├── README.md
├── Architecture.md
├── Problems.md
└── Lessons.md
```


记录：

- 项目背景
- 硬件平台
- 软件架构
- 负责内容
- 遇到的问题
- 解决方案
- 项目总结


---

# Protocol


## 用途

记录通信协议经验。


包含：

- UART
- SPI
- I2C
- BLE
- MQTT
- TCP/IP
- 自定义协议


目录：

```text
Protocol/

├── UART/
├── SPI/
├── I2C/
├── BLE/
├── MQTT/
└── Custom/
```


重点记录：

- 数据格式
- 通信流程
- 调试方式
- 常见错误


---

# RTOS


## 用途

记录RTOS开发经验。


包含：

- FreeRTOS
- Task设计
- Queue
- Semaphore
- Mutex
- Memory管理


目录：

```text
RTOS/

├── Task/
├── Queue/
├── Semaphore/
├── Memory/
└── Problems/
```


记录：

- 使用场景
- 设计方案
- 遇到的问题
- 解决方式


---

# Tools


## 用途

记录开发工具经验。


包含：

- 编译工具
- 调试工具
- 烧录工具
- 自动化脚本


目录：

```text
Tools/

├── GCC/
├── Keil/
├── JLink/
├── OpenOCD/
└── Scripts/
```


---

# Notes


## 用途

个人笔记和规划。


文件：

Guide.md

仓库说明。


Plan.md

长期维护计划。


---

# 经验记录模板


```md
# 标题


## 背景

为什么需要处理这个问题。


## 问题现象

发生了什么。


## 分析过程

如何定位。


## 解决方案

最终怎么解决。


## 总结

以后如何避免。
```


---

# Bug记录模板


```md
# Bug名称


## 环境

MCU:

硬件:

软件版本:


## 现象


## 排查过程


## 根因


## 修复方案


## 防止再次发生的方法
```


---

# 项目总结模板


```md
# 项目名称


## 项目介绍


## 硬件平台


## 软件架构


## 负责内容


## 技术难点


## 问题记录


## 项目经验总结
```


---

# 维护规则


每天：

记录一个小问题或者一个知识点。


每周：

整理一次目录。


每月：

输出一个专题总结。


例如：

- STM32 UART DMA总结
- Bootloader OTA总结
- I2C问题定位总结
- 嵌入式Debug方法总结
```