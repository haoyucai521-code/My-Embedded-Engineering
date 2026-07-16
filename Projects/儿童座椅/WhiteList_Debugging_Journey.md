# 儿童座椅 BLE 白名单机制调试

## 环境

- **工程**：儿童座椅
- **平台**：OM6626（仅当前构建启用）
- **宏控**：`CONFIG_OM6626_BLE_WHITELIST=1`
- **涉及协议**：BLE GAP、RPA（Resolvable Private Address）、IRK
- **关键文件**：
  - `lib-zos/src/platform/om6626/ZOSHalBle.c`
  - `src/BleProfile.c`
  - 当前平台构建 Makefile

---

## 现象

儿童座椅业务协议要求：

- 设备未绑定前：所有人可扫描、可连接。
- 设备绑定一台车机后：所有人仍可扫描，但**只有已绑定的车机能保持连接**，其他设备发起连接后必须被拒绝。
- 仅影响当前平台构建，另一平台保持原样。

核心约束：可被扫描，但不可被非白名单设备连接。

---

## 排查过程

### 阶段一：Controller 层白名单（尝试，失败）

SDK 提供了 `ob_gap_accept_list_set()` 和广播 `filter_policy`：

```c
enum ob_adv_filter_policy {
    OB_ADV_FILTER_NONE,      // 不过滤
    OB_ADV_FILTER_SCAN,      // 只过滤 scan request
    OB_ADV_FILTER_CONN,      // 过滤连接请求，不过滤 scan
    OB_ADV_FILTER_CONN_SCAN, // scan 和 connect 都过滤
};
```

理想方案：

1. Bond 完成后，把对端 identity 地址写入 `ob_gap_accept_list_set()`。
2. 广播 `filter_policy` 切到 `OB_ADV_FILTER_CONN`。
3. 所有人都能 scan 到，但只有白名单设备能发起连接。

#### 遇到的问题

1. **初次上电 Bond 完成后，设备无法被任何设备 scan 到。**
   - 误用了 `OB_ADV_FILTER_CONN_SCAN`，该策略会同时过滤 scan request。
   - 即使改成 `OB_ADV_FILTER_CONN`，后续问题仍未解决。

2. **Bond 设备也连不上。**
   - Accept List 里存的是 identity 地址，但 Bond 设备重连时大概率使用 RPA。
   - 当前平台 SDK 没有公开 Resolve List（RL）管理 API，Controller 无法自动把 RPA 解析成 identity 地址再匹配白名单。

3. **连接断开后广播消失。**
   - `ob_gap_accept_list_set()` 要求**在无广播使能时调用**。
   - 实现时必须先 `ob_gap_adv_stop()`，设白名单，再 `ob_gap_adv_start()`。
   - 但当前平台 SDK 的广播状态机在这台设备上本身就不稳定，停播后经常起不来。

4. **广播重启后白名单策略没生效，或者设备直接不可见。**
   - 调用时机和返回值没严格处理，失败也没重试，导致状态混乱。

#### 结论

Controller 层 Accept List 方案放弃。原因不是 API 不存在，而是：

- 调用时机受限（必须停广播）。
- RPA/RL 是黑盒，Bond 设备重连无法保证命中。
- SDK 广播状态机不稳定，反复重启广播容易掉广播。

### 阶段二：应用层白名单过滤（最终方案）

把过滤点放在**连接建立之后、业务交互之前**：

1. 广播保持全开放（`OB_ADV_FILTER_NONE`），所有人都能 scan 和发起连接。
2. 收到 `OB_GAP_EVT_CONNECTED` 后，在应用层判断对端地址是否在白名单内。
3. 如果不在白名单，立即调用 `ob_gap_disconnect(conn_idx, 0x05)` 断开。

这样非白名单设备能短暂建立 GAP 连接，但会被立刻踢掉，业务层不会感知。

---

## 根因

1. **Controller 层过滤不可控**：SDK 对 Accept List + RPA/RL 的支持不完整，调用时机受限，广播状态机不稳定。
2. **IRK 持久化缺失**：Bond 信息保存时只存了部分字段，重启后 IRK 丢失，导致 RPA 解析失败。
3. **广播恢复机制缺失**：当前平台 SDK 连接断开后不会自动恢复广播，必须应用层补齐。
4. **协议层无法阻止对端重连**：断开原因码只是建议，系统/APP 层自动重连逻辑可以忽略。

---

## 修复方案

### 1. 白名单匹配逻辑

白名单数据来源：已持久化的 Bond Slot。

匹配顺序（三者满足其一即放行）：

1. 对端地址是 RPA，且能用保存的 IRK 解析成功。
2. 对端地址与保存的 identity 地址一致。
3. 对端地址与 Bond 时记录的 `peerAddr` 一致。

