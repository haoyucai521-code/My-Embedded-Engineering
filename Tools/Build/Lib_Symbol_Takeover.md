# 链接期接管闭源库函数（LIB SYMBOL TAKEOVER）

> 适用：闭源 SDK 静态库（`.a`）中的某个函数行为固定、无源码可改，但业务需要接管其返回值/行为。
> 本文档以 R157H 工程接管 HiLink SDK `GetErrcodePayload`（custom 通道 RSP payload）为例，沉淀**从"能不能改"到"定位函数"到"推断签名"到"脚本接管"的完整方法论**。

---

## 一、可行性判断：这个库能不能改？

动手逆向前先判断，避免白干。

### 1.1 三个硬条件

| # | 条件 | 证据 | 不满足的后果 |
|---|------|------|-------------|
| 1 | 链接形态是 `.a` 静态库 | CMakeLists 里 `WHOLE_LINK=true` 引入的库 | `.so` 走 LD_PRELOAD，镜像走二进制 patch，都不归此链路 |
| 2 | 目标符号是**全局**定义（大写 `T`） | `nm lib.a \| grep 符号` | `t`（local）改名无效 |
| 3 | 有**未解析引用**（`U`）未被内联 | 同一命令里看到多个 `U` | LTO/内联掉就改名无用 |

### 1.2 一眼判断的符号表

```
$ riscv32-linux-musl-nm libhilinkbtsdk.a | grep -i err
00000000 T GetErrcodePayload       ← 大写 T = 全局定义，可被外部接管
         U GetErrcodePayload       ← U = 调用点，链接期解析，会绑到我们的实现
00000000 t SendAdvStartErrGapEvent ← 小写 t = local，改不了
```

**大写 `T`（可接管）vs 小写 `t`（改不了）是第一个分水岭。**

### 1.3 不可改的信号（换思路）

- 只有 `t` 没有 `T`：local 符号，改名无效
- 完全无符号（`strip -s` 后的 release 库）：符号表没了，只能按地址硬找，风险陡增
- 有 `T` 但无 `U`：唯一调用点被内联，改定义不影响行为
- 多成员重复定义：`ar x` 只能解出一个，需确认链接顺序谁生效

---

## 二、逆向准备：工具与命令

### 2.1 工具链选型

**必须用 SDK 自带交叉工具链的 Linux 版**（通常和编译器同目录），不要用系统工具：

```
<SDK>/tools/bin/compiler/riscv/cc_riscv32_musl_xxx/cc_riscv32_musl_fp/bin/
    riscv32-linux-musl-{nm,objdump,ar,objcopy,readelf}
```

三个常见坑（本工程都踩过）：

| 坑 | 现象 | 解法 |
|----|------|------|
| 系统 `objdump` | `can't disassemble for architecture UNKNOWN` | 它不认 RISC-V，换交叉工具链 |
| Windows 版工具 | exit 127 / 缺 DLL | 在 WSL 里跑 Linux 版 |
| 相对路径在 `cd` 后失效 | `No such file or directory` | 全程用绝对路径 |

### 2.2 命令速查

| 目的 | 命令 |
|------|------|
| 列成员 | `ar t lib.a` |
| 解出成员 | `ar x lib.a member.c.o` |
| 看符号（定义/引用/类型） | `nm [-A] lib.a` |
| 反汇编代码 | `objdump -d x.c.o` |
| 看字符串段 | `objdump -s -j .rodata x.c.o` |
| 看重定位（call 到哪） | `objdump -r x.c.o` |
| 二进制里搜字符串 | `grep -aob 'keyword' lib.a` |

---

## 三、定位目标函数：四路并进

闭源库无源码，逆向是**从输出特征反推源头**，不是从头读逻辑。四条路互相印证。

### 3.1 路 1：字符串反查（最常用）

函数的固定输出必然以明文躺在 rodata 里。本例 RSP payload 固定 `{"errcode":%d}`：

```
$ grep -aob 'errcode' libhilinkbtsdk.a
156576:errcode
208210:errcode
```

