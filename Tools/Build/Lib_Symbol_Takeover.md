# 链接期接管闭源库函数（LIB SYMBOL TAKEOVER）

> 适用：闭源 SDK 静态库（`.a`）中的某个函数行为固定、无源码可改，但业务需要接管其返回值/行为。
> 本文档以某工程接管 HiLink SDK `GetErrcodePayload`（custom 通道 RSP payload）为例，沉淀可复用的方法与脚本。

## 一、为什么可以做到

闭源 `.a` 静态库**不是最终产物**。库里的 `.o` 是可重定位目标文件，符号表公开可改，链接发生在构建机上。

`GetErrcodePayload` 在 `libhilinkbtsdk.a` 内是全局 `T` 符号，SDK 发送 RSP 前调用它取 payload（原始实现返回固定 `{"errcode":%d}`）。我们可以：

1. 用 `objcopy --redefine-sym` 把库内该符号改名（**只改符号表字符串，机器码一个字节不动**）
2. 改名后库内所有调用点变成"未解析引用（U）"
3. 在自己的业务源码里提供一个同名实现，链接器把未解析引用绑到我们的实现
4. 我们的实现需要原始行为时转调改名后的旧符号（兜底）

```
SDK 内部调用 ──→ GetErrcodePayload（业务实现）
                    ├─ 业务预设 payload → 返回业务数据
                    └─ 未预设 → 转调 GetErrcodePayload_Legacy（库内原实现）
```

## 二、三步法

```
1. 定位符号：nm 确认全局 T 定义，反汇编确认函数签名与调用时序
2. 制造缺口：objcopy 改名，ar 重打回库
3. 填缺口 + 兜底：提供同名新实现，必要时转调旧符号
```

## 三、通用脚本

以下脚本参数化 lib 路径、符号名、工具链目录，可独立复用。核心流程与当前工程 `build_bsl600.sh` 的 patch 段一致。

```sh
#!/bin/sh
# ==============================================================================
# lib_symbol_takeover.sh
# 通用：将静态库内某全局函数改名为 _Legacy，供业务层同名接管。
# 用法:
#   lib_symbol_takeover.sh \
#     <lib> <obj_member> <symbol> <new_symbol> \
#     <binutils_dir> <backup_dir>
#   例:
#   lib_symbol_takeover.sh \
#     sdk/lib/libfoo.a foo.c.o GetFoo GetFoo_Legacy \
#     /opt/riscv-gcc/bin temp/back
# ==============================================================================

set -u

LIB="$1"            # 目标静态库路径（将被打 patch）
OBJ_MEMBER="$2"     # 符号定义所在 .o 成员名（ar t 查看）
SYMBOL="$3"         # 要接管的符号
NEW_SYMBOL="$4"     # 改名后的符号（业务实现以此转调原函数）
NM="$5/nm"          # 工具链 nm
AR="$5/ar"
OBJCOPY="$5/objcopy"
BACKUP_DIR="$6"

# 幂等 + 版本保护：每次构建可重复执行，SDK 换库不会静默出错
if "${NM}" "${LIB}" 2>/dev/null | grep -q "T ${NEW_SYMBOL}\$"; then
    echo "lib 已处于 patch 状态，跳过"
    exit 0
fi
if ! "${NM}" "${LIB}" 2>/dev/null | grep -q "T ${SYMBOL}\$"; then
    echo "警告: lib 中未找到 ${SYMBOL} 符号，跳过 patch（SDK 版本可能变更，需人工确认）" >&2
    exit 1
fi

# 原始 lib 备份，构建完成后可还原
mkdir -p "${BACKUP_DIR}/$(dirname "${LIB}")"
cp -p "${LIB}" "${BACKUP_DIR}/${LIB}"

PATCH_TMP=$(mktemp -d)
(
    cd "${PATCH_TMP}" || exit 1
    "${AR}" x "${LIB}" "${OBJ_MEMBER}" &&
    "${OBJCOPY}" --redefine-sym "${SYMBOL}=${NEW_SYMBOL}" "${OBJ_MEMBER}" &&
    "${AR}" r "${LIB}" "${OBJ_MEMBER}"
)
patch_ret=$?
rm -rf "${PATCH_TMP}"
if [ "$patch_ret" -ne 0 ]; then
    echo "错误: lib patch 失败" >&2
    exit 1
fi
echo "lib patch 完成: ${SYMBOL} -> ${NEW_SYMBOL}"
```

