# stdlib 安全审计清单

本文档列出 xray 内置函数（builtin）和标准库的安全审计项，按风险级别分类。

## 审计范围

| 模块 | 文件 | 方法数 |
|------|------|--------|
| Array | `xarray_builtins.c`, `xvm_builtins.c` | ~15 (push/pop/map/filter/sort/...) |
| String | `xstring_builtins.c`, `xvm_builtins.c` | ~12 (slice/indexOf/split/...) |
| Map | `xmap_builtins.c`, `xvm_builtins.c` | ~8 (get/set/has/delete/...) |
| Set | `xset_builtins.c`, `xvm_builtins.c` | ~6 (add/has/delete/...) |
| Slice | `xslice_builtins.c` | ~15 (各种 slice 操作) |
| StringBuilder | `xstringbuilder_builtins.c` | ~5 (append/toString/...) |
| Enum | `xenum_builtins.c` | ~12 (values/name/ordinal/...) |
| DateTime | `stdlib/datetime/` | 格式化、解析 |
| Regex | `stdlib/regex/` | 编译、匹配 |
| JSON | `xvm_builtins.c` + `xjson.c` | 解析、序列化 |

## 高风险项

### H1: Array(n) 无大小上限

**位置**: `xarray_builtins.c:42`

```c
int n = (int)XR_TO_INT(args[0]);
if (n < 0) { ... }  // 仅检查负数
XrArray *arr = xr_array_with_capacity(xr_current_coro(isolate), n);
```

**风险**: `n` 可以是 `INT_MAX`，导致巨大内存分配。
**建议**: 添加合理上限（如 `XR_MAX_ARRAY_LENGTH`），超出时抛 RangeError。

### H2: 整数截断 — int64 → int 强转

**位置**: 多个 builtin 函数

```c
int n = (int)XR_TO_INT(args[0]);  // XR_TO_INT 返回 int64_t
```

**风险**: 如果值超过 `INT_MAX`，截断后变为负数或随机值。
**建议**: 添加范围检查宏，确保 int64 值在 int 范围内。

### H3: Receiver 类型未验证

**位置**: `xvm_builtins.c` 中的 method handler

```c
static XrValue map_has_handler(..., XrValue receiver, ...) {
    XrMap *map = XR_TO_MAP(receiver);  // 无类型检查
```

**风险**: 如果 dispatch 出错，receiver 类型不匹配会导致类型混淆。
**现状**: dispatch 层保证了类型，但缺乏防御性检查。
**建议**: 在 debug 模式添加 `XR_DCHECK(XR_IS_MAP(receiver), ...)` 断言。

### H4: Regex 回溯攻击 (ReDoS)

**位置**: `stdlib/regex/`

**风险**: 恶意正则表达式可能导致指数级回溯。
**建议**: 考虑添加匹配步数上限或超时机制。

## 中风险项

### M1: Isolate NULL 仅 Debug 检查

**位置**: 所有 builtin 构造函数

```c
XR_DCHECK(isolate != NULL, "xxx: NULL isolate");
// Release 模式下 isolate=NULL 会直接 crash
```

**现状**: 可接受——isolate 由 VM 保证非 NULL。
**建议**: 公共 API 入口点（`xray_` 前缀）应使用 `xray_api_check` 而非 `XR_DCHECK`。

### M2: snprintf 缓冲区大小

**位置**: `xenum_builtins.c` 及其他格式化场景

**现状**: 使用 `snprintf` 而非 `sprintf`，安全。
**建议**: 继续保持，禁止使用 `sprintf`/`strcpy`/`strcat`。

### M3: Map/Set 的并发访问

**位置**: `xmap.c`, `xset.c`

**风险**: 多协程并发读写同一 Map/Set 可能导致数据竞争。
**现状**: xray 通过 channel 通信避免共享，但 `shared` 变量可能暴露。
**建议**: shared Map/Set 需要加锁或使用 copy-on-write。

### M4: JSON 解析深度

**位置**: `xjson.c` 解析器

**风险**: 深度嵌套的 JSON 可能导致栈溢出。
**建议**: 添加最大嵌套深度限制（如 256 层）。

## 低风险项

### L1: 字符串操作的编码安全

**现状**: xray 字符串是 UTF-8，slice/indexOf 按字节操作。
**建议**: 文档说明字节 vs 字符 vs 码点语义差异。

### L2: 数组 sort 稳定性

**现状**: 使用标准库 qsort。
**建议**: 确认 comparator 回调异常不会导致 qsort 状态损坏。

### L3: GC 安全 — builtin 中的临时指针

**现状**: 大部分 builtin 在单次 C 调用内完成，GC 不会触发。
**建议**: 长时间运行的 builtin（如 Array.map with callback）需确保 GC root 正确标记。

## 审计检查清单（逐模块）

### 每个 builtin 函数必须满足：

- [ ] **NULL isolate 检查**: 至少有 `XR_DCHECK`（公共 API 用 `xray_api_check`）
- [ ] **参数数量检查**: `nargs`/`argc` 验证，不足时返回合理默认值
- [ ] **参数类型检查**: `XR_IS_INT`/`XR_IS_STRING` 等验证
- [ ] **整数范围检查**: int64 → int 转换前验证范围
- [ ] **大小上限**: 数组/字符串创建有合理上限
- [ ] **无 sprintf/strcpy/strcat**: 仅使用 `snprintf`/`memcpy`
- [ ] **GC 安全**: 长操作中的 GC root 正确
- [ ] **并发安全**: shared 数据的访问保护

### 已通过审计的模块

| 模块 | NULL 检查 | 参数验证 | 类型检查 | 范围检查 | GC 安全 |
|------|-----------|----------|----------|----------|---------|
| Array construct | ✅ DCHECK | ✅ | ✅ | ⚠️ 无上限 | ✅ |
| String construct | ✅ DCHECK | ✅ | ✅ | N/A | ✅ |
| Map handlers | ❌ 无 | ✅ | ❌ receiver | N/A | ✅ |
| Set handlers | ❌ 无 | ✅ | ❌ receiver | N/A | ✅ |
| Slice builtins | ✅ DCHECK | ✅ | ✅ | ⚠️ 需审查 | ✅ |
| Enum builtins | ✅ DCHECK | ✅ | ✅ | ✅ | ✅ |

## 优先修复建议

1. **H2 整数截断**: 添加 `XR_INT_TO_INT32_SAFE(val, &out)` 安全转换宏
2. **H1 数组上限**: 添加 `XR_MAX_ARRAY_LENGTH` (如 2^28)
3. **H3 Receiver 断言**: 在所有 method handler 添加 debug 断言
4. **M4 JSON 深度**: 解析器添加深度计数器