拿到 rodata 偏移后，反汇编搜索**谁读取了这个地址**（xref），即找到生成它的函数 `GetErrcodePayload`。

### 3.2 路 2：符号表直搜

知道目标行为可能叫某名字（payload / errcode / report），直接 `nm | grep` 关键字。

### 3.3 路 3：调用链顺藤摸瓜

从**业务入口**沿调用链往下走（本例从接收链路 `HILINK_BT_ProcessBtData → HILINK_BT_ProcessCmd → PutServiceCustomData`），一路看 `call` 目标，直到找到 payload 来源。

### 3.4 路 4：行为日志对照

App 日志里能看到的固定输出（`responseData: {"errcode":0}`）是逆向的路标——去找生成这个字符串的函数，比漫无目的翻库快得多。

> 本例四条路汇聚到同一个函数：`GetErrcodePayload`，定义在 `hilink_bt_link_common.c.o`。

---

## 四、签名推断：四类证据

**签名错一个参数就是内存踩踏，必须靠证据交叉印证，禁止猜测。**

### 4.1 RISC-V 调用约定速览

| 约定 | 含义 |
|------|------|
| 参数 | 前 8 个整数/指针参数按序放 `a0`~`a7` |
| 返回值 | 返回时 `a0` 装返回值 |
| `s` 寄存器 | callee-saved，被调函数用前必须保存 |

### 4.2 证据 A：函数体对参数寄存器的使用 → 参数个数

```
00000000 <GetErrcodePayload>:
   0:  push {ra,s0-s5},-32   ← 保存 s0-s5，函数内部有局部逻辑
   2:  bnez a1,a <.L18>      ← 判断 a1 → 至少有第 2 个参数
```

`bnez a1` 说明 `a1` 不是摆设，函数要根据它分支。

### 4.3 证据 B：rodata 字符串引用 → 参数类型

函数体引用 `{"errcode":%d}`，`%d` 说明**第一个参数是被格式化的整数** → `int32_t errcode`。

### 4.4 证据 C：向指针写值 → 出参

反汇编里出现 `store ? → (a1)`，即往 `a1` 指向的地址写数据，说明 `a1` 是**出参指针**。本例写入的是 `strlen(payload)` → `uint32_t *outLen`。

### 4.5 证据 D：调用者数据流（最可靠，定案）

签名最终靠调用者定死——看 call 前装什么、call 后怎么用返回值：

```
调用点之前：  li a0, <errcode>       ← 第1参装整数
              la a1, <局部长度变量>   ← 第2参装指针
              call GetErrcodePayload
调用点之后：  mv s?, a0              ← a0 被保存下来当 buffer 用
              lw t?, (a1)            ← 从 a1 指向处读回长度
```

- 第 1 参装载**整数值** → `int32_t`
- 第 2 参装载**指针**、调用后**读回** → `uint32_t *`（出参）
- 返回值被当**数据缓冲指针**用（传给后续编码/发送函数）→ `char *`

### 4.6 推理链完整拼图

```
char *GetErrcodePayload(int32_t errcode, uint32_t *outLen)
│        │                │              └─ a1: 出参指针，函数向其写 strlen(payload)  [证据 C/D]
│        │                └─ a0: 被 {"errcode":%d} 格式化的 int                    [证据 B/D]
│        └─ a0: 返回 payload 字符串指针，调用者当 buffer 用                         [证据 D]
└─ 返回类型 char*
```

A/B/C 至少两条相符 + D 吻合，签名锁死。

---

## 五、实操：从 .a 到反汇编

### 5.1 解出成员

```
$ ar t libhilinkbtsdk.a | grep link_common
hilink_bt_link_common.c.o
$ ar x libhilinkbtsdk.a hilink_bt_link_common.c.o
$ file hilink_bt_link_common.c.o
ELF 32-bit LSB relocatable, UCB RISC-V ... not stripped
```

`relocatable + not stripped` = 符号表完整，正是能改的根本原因。

### 5.2 三种 objdump 视角