```c
static zbool_t omBleWhitelistAllowConnection(const ob_gap_addr_t *addr)
{
    for (int i = 0; i < MAX_BOND_PEER_NUM; i++) {
        ZOSHalBleBondSlot_t *slot = &g_bondSlots[i];
        if (!slot->used) continue;
        if (slot->peerIrkValid && ob_gap_rpa_check(slot->peerIrk, addr->addr)) return true;
        if (slot->peerIdAddrValid &&
            slot->peerIdAddr.addr_type == addr->addr_type &&
            memcmp(slot->peerIdAddr.addr, addr->addr, 6) == 0) return true;
        if (slot->peerAddr.addr_type == addr->addr_type &&
            memcmp(slot->peerAddr.addr, addr->addr, 6) == 0) return true;
    }
    return false;
}
```

### 2. IRK 持久化修复

问题：只在 `OB_GAP_EVT_BOND_INFO` 里保存 IRK 到 `peerIrk[]`，但 `saveBondStore()` 持久化时读的是 `bondSlot->irk.id_info.irk`。

修复：把事件里的 bond_info 同时 `memcpy` 到 `bondSlot->irk`：

```c
memcpy(&bondSlot->irk, &evt->gap.bond_info, sizeof(bondSlot->irk));
bondSlot->irkValid = true;
```

### 3. 断开后广播恢复

当前平台 SDK 连接断开后**不会自动恢复广播**。

修复：在 `OB_GAP_EVT_DISCONNECTED` 里主动恢复广播，并参考另一平台引入 `g_advShouldRun` / `g_disconnectNoRestart` 机制。

```c
if (g_advShouldRun && !g_disconnectNoRestart) {
    startAdvInternal();
}
```

### 4. 非白名单设备重连抑制

应用层过滤的副作用：非白名单设备被断开后，如果它有自动重连逻辑，会立即再次发起连接，形成循环。

缓解手段：

- 断开原因码从 `0x13`（Remote User Terminated Connection）改为 `0x05`（Authentication Failure），效果有限。
- 非白名单拒绝后延迟恢复广播：先 2 秒，后调到 5 秒。

延迟恢复可以把对端重连周期拉长，但无法彻底阻止。

---

## 最终行为

1. **未绑定状态**：广播全开，所有人可扫描、可连接、可 Bond。
2. **绑定完成后**：
   - 所有人仍可扫描。
   - 非白名单设备发起连接后，GAP 连接短暂建立，随后被 `0x05` 断开。
   - Bond 设备可以正常连接（包括重启后用 RPA 重连）。
3. **非白名单设备疯狂重连时**：
   - 每次拒绝后广播静默 5 秒，再恢复。
   - 重连频率被强制降低，但无法完全消除。

---

## 经验教训

1. **不要迷信 Controller 层 API。**
   - 有 API 声明不代表行为稳定，尤其是调用时机受限、RPA/RL 黑盒时。

2. **应用层过滤是兜底方案。**
   - 虽然不够优雅，但可控、可调试、不依赖 SDK 黑盒行为。

3. **广播状态机要重点关注。**
   - 当前平台 SDK 连接断开后不自动恢复广播，这是 SDK 行为差异，必须在 HAL 层补齐。

4. **RPA/identity/IRK 要保存完整。**
   - 不能只存 RPA，必须保存 identity 地址和 IRK，否则重启后白名单失效。

5. **协议层无法阻止对端重连。**
   - 断开原因码只是建议，系统/APP 层的自动重连逻辑可以忽略。
   - 只能通过设备端行为（延迟恢复广播、停止广播）降低频率。

---

## 遗留问题

1. **非白名单设备仍会持续尝试重连。**
   - 当前方案只能降低频率，无法根治。
   - 根治需要发起端配合（停止自动重连），或切换为不可连接广播类型（但会影响 Bond 设备连接）。

2. **延迟恢复广播期间，Bond 设备也无法连接。**
   - 这是延迟恢复的副作用，目前可接受。

3. **5 秒延迟是否足够仍需实测。**
   - 如果对方重连间隔小于 5 秒，静默期内它会连续失败；恢复后又会连上再断开。
   - 若仍刷屏，可考虑指数退避或更长的静默窗口。

---

## 防止再次发生的方法

- 涉及 BLE 安全、Bond、白名单的功能，必须先确认 SDK 对 RPA/RL/Accept List 的支持程度，再决定过滤点。
- Bond 信息持久化时，必须检查 IRK、identity 地址、peerAddr 是否完整保存并正确读取。
- 跨平台 SDK 移植时，必须验证广播状态机在连接/断开后的行为差异。
- 遇到对端自动重连问题时，优先从设备端行为控制频率，不要期望协议层 error code 能彻底阻止。

---

## 调试建议

后续若继续优化，可观察以下日志：

```text
om ble conn reject by whitelist, addr=...
om ble adv restart delayed 5s by whitelist reject
om ble delayed restart adv ok
```

重点关注：

- 拒绝频率是否下降。
- Bond 设备重启后是否能用 RPA 连上。
- 正常断开后广播是否自动恢复。
