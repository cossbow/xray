# Bug 模式分类

从历史 bug 修复中提炼的可复用模式。每个 pattern 包含：触发条件、典型案例、防御手段。

---

## Pattern 1: 裸构造绕过工厂函数

**描述**：手动赋值结构体字段时遗漏关键字段，绕过了工厂函数内置的完整性保证。

**典型案例**：
- `xr_jit_invoke_direct` 中手动构建 `XrValue`，设置 `descriptor=0` 清除了 `heap_type`，
  导致 `xr_value_to_instance` 返回 NULL → SIGSEGV → crash handler 无限信号循环
- Commit: `8978e1ab`

**触发条件**：任何绕过 `xr_value_from_xxx()` / `xr_make_ptr_val()` 的 XrValue 构造

**防御手段**：
- 统一使用工厂函数 `xr_value_from_instance()` / `xr_make_ptr_val()`
- Lint 脚本检测 `.descriptor = 0` 后跟 `.tag = XR_TAG_PTR` 的模式
- 合法的裸构造加 `// raw-ok` 白名单注释

**检测脚本**：
```bash
grep -rn '\.descriptor = 0' src/ --include="*.c" | grep -v 'xvalue.c\|xvalue.h' | grep -v '// raw-ok'
```

---

## Pattern 2: 快速路径与慢速路径不一致

**描述**：优化的快速路径跳过了慢速路径中的必要设置步骤，导致状态不完整。

**典型案例**：
- `XIR_CALL_KNOWN` 快速路径跳过了 `call_closure` 的 closure 设置
- Commit: `cb25f619`

**触发条件**：任何有 fast-path / slow-path 分支的代码

**防御手段**：
- 将共享设置逻辑抽取为独立函数，fast/slow path 都调用
- Code review 重点检查分支路径的共享状态是否一致
- 添加断言验证 fast-path 出口状态与 slow-path 一致

---

## Pattern 3: 跨模块接口契约隐式依赖

**描述**：模块 A 假设模块 B 一定设置了某个字段，但没有显式验证，B 在某些路径下未设置。

**典型案例**：
- JIT builder 假设 upval 的 `type_info` 已被编译器 Phase 1 设置，但 mutable capture 路径未设置
- Commit: `cb25f619`

**触发条件**：跨模块调用，尤其是编译器→JIT、JIT→VM 边界

**防御手段**：
- 在接口入口加 `XR_DCHECK` 验证前置条件
- 文档化每个跨模块接口的契约（见 `invariants.md`）

---

## Pattern 4: Off-by-one / 硬编码偏移错误

**描述**：数组索引或寄存器偏移使用硬编码常量，当底层布局变更时未同步更新。

**典型案例**：
- GC `callee_saved_offsets` 数组中 `alloc_regs[15]` 实际对应 x21 而非 x20，偏移错一位
- Commit: `cb25f619`

**触发条件**：手动编码的寄存器映射、硬编码数组偏移

**防御手段**：
- 用 `static_assert` 或运行时断言验证映射关系
- 尽量用命名常量代替硬编码索引
- 在 debug 构建中验证偏移的一致性

---

## Pattern 5: GC 安全点 bitmap 包含死亡指针

**描述**：safepoint bitmap 中标记了已死亡的 PTR vreg，GC 扫描时访问无效指针。

**典型案例**：
- `record_safepoint` 未检查 vreg 在该点是否存活，将已死亡的 PTR 纳入 spill bitmap
- Commit: `cb25f619`

**触发条件**：safepoint 记录逻辑、GC root 扫描

**防御手段**：
- safepoint 记录时检查 vreg 的 liveness（`vreg->last_use >= current_pos`）
- ASan 可捕获 GC 扫描时的越界访问

---

## Pattern 6: JIT crash handler 信号无限循环

**描述**：JIT crash handler 捕获非 JIT 代码的 SIGSEGV，错误地当作 guard page fault 处理，
disarm 后重试，但重新执行时再次触发 SIGSEGV，形成无限循环。