| 视角 | 命令 | 回答什么问题 |
|------|------|-------------|
| 代码 | `objdump -d x.o` | 函数逻辑、寄存器使用、call 目标 |
| 字符串 | `objdump -s -j .rodata x.o` | 格式串、错误码，反推参数类型 |
| 重定位 | `objdump -r x.o` | 未定义符号引用在哪、什么类型 |

### 5.3 `.o` 与最终 ELF 的差别

- **`.o` 反汇编**：符号引用是占位（`call <GetErrcodePayload>` 未解析），适合看结构
- **最终 ELF 反汇编**：地址已定、引用已解析，适合确认**链接后**谁调谁、函数完整形态（冷热路径合并后）

签名确认建议以 `.o` 为主、ELF 为校验；LTO 折叠严重时反着来。

---

## 六、坑与排障

### 6.1 冷热路径（最容易误判）

```
00000000 <GetErrcodePayload>:
   0:  push {ra,s0-s5},-32
   2:  bnez a1,a <.L18>      ← 冷路径被放别处
   4:  li s0,0
   6:  mv a0,s0
   8:  popret {ra,s0-s5},32  ← 函数体看着"很短"
```

`.L18` 冷路径在函数体之外，单段 dump 只看到热路径 → 误判"这是个 stub"。看完整实现要配合 `.L18` 标签或最终 ELF。

### 6.2 LTO trampoline

调用点若被 LTO 折叠成 stub，`objdump -d` 看不到直接 `call GetErrcodePayload`。此时：
- 用 `objdump -r`（重定位）看未解析引用
- 或直接在最终 ELF 上反汇编

### 6.3 符号被 strip

release 库可能 `strip -s` 掉符号表 → `nm` 空。只能靠字符串 xref + 地址推算，风险陡增，需更强的证据链。

### 6.4 工具链坑（见 2.1）

---

## 七、交叉印证清单

签名推断完成后，动手 patch 前逐项确认：

- [ ] 签名与 SDK 公开头文件/文档一致（`ble_cfg_net_api.h` 等，如存在）
- [ ] **内存所有权**：返回值谁分配、谁释放。本例 SDK 用 `HILINK_BT_Free` 释放，故接管实现必须用 `HILINK_BT_Malloc`，不能用 `malloc`
- [ ] **时序**：调用点上下文是"回调返回后同步取走"还是异步/并发？后者单全局变量方案会错乱
- [ ] **行为**：接管实现输出与 App 日志期望一致（格式、字段名、单位）
- [ ] 全部调用点语义一致：同一个符号被多处调用时，一个全局接管是否够

---

## 八、三步法

```
1. 定位符号：nm 确认全局 T 定义，反汇编确认函数签名与调用时序
2. 制造缺口：objcopy 改名，ar 重打回库
3. 填缺口 + 兜底：提供同名新实现，必要时转调旧符号
```

---

## 九、打 patch：完整流程与原理

### 9.1 原理：改名如何制造缺口

```
原始库：
  hilink_bt_link_common.c.o : T GetErrcodePayload        ← 定义
  其他成员 (cmd.c.o 等)      : U GetErrcodePayload        ← 调用点

patch 后库：
  hilink_bt_link_common.c.o : T GetErrcodePayload_Legacy  ← 定义改名
  其他成员                   : U GetErrcodePayload        ← 调用点不变
```

改名后**库里不再有 `T GetErrcodePayload`**。链接固件时：

```
所有 U GetErrcodePayload ──→ 必须由外部提供同名 T 才能解析
        │                          └─ 我们的 HilinkSdkRsp.c 提供 T GetErrcodePayload
        ▼                                      │
   绑定到我们的实现 ←───────────────────────────┘
        │
        └─ 实现内 call GetErrcodePayload_Legacy ──→ 绑定到库内原实现
```

三个关键认知：

- `--redefine-sym` 只改**这一个 `.o`** 的符号表字符串，同时改"定义"和"这个 `.o` 内部的引用"；其他成员的 `U` 引用不在此 `.o` 内，天然不受影响
- **机器码一个字节不动**，原函数体以新名字继续可调（这就是兜底的来源）
- 我们没有改任何一个调用点，只是把定义挪走，**迫使链接器找外部同名定义**