### 工程实际调用（build_bsl600.sh 内）

```sh
HILINK_LIB="${MY_SDK_DIR}/application/samples/hilink/libhilink/libhilinkbtsdk.a"
RISCV_BIN="${MY_SDK_DIR}/tools/bin/compiler/riscv/cc_riscv32_musl_b010/cc_riscv32_musl_fp/bin"

lib_symbol_takeover.sh \
    "${HILINK_LIB}" \
    hilink_bt_link_common.c.o \
    GetErrcodePayload GetErrcodePayload_Legacy \
    "${RISCV_BIN}" \
    "${BACKUP_DIR}"
```

### 要点

- `ar x` 解出符号定义所在的那个 `.o`（用 `ar t <lib>` 查看成员名）
- `objcopy --redefine-sym` 只作用于单 `.o`，定义在哪个 `.o` 就改哪个
- 幂等检查放在最前：已 patch 跳过、符号缺失报警退出
- 必须备份原库，构建完用备份还原（当前工程由 `restore_sdk` trap 统一恢复）

## 四、接管实现示例

`src/HilinkSdkRsp.c`（示意，符号/签名以实际逆向为准）：

```c
extern char *GetErrcodePayload_Legacy(int32_t errcode, uint32_t *outLen);

/** 业务预设的 RSP payload，仅单全局变量；依赖"回调返回后同步取走"的时序 */
static char *g_pendingPayload = NULL;

int32_t HilinkSdkRspSetPayload(const char *payload, uint32_t len) {
    char *copy = (char *)HILINK_BT_Malloc(len + 1); // 用库的内存分配，匹配库的释放
    if (copy == NULL || payload == NULL) {
        return -1;
    }
    memcpy(copy, payload, len);
    copy[len] = '\0';
    g_pendingPayload = copy;
    return 0;
}

char *GetErrcodePayload(int32_t errcode, uint32_t *outLen) {
    char *payload = g_pendingPayload;
    g_pendingPayload = NULL;
    if (payload != NULL && outLen != NULL) {
        *outLen = (uint32_t)strlen(payload);
        return payload;
    }
    return GetErrcodePayload_Legacy(errcode, outLen); // 兜底原实现
}
```

关键点：

- **签名必须与库内原函数完全一致**（逆向确认），错一个参数就是内存踩踏
- **内存所有权**：返回的 payload 由 SDK 释放，必须用库配套的分配函数（如 `HILINK_BT_Malloc`），不能用 `malloc`
- **时序假设**："预设 → 返回后同步取走"必须在同一任务上下文，否则单全局变量会错乱

## 五、验证

```sh
# 固件 ELF 中应同时存在两个符号
riscv32-linux-musl-nm application.elf | grep GetErrcodePayload
# 9012aa28 T GetErrcodePayload        ← 业务实现
# 9011d9d6 T GetErrcodePayload_Legacy ← 库内改名后的原实现

# 构建完成后库应恢复原始状态
riscv32-linux-musl-nm libhilinkbtsdk.a | grep GetErrcodePayload
# T GetErrcodePayload                 ← 仅一个，未被打 patch
```

## 六、失效条件与风险

| 场景 | 是否可行 | 说明 |
|------|---------|------|
| 静态库 `.a` + GNU/LLVM binutils | ✓ | 本方法适用前提 |
| 目标符号为全局 `T`，调用未被内联 | ✓ | 改名后引用变 U |
| LTO / ThinLTO 内联 | ✗ | 符号可能被优化掉，改名失效 |
| 动态库 `.so` | ✗ | 改走运行期 `LD_PRELOAD` / 符号插值 |
| 已生成的最终镜像 | ✗ | 那是二进制 patching，非此链路 |
| 原函数被并发/多实例调用 | ⚠️ | 单全局变量方案错乱，需按调用点传上下文 |

## 七、新工程复用 Checklist

- [ ] `nm` 确认目标符号为全局 `T`，`ar t` 确认所在 `.o` 成员
- [ ] 反汇编确认函数签名、返回值的生命周期（谁分配、谁释放）
- [ ] 确认"预设 → 返回后同步取走"的时序成立
- [ ] 用上述通用脚本打 patch，保留幂等检查与版本保护
- [ ] 业务实现提供同名函数 + 兜底转调 `_Legacy`
- [ ] 固件 ELF 双符号验证、构建后库还原验证