**典型案例**：
- VM 解释器在 `OP_GETFIELD` 中 SIGSEGV（因 Pattern 1 导致 NULL 解引用），
  crash handler 的 `try_handle_guard_page_fault` 将其视为 C 代码中的 guard page fault，
  disarm 并返回 true，导致无限重试
- 表现为测试 timeout

**触发条件**：JIT → VM 解释器调用路径中的任何 SIGSEGV

**防御手段**：
- crash handler 应区分 guard page fault 和普通 SIGSEGV
- 考虑在 `try_handle_guard_page_fault` 中加重试计数器
- 用 `XRAY_NO_JIT_CRASH_HANDLER=1` 环境变量调试 timeout 问题

---

## Pattern 7: fd 重用导致 PollDesc 残留

**描述**：`close(fd)` 后未调用 `xr_netpoll_close`，内核回收 fd 后分配给新连接，
但 `xr_netpoll_open` 返回旧的 PollDesc（stale），且 kqueue 注册在 close 时已被删除。
新连接永远收不到事件 → 协程永久阻塞。

**典型案例**：
- HTTP 服务器连接 handler 中 `close(fd)` 前未清理 netpoll
- 表现为第二个连接永久挂起

**触发条件**：任何 fd 关闭后重新被 accept 分配的场景

**防御手段**：
- 关闭 fd 前**必须**调用 `xr_netpoll_close`
- 在 netpoll 模块中加断言：`open` 时检查旧 PollDesc 是否已清理

---

## Pattern 8: raw=0 歧义（int(0) vs null）

**描述**：JIT 运行时用 `raw==0` 启发式判断 NULL，但 `int(0)` 的 raw payload 也是 0。
当 slot type 为 UNKNOWN 时，int(0) 被误判为 null。

**典型案例**：
- `jit_value_from_tag` 的 UNKNOWN 路径：`raw==0 → NULL`
- `OP_LOADNULL` 使用 `VTAG_TAGGED` 导致 bitmap 编码为 UNKNOWN(0xF)

**修复**：
- 引入 `VTAG_NULL`，null 值通过精确 tag 携带（bitmap nibble=0）
- UNKNOWN+raw==0 → 改为 I64(0)（int(0) 远比 null 常见）
- PTR+raw==0 → 保持 NULL（空指针就是 null）

**防御手段**：
- 所有 JIT 类型标记应尽量精确（VTAG_I64/F64/NULL/BOOL），避免 fallback 到 UNKNOWN
- 启发式判断应有文档说明其边界条件

---

## Pattern 9: 协程跨线程迁移导致 TLS stack canary 失配

**适用范围**：native-stackful 协程实现风险；当前 VM 协程是 native-stackless，不通过切换 C 栈恢复。

**描述**：stackful 协程在线程 A 创建（TLS stack canary = A），被 work stealing 迁移到线程 B 执行，
函数返回时 stack canary 检查用线程 B 的 TLS 值 → `__stack_chk_fail`。

**触发条件**：stackful 协程 + work stealing 调度器

**防御手段**：
- 协程 C 栈函数加 `__attribute__((no_stack_protector))`
- 或者禁止 stackful 协程跨线程迁移

---

## Pattern 10: RA caller-saved 寄存器范围不完整

**描述**：JIT 寄存器分配器只标记部分寄存器为 caller-saved（call-clobbered），
其余 caller-saved 寄存器被 RA 当作 callee-saved 使用，跨 CALL 时未 spill。

**典型案例**：
- 只标记 X1-X7 为 clobbered，X8-X15 也是 ARM64 caller-saved 但未标记
- RA 将 live-across-call 的 vreg 分配到 X8-X15 → 调用后值被覆盖

**防御手段**：
- RA 的 fixed intervals 必须覆盖**所有** caller-saved 寄存器
- 用 `static_assert` 验证 fixed interval 数量与 ABI 一致

---

## Pattern 11: PHI dst bc_slot 被 FORLOOP 重定位