### 9.2 三步物理操作

```sh
$ ar x lib.a hilink_bt_link_common.c.o   # 1. 解出定义所在的 .o
$ objcopy --redefine-sym GetErrcodePayload=GetErrcodePayload_Legacy \
      hilink_bt_link_common.c.o          # 2. 符号表改名
$ ar r lib.a hilink_bt_link_common.c.o   # 3. 替换回库（r = replace）
```

`ar r` 的语义是 **replace**——同名成员直接替换；不能用 `ar q`（追加），否则库里出现两份同名成员。

### 9.3 脚本设计要点

| 设计 | 原因 |
|------|------|
| 只解出**定义所在**的 `.o` | 改名只对"定义 + 同 `.o` 内引用"有意义；其他成员的 `U` 引用由链接期解析，不在此 `.o` 内 |
| 用 `ar r` 替换 | `ar q` 追加会产生重复成员；`r` 才是替换 |
| 幂等检查放最前 | 上次构建残留未还原时跳过，避免 `_Legacy` 再被改名成 `_Legacy_Legacy` |
| grep 锚定 `$` 结尾 | `grep 'T GetErrcodePayload$'` 不会误匹配到 `T GetErrcodePayload_Legacy`（子串） |
| 符号缺失**必须 fail** | patch 没生效时，我们的实现和库内原定义会"多重定义"链接错误，行为不可控 |
| 警告而非静默 | SDK 升级改名后，nm 找不到符号 → 明确提示人工确认，不静默改变行为 |

### 9.4 备份与还原

patch 是破坏性的，且 SDK 目录是共享的，不能残留：

- **备份**：patch 前 `cp -p lib.a BACKUP_DIR/<同路径>`
- **还原**：构建脚本 `trap restore_sdk EXIT`，脚本无论成败退出时把备份拷回
- **节奏**：每次构建**临时打 patch → 编译 → 构建完还原**，保证 SDK 目录始终干净

### 9.5 完整脚本

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

### 当前工程实际调用

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

- `ar x` 解出**符号定义所在**的那个 `.o`（`nm -A` 直接给出）
- `objcopy --redefine-sym` 只作用于单 `.o`，定义在哪个 `.o` 就改哪个
- 幂等检查放最前：已 patch 跳过、符号缺失报警退出
- 必须备份原库，构建完还原（当前工程由 `restore_sdk` trap 统一恢复）

---

## 十、接管实现：运行时序与关键点

### 10.1 运行时序（为什么单全局变量安全）

一次 exceptionInfo 查询的完整时序，**全部在同一任务上下文同步执行**：

```
1. SDK 回调业务层  HilinkOnSdkCustomData({"sid":"exceptionInfo",...})
2. 业务组好 dataObj → hilinkSetExceptionInfoRspPayload(dataObj)
3.    → HilinkSdkRspSetPayload(rspStr, rspLen)
4.        → HILINK_BT_Malloc(len+1) + memcpy，存 g_pendingPayload
5. 回调返回
6. SDK 同步调用 GetErrcodePayload(errcode, &outLen)  ← 我们的实现
7.    取走 g_pendingPayload 并清零，*outLen = strlen(payload)
8. SDK 用 payload+outLen 编码并发送 RSP 帧
9. SDK 调 HILINK_BT_Free(payload) 释放
```

第 3 步和第 6 步之间**没有任务切换、没有其他消息插入** → 单全局变量 `g_pendingPayload` 安全。
若 SDK 改成"异步取 payload"或并发调用，此方案失效，需改为按消息上下文存值（队列/句柄绑定）。

### 10.2 为什么必须用 `HILINK_BT_Malloc`

返回值 payload 的生命周期归 SDK：第 9 步由 `HILINK_BT_Free` 释放。
若分配用 libc `malloc`、释放用 `HILINK_BT_Free`，两个堆不同 → 崩溃。
**分配/释放必须来自同一内存接口**，这是本方案最容易踩的坑。

### 10.3 为什么取走即清零

