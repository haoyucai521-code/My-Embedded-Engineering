# r156h_byd 平台开发笔记

## 平台说明

- **平台代号**：r156h_byd
- **所属工程**：儿童座椅
- **SDK 路径**：`genie-bt-mesh-sdk-rel_1.3.4/app/example/prod-mkyf-cy041c/r156h_byd`
- **主要用途**：BLE 连接与业务交互

> 注：芯片具体型号、厂商、核心架构等硬件信息当前未整理，后续补充。

---

## 开发环境

- SDK：genie-bt-mesh-sdk-rel_1.3.4
- 构建宏控示例：`CONFIG_OM6626_BLE_WHITELIST=1`
- 当前在该平台上启用白名单功能，另一平台保持原样。

---

## 已验证 BLE SDK 行为

### 1. 连接断开后广播不会自动恢复

**现象**：

BLE 设备连接并断开后，广播消失，其他设备无法再次扫描到。

**根因**：

当前平台 SDK 在连接断开后**不会自动恢复广播**。这与另一平台的行为存在差异（另一平台可通过 `disconnectNoRestart` 控制）。

**处理方案**：

在 `OB_GAP_EVT_DISCONNECTED` 事件里主动恢复广播，并引入 `g_advShouldRun` / `g_disconnectNoRestart` 状态机：

```c
if (g_advShouldRun && !g_disconnectNoRestart) {
    startAdvInternal();
}
```

**风险**：

- 如果应用层不主动恢复，设备断连后对客户不可见。
- 不同平台 SDK 行为不一致，移植代码时需要显式适配。

---

### 2. SDK 对 RPA/IRK 的支持存在问题

**现象**：

Controller 层 Accept List 方案中，Bond 设备重连时无法命中白名单，导致 Bond 设备也连不上。

**根因**：

- Bond 完成后保存的是 identity 地址。
- Bond 设备重连时大概率使用 RPA（Resolvable Private Address）。
- 当前平台 SDK 没有公开 Resolve List（RL）管理 API，Controller 无法自动把 RPA 解析成 identity 地址再匹配白名单。

**处理方案**：

放弃 Controller 层过滤，改为应用层在 `OB_GAP_EVT_CONNECTED` 后用保存的 IRK 自行解析 RPA：

```c
if (slot->peerIrkValid && ob_gap_rpa_check(slot->peerIrk, addr->addr)) return true;
```

同时需要确保 IRK 被完整持久化（见 [Projects/儿童座椅/WhiteList_Debugging_Journey.md](../../Projects/儿童座椅/WhiteList_Debugging_Journey.md)）。

**风险**：

- 应用层过滤会让非白名单设备短暂建立 GAP 连接，存在被探测到的窗口。
- 非白名单设备可能触发自动重连循环，需要额外抑制策略。

---

### 3. Accept List API 调用时机受限

**现象**：

调用 `ob_gap_accept_list_set()` 设置白名单后，广播状态机容易混乱，出现广播消失或策略不生效。

**根因**：

`ob_gap_accept_list_set()` 要求**在无广播使能时调用**，即必须先 `ob_gap_adv_stop()`，设白名单，再 `ob_gap_adv_start()`。

但当前平台 SDK 广播状态机本身不稳定，反复重启广播容易失败。

**处理方案**：

放弃 Controller 层 Accept List 方案，改用应用层过滤。

---

## 常见坑

| 问题 | 触发条件 | 规避方案 |
|------|----------|----------|
| 断连后广播不恢复 | 任何 BLE 连接断开后 | 应用层在 `OB_GAP_EVT_DISCONNECTED` 里主动恢复广播 |
| RPA 无法被 Controller 解析 | Bond 设备使用 RPA 重连 | 应用层用保存的 IRK 自行解析 |
| Accept List 设置导致广播异常 | 在广播使能期间调用 | 必须停广播再设置，或改用应用层过滤 |
| IRK 重启后丢失 | Bond 信息保存不完整 | 确保 `irk.id_info.irk` 被完整持久化 |

---

## 待补充

- [ ] 芯片具体型号与厂商
- [ ] 核心架构与主频
- [ ] 启动流程
- [ ] 时钟系统
- [ ] 内存布局（RAM / Flash）
- [ ] 其他外设使用情况
- [ ] 烧录与调试工具链

---

## 相关记录

- [儿童座椅 BLE 白名单调试心路历程](../../Projects/儿童座椅/WhiteList_Debugging_Journey.md)