**描述**：`builder_set_slot(slot, new_vreg)` 会将 `slot_map[slot]` 原 vreg 的 `bc_slot` 重定位，
但如果原 vreg 是 PHI dst，其 `bc_slot` 被改变会导致 OSR stub 加载错误的 values[] 偏移。

**触发条件**：FORLOOP 指令 + PHI 节点 + OSR entry

**防御手段**：
- `builder_set_slot` 应检查被替换的 vreg 是否为 PHI dst，如果是则不重定位
- 或 PHI dst 的 bc_slot 设为不可变

---

## Pattern 12: deopt_id=UINT32_MAX 无匹配 recovery entry

**描述**：JIT helper 遇到特殊情况（如 yieldable C 函数）时写入 `deopt_id=UINT32_MAX`，
但 deopt recovery 表中没有 UINT32_MAX 对应的 entry → 整个函数从头重新执行。

**典型案例**：
- `xr_jit_invoke_method` 遇到 yieldable 函数时设 `deopt_id=UINT32_MAX`

**修复**：builder 为每个 OP_INVOKE 创建 deopt snapshot，将有效 deopt_id 打包到 encoded 参数中。

**防御手段**：
- 所有设置 `deopt_id=UINT32_MAX` 的路径都应有对应的 recovery 机制
- deopt recovery 时对 UINT32_MAX 特殊处理或断言

---

## Pattern 13: 枚举值为 0 导致哨兵检查失效

**描述**：C 枚举中某个有意义的值恰好为 0，而代码用 `== 0` 作为"未匹配"哨兵检查，
导致该枚举值对应的逻辑被跳过。

**典型案例**：
- `xa_narrow_by_typeof`：`XR_KIND_INT == 0`，`target_flags == 0` 的早返回导致 typeof 窄化失效
- 修复：`uint32_t target_flags` 改为 `int target_kind = -1`，用 `< 0` 判断未匹配

**防御手段**：
- 枚举中避免有意义的值为 0，或用 `-1` / `UINT32_MAX` 作为哨兵值
- 哨兵检查应明确为 `sentinel == INVALID_VALUE`，不依赖与零比较

---

## Pattern 14: 隐式数据引用对 liveness/regalloc 不可见

**描述**：数据存储在指令 `args[]` 之外的辅助结构中（如 pool、side table），
regalloc/liveness 只遍历 `args[]`，辅助结构中引用的 vreg 被提前释放。

**典型案例**：
- Call arg pool：参数存在 `XirFunc.call_arg_pool` 而非 STORE_CORO 指令
- regalloc 不知道 pool 中 vreg 仍活跃，寄存器被分配给其他 vreg，调用参数损坏

**防御手段**：
- 新增辅助数据结构引用 vreg 时，必须同步更新 liveness 和 regalloc
- IR 验证 pass 中检查所有 vreg 引用是否被 liveness 覆盖

---

## Pattern 15: 多 Pass 分析器 scope 结构不一致

**描述**：编译器多个 Pass 独立创建 scope 树，但某些 AST 节点在不同 Pass 中
scope 创建策略不同，导致 scope 树结构不一致，变量查找失败。

**典型案例**：
- Pass 1 函数体不创建 block scope，Pass 3 `xa_visit_block_stmt` 总是创建
- for-in 的循环变量在 Pass 1 中注册到 function scope 的直接子 scope
- Pass 3 中多了一层 block scope，for-in scope 变成"孙子"，变量 `i` 找不到
- 修复：Pass 3 函数体直接遍历 body statements，跳过 block scope 创建

**防御手段**：
- 多 Pass 分析器的 scope 创建逻辑应统一封装，避免各 Pass 独立实现
- 添加断言验证 Pass 间 scope 树的 depth 一致性

---

## Pattern 16: 嵌套 JIT deopt 未处理（DEOPT_MARKER 泄漏）

**描述**：`--jit-force` 下所有函数被 JIT，回调函数（map/filter lambda）deopt 返回
`DEOPT_MARKER`，外层 JIT 代码未检查 marker，当作普通指针/值使用导致 SIGSEGV。