`g_pendingPayload = NULL` 保证"一次性"语义：
- 本消息的 payload 只被本条 RSP 消费一次
- 若不清零，下一条消息（PUT、配网等）的 `GetErrcodePayload` 会拿到上一条的残留数据，错发到别的响应里

### 10.4 兜底回退

`g_pendingPayload == NULL`（业务未预设，如其他 custom 消息）：
```
return GetErrcodePayload_Legacy(errcode, outLen);  // 保持库内原始行为
```
→ 无关消息的 RSP 仍是 `{"errcode":N}`，本方案对它们**零影响**。

### 10.5 签名必须与库内完全一致

extern 声明、定义、调用约定（ABI）都必须和库内原函数相同：
```c
char *GetErrcodePayload(int32_t errcode, uint32_t *outLen);
```
参数个数/类型/返回值错一个，链接可能通过（同名符号）但**运行时寄存器错乱 → 内存踩踏**。

### 10.6 覆盖保护

`SetPayload` 在旧 payload 未被取走时再次被调用（正常时序不会发生，防御性）：
- 打印 `SYS_LOG` 警告后覆盖
- 旧内存按泄漏处理（时序正确时走不到）

### 10.7 失败路径

- `SetPayload` 入参 NULL / len==0 / `HILINK_BT_Malloc` 失败 → 返回 -1，业务忽略
- 效果：`g_pendingPayload` 保持 NULL，`GetErrcodePayload` 回退 Legacy
- 不会发出半截 payload，RSP 仍是合法的 `{"errcode":N}`

### 10.8 实现示例

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

关键点回顾：

- **签名必须与库内原函数完全一致**（逆向确认），错一个参数就是内存踩踏
- **内存所有权**：返回的 payload 由 SDK 释放，必须用库配套的分配函数（如 `HILINK_BT_Malloc`），不能用 `malloc`
- **时序假设**："预设 → 返回后同步取走"必须在同一任务上下文，否则单全局变量会错乱

---

## 十一、验证

```sh
# 固件 ELF 中应同时存在两个符号
riscv32-linux-musl-nm application.elf | grep GetErrcodePayload
# 9012aa28 T GetErrcodePayload        ← 业务实现
# 9011d9d6 T GetErrcodePayload_Legacy ← 库内改名后的原实现

# 构建完成后库应恢复原始状态
riscv32-linux-musl-nm libhilinkbtsdk.a | grep GetErrcodePayload
# T GetErrcodePayload                 ← 仅一个，未被打 patch
```

---

## 十二、失效条件与风险

| 场景 | 是否可行 | 说明 |
|------|---------|------|
| 静态库 `.a` + GNU/LLVM binutils | ✓ | 本方法适用前提 |
| 目标符号为全局 `T`，调用未被内联 | ✓ | 改名后引用变 U |
| LTO / ThinLTO 内联 | ✗ | 符号可能被优化掉，改名失效 |
| 动态库 `.so` | ✗ | 改走运行期 `LD_PRELOAD` / 符号插值 |
| 已生成的最终镜像 | ✗ | 那是二进制 patching，非此链路 |
| 原函数被并发/多实例调用 | ⚠️ | 单全局变量方案错乱，需按调用点传上下文 |

---

## 十三、新工程复用 Checklist

- [ ] **可行性**：`nm` 确认全局 `T` + 有 `U` 引用，链接形态是 `.a`
- [ ] **定位**：字符串反查 / 符号表 / 调用链 / 行为日志 四路交叉
- [ ] **签名**：四类证据（参数寄存器 / rodata / 指针写回 / 调用者数据流）交叉，`ar x` 确认定义成员
- [ ] **所有权**：返回值谁分配、谁释放，接管实现用库配套分配函数
- [ ] **时序**：确认"预设 → 返回后同步取走"成立
- [ ] **脚本**：用第九节通用脚本打 patch，保留幂等检查与版本保护
- [ ] **接管**：业务实现提供同名函数 + 兜底转调 `_Legacy`
- [ ] **验证**：固件 ELF 双符号、构建后库还原