**触发条件**：`--jit-force` + 嵌套函数调用（回调、高阶函数）

**防御手段**：
- CALL 指令后生成 deopt marker 检查
- 或在 JIT helper 返回路径中统一处理 marker 传播

---

## Pattern 17: 返回值类型丢失（float 被误判为 PTR）

**描述**：无返回类型注解的函数，JIT 重构返回值时用 raw 位模式启发式推断类型。
float 的 IEEE754 位模式可能被误判为有效堆指针（PTR）。

**典型案例**：
- `return 3.14` → IEEE754 `0x40091EB851EB851F` 看起来像合法地址 → 误判为 PTR

**防御手段**：
- RET 指令在 codegen 中保存返回值 tag
- 或从 TFA 推导返回类型，避免依赖位模式启发式

---

## Pattern 18: INT64_MIN 取负的未定义行为

**描述**：`-val` 对 `INT64_MIN` 是未定义行为（二进制补码溢出）。

**典型案例**：
- `xstrbuf_append_int` 中 `-val` 用于获取绝对值
- 修复：先转为 `uint64_t` 再取负

**防御手段**：
- 整数取负前检查是否为最小值，或直接用无符号类型
- 启用 `-fsanitize=undefined` 检测此类问题

---

## Pattern 19: Padding Oracle — 非 constant-time 比较

**描述**：加密模块的 padding 验证或 timing-safe 比较使用了 early-return 或
条件分支，导致执行时间依赖于数据内容，可被 padding oracle 攻击利用。

**典型案例**：
- `crypto.c`：padding 验证循环遇到错误字节立即 break
- `timingSafeEqual`：长度不同时直接 return false，泄露长度信息
- 修复：固定 16 次迭代 + 位运算积累结果；长度不同仍比较所有字节

**防御手段**：
- 密码学比较必须使用 constant-time 实现（固定迭代次数 + 位运算）
- 不得有依赖秘密数据的 early-return 或条件分支

---

## Pattern 20: 缓冲区越界读（off-by-one / 不安全指针算术）

**描述**：解析函数假设输入以 NULL 结尾或长度足够，未做边界检查，
导致越界读取。

**典型案例**：
- `json.c`：`xr_json_parse_from_cstr` 直接对 `json_str[len]` 写 `\0`（可能越界）
- `json.c`：surrogate pair 解析未检查剩余 6 字符是否可读
- `compress.c`：`xr_gzip_original_size` 边界检查 `len < 8` 应为 `len < 18`
- 修复：始终拷贝输入，增加边界检查

**防御手段**：
- 解析函数总是先验证 remaining length >= expected
- 使用 `XR_CHECK(offset + needed <= len)` 断言

---

## Pattern 21: 递归无深度限制导致栈溢出

**描述**：递归处理数据结构时无深度限制，自引用或超深嵌套对象导致栈溢出。

**典型案例**：
- `json.c`：`stringify_value` 递归序列化无深度检查
- 修复：添加 `depth` 参数，超过 `JSON_MAX_DEPTH` 时返回错误

**防御手段**：
- 递归函数必须带 depth 参数和上限检查
- 或改为迭代 + 显式栈实现

---

## Pattern 22: class_free 不完整（内存泄漏）

**描述**：复杂对象的 `free` 函数遗漏部分字段的释放，导致内存泄漏。

**典型案例**：
- `xr_class_free`：未释放 `static_field_values`、`interfaces`、`itable`、
  `abstract_methods`、`reflect_cache` 等字段
- Reflect wrapper GC 类型设为 `XR_TINSTANCE`，GC 尝试遍历不存在的 field 布局 → 崩溃
- 修复：补全所有字段释放；Reflect wrapper 改为 `XR_TBLOB`

**防御手段**：
- 每个 `_new` / `_create` 函数有对应的 `_free`，且字段一一对应
- ASan 的 leak sanitizer 可检测此类遗漏

---

## Pattern 23: 接口检查不遍历继承链

**描述**：`implements_interface` 只检查当前类，不向上查找父类实现的接口。

**典型案例**：
- `xr_class_implements_interface_fast`：只在 `klass->interfaces[]` 中搜索
- 父类实现了接口，子类继承后却检查失败
- 修复：添加 `while (klass)` 循环沿继承链查找

**防御手段**：
- 涉及类型层次结构的查找应始终考虑继承链
- 为接口检查添加覆盖父类的测试用例

---

## Pattern 24: CALL_C helper 返回 int64_t 但未设置 call_result_tag

**描述**：JIT `call_c_stub` 总是将 x1 寄存器存储到 `call_result_tag`，然后 codegen 将其复制到
`runtime_tags[bc_slot]`。当 C helper 返回 `int64_t`（而非 `XrJitResult`）时，x1 是 ARM64 ABI
未定义的垃圾值。如果 builder 将结果标记为 `VTAG_TAGGED/UNKNOWN`，垃圾 tag 被写入 `runtime_tags`，
后续多态 helper（如 `jit_value_from_tag`）使用错误 tag 导致类型混淆。

**典型案例**：
- `xr_jit_getbuiltin`：返回任意类型的 `XrValue.i`，未设 `call_result_tag`
- `xr_jit_spawn_cont`：返回 Task PTR，builder 标记为 TAGGED
- `xr_jit_getfield_ic`：返回字段值（任意类型），builder 标记为 TAGGED
- `xr_jit_enum_access/enum_convert`：返回枚举实例 PTR，builder 标记为 TAGGED

**触发条件**：helper 返回 `int64_t` + builder 不设具体 VTAG + 结果在多态上下文使用

**防御手段**：
- 所有返回有用值的 `int64_t` helper 必须显式设 `coro->jit_ctx->call_result_tag`
- 返回 `XrJitResult` 的 helper 天然通过 x1 传递 tag，无需额外处理
- Builder 标记 `VTAG_PTR/I64/F64/BOOL` 等具体类型时，codegen 使用静态 tag，不依赖 `call_result_tag`
- Lint 检查：`int64_t xr_jit_*()` 函数中应有 `call_result_tag` 赋值（除非结果未使用）

**检测脚本**：
```bash
# 查找返回 int64_t 的 JIT helper，检查是否设置了 call_result_tag
grep -n '^int64_t xr_jit_' src/jit/xir_jit.c | while read line; do
    fn=$(echo "$line" | sed 's/.*xr_jit_/xr_jit_/' | sed 's/(.*//');
    if ! grep -A30 "^int64_t $fn" src/jit/xir_jit.c | grep -q 'call_result_tag'; then
        echo "WARNING: $fn missing call_result_tag"
    fi
done
```

---

## Pattern 25: CPS suspend result vreg 缺少 bc_slot（braun_write_var vs builder_set_slot）

**描述**：`braun_write_var` 只向 Braun SSA 定义表写入变量映射，不设置 vreg 的 `bc_slot`。
`builder_set_slot` 同时更新 `slot_map`、`bc_slot` 和 Braun 定义表。误用 `braun_write_var`
导致 vreg 的 `bc_slot=-1`，codegen 跳过 `runtime_tags` 写入，suspend→resume 后 stale tag 不被更新。

**典型案例**：
- JIT Await Race Bug：AWAIT builder 用 `braun_write_var` 写结果 → `bc_slot=-1` →
  `runtime_tags` 不被写入 → resume 后 stale NULL tag → ADD 把整数当 NULL 处理
- 修复：在 `builder_emit_cps_suspend` 中显式设置 `vregs[vi].bc_slot = result_slot`

**触发条件**：CPS suspend/resume 路径中使用 `braun_write_var` 替代 `builder_set_slot`

**防御手段**：
- `braun_write_var` 调用后必须验证目标 vreg 的 `bc_slot` 已设置
- 优先使用 `builder_set_slot`，仅在写入非当前块时才用 `braun_write_var`
- 写入非当前块时，手动设置 `vregs[vi].bc_slot`
