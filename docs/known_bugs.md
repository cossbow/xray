# 已知 Bug 清单

记录暂时无法立即修复但已知的 bug，遵循 main rules 中"Bug 修复铁律"的要求：

格式：日期 + 现象 + root cause（若已查明）+ 影响 + 复现方法。

---

### 2026-05-12 — 本地架构/注释质量门禁存在既有失败项

- **现象**：更新 `076-codegen-master-spec.md` 后执行本地质量门禁时，`bash scripts/check_architecture.sh` 返回 3 errors / 46 warnings；`bash scripts/check_comment_rules.sh` 返回 1 forbidden pattern。快速测试 `ctest --output-on-failure --test-dir build` 为 113/113 通过。
- **root cause（待拆分修复）**：
  1. 架构检查暴露既有代码质量债：`src/ir/xi.h` 超过头文件行数限制、若干系统分配接口仍被直接调用、存在 base → runtime upward include。
  2. 注释规则检查暴露既有脚本注释违规：`scripts/win_test.sh` 与 `scripts/win_build.sh` 注释中引用了文档路径。
  3. 这些失败项不是本次 076 文档修改引入；仓库中已有审计基线记录过同类架构债务。
- **影响**：本地质量门禁不能作为绿色信号；后续涉及架构/注释规则的任务需要先拆分修复这些既有失败项，避免 CI 或人工验收误判。
- **复现方法**：
  ```bash
  bash scripts/check_architecture.sh
  bash scripts/check_comment_rules.sh
  ```

### 2026-05-20 — Exception subclass instance crashes when assigned to variable

- **现象**：`let err = new HttpError(500, "server error")` 静默崩溃（exit code 1，无输出），而 `throw new HttpError(...)` + catch 正常工作。`let e = new Exception(...)` 也正常。
- **root cause**：Exception 是 native class（`XR_CLASS_BUILTIN | XR_CLASS_HAS_NATIVE_BODY`），子类构造走 `vm_invoke_class` 时分配大小或 native body 布局计算对用户继承场景有缺陷。`throw` 路径绕过了普通变量赋值。
- **影响**：任何 Exception 子类在非 throw 场景（存入变量、记录日志、存入集合）都会崩溃。spec §18.8 鼓励继承 Exception 添加业务字段，此 bug 阻塞该用法。
- **复现方法**：
  ```xray
  class MyError extends Exception {
      constructor(message: string) { super(message) }
  }
  let err = new MyError("oops")
  print(err.message)
  ```

---

## 历史已修复

### 2026-05-19 — xr_tref_tuple: empty tuple 断言失败 + TUPLE_NEW 无法 DCE  ✅ 已修复

- **现象**：`test_fmt_roundtrip`、`test_xi_lower`、`test_xi_opt`、`0540_higher_order.xr`、`1230_function_type_annotation.xr`、`1231_type_alias.xr` 触发 `[FATAL] xtype_ref.c:163: Check failed: xr_tref_tuple: empty tuple`；修掉空 tuple 后 `test_xi_opt` 的 `tuple_new_eliminated_after_full_projection` 又因 `TUPLE_NEW` 被错误归类为 SIDE_EFFECT 而保留。
- **真正根因**：
  1. `xparse_type.c` 解析 `(T1, T2)` 后若没有 `->` 跟随，对 count=0 的情况仍调用 `xr_tref_tuple(...,0)`，触发 `xr_tref_tuple: empty tuple` 断言。`()` 应当是 unit 类型 `XR_TREF_UNIT`，不是 0 元 tuple。
  2. `xi_effect.h` 把 `XI_TUPLE_NEW` 与可变容器 (`ARRAY_NEW`/`MAP_NEW` 等) 同列为 `SIDE_EFFECT | WRITES_MEM`。tuple 是不可变值，分配结果只能通过 use chain 观察，未使用的 `TUPLE_NEW` 本应是 DCE 候选。
- **修复**：
  1. `xparse_type.c`：`(...)` 且无 `->` 时，count==0 返回 `xr_tref_unit()`；count>0 保留 tuple 路径。
  2. `xi_effect.h`：拆出"纯不可变分配"组，`XI_TUPLE_NEW` 只标 `WRITES_MEM`，不再标 `SIDE_EFFECT`，DCE 可正确收割。
- **验证**：
  - `ctest`：117/117 通过。
  - 回归脚本：307/307 通过。

### 2026-05-14 — 编译错误测试集 readonly/strict-field/index 失败项  ✅ 已修复

- **崩溃表象**：完整编译错误套件有 10 个失败；readonly/strict-field fixture 停在 parser，动态字段添加和数组字符串索引未被 analyzer 拒绝。
- **真正根因**：
  1. JSON readonly/strict-field fixture 使用旧对象类型简写 `{id,name}`，当前对象类型语法要求字段类型。
  2. 对象类型注解中的 `const` 字段未落到 `XrType.object.field_readonly`。
  3. sealed Json 的未知字段读取/写入、索引写入，以及数组/string 索引类型缺少静态检查。
- **修复**：
  1. fixture 改为显式对象类型别名，并补充动态字段添加与非法索引期望文件。
  2. 对象类型引用保留 readonly 元数据，解析为 `XrType` 后参与只读字段赋值检查。
  3. analyzer 拒绝 sealed Json 未知字段访问/添加和明确错误的数组/string 索引类型。
- **验证**：
  - `bash scripts/run_compile_error_tests.sh`：23/23 通过。
  - `ctest --output-on-failure`：114/114 通过。
  - `XRAY_SKIP_BUILD=1 scripts/run_regression_tests.sh`：296/296 通过。

### 2026-05-12 — Windows 端 `1131_linked_go_expr.xr` linked scope wait race  ✅ 已修复

- **崩溃表象**：Win11 Debug 下 5× hammer 可复现内部 timeout。拆分后 `go + await`、`task.monitor().recv()`、`linked scope + await` 单独 5/5 通过；`go + await` 后接 `linked scope + await` 组合在 5× hammer 中复现 timeout。
- **真正根因**：scope barrier 和显式 `await t` 共享 `XrTask.waiter` 单槽。scope child 创建时把 task waiter 注册为 scope waiter；随后 scope 内显式 `await t` 会覆盖这个 waiter。大多数时序下 `scope->count` 能兜住，但 Windows 调度交错下 `OP_SCOPE_EXIT` 进入 scope wait 后可能只依赖 stale `wait_count`，child 完成后没有可靠的 owner wake，导致父协程一直阻塞到测试内部 120s deadline。
- **修复**：
  1. `XrScopeContext` 增加 owner coroutine，scope wait 以 `scope->count` 为单一真相源。
  2. scope child 完成时在 `xr_coro_wake_waiter` 中递减 `scope->count`，归零后直接唤醒正在等待该 scope 的 owner。
  3. scope 内 `go` 不再占用 `task->waiter` 单槽；显式 `await` 和 scope barrier 互不覆盖。
  4. JIT scope enter/exit/go 路径同步使用 `scope->count` 和 owner wake contract。
- **验证**：
  - macOS Debug build 通过；`ctest --test-dir build --output-on-failure`：113/113 通过。
  - macOS Debug 最小复现 + 原始 `1131_linked_go_expr.xr`：通过。
  - Win11 Debug build 通过。
  - Win11 Debug 最小复现 `go + await` 后接 `linked scope + await`：5/5 hammer 通过。
  - Win11 Debug 原始 `1131_linked_go_expr.xr`：10/10 hammer 通过。
  - Win11 Debug ctest：105/105 通过。

### 2026-05-12 — Windows 端 `1120_regex_re2_compat.xr` / `1122_regex_jit_dynamic_null_index.xr` JIT regex null/tagged 参数失败  ✅ 已修复

- **崩溃表象**：Windows Debug/Release JIT 下，regex 相关测试在 `regex.find(re, text) != null`、动态 tagged null 比较、以及 regex match result index-get 场景中失败；macOS arm64 Debug 稳定通过。
- **真正根因**：
  1. null 比较 lowering/codegen 只看 payload，动态 tagged null 返回值需要使用运行时 tag 判断。
  2. Win64 regalloc 的 CALL fixed intervals 使用了 ARM64 caller-saved 寄存器数量，错误地把 Win64 callee-saved alloc regs 当作 CALL clobber，导致 live-across-call 参数被不必要 split/spill。
  3. x64 edge copy 缺少 block 边界 `reg -> spill` split transition，后继 block 从 stale spill slot 读取 `regex.find` 的 `text` 参数。
- **修复**：
  1. null 比较统一 lowering 到 runtime null check；x64/ARM64 对动态 tagged vreg 从 `vreg_runtime_tags` 读取 tag。
  2. regalloc 使用 target 的 caller-saved 寄存器数量创建 CALL fixed intervals。
  3. `XraEdgeCopy` 增加 store transition，x64 edge-copy emission 支持 `reg -> spill`。
- **验证**：
  - macOS Debug `ctest --test-dir build --output-on-failure`：113/113 通过。
  - macOS Debug `1120_regex_re2_compat.xr` + `1122_regex_jit_dynamic_null_index.xr`：62/62 通过。
  - Win11 Debug `1122_regex_jit_dynamic_null_index.xr`：通过。
  - Win11 Debug `1120_regex_re2_compat.xr`：61/61 通过。
  - Win11 Debug 最小 stale text 复现修复后通过。

### 2026-05-12 — Windows 端 `1148_scope_race_stress.xr` linked scope 取消竞态  ✅ 已修复

- **崩溃表象**：Win11 Debug 下 `1148_scope_race_stress.xr` 的 linked scope 子测试可长时间占满 CPU 且超过 360s watchdog；更早一次跑以 `-1073741819`（0xC0000005 access violation）退出。拆分后 supervisor 三个子测试在 Windows 均 <500ms 通过，linked-only 子测试稳定复现 hang。
- **真正根因**：linked scope 的首个失败 child 在 `wake_waiter_cancel_linked_siblings_locked` 中远程取消 sibling：直接 `xr_coro_cancel(sib)` 标记 `DONE`，随后 `xr_task_cancel`、清 `sib->parent_scope`、递减 `scope->count` 并唤醒 waiter。这个路径绕过了 sibling 所属 worker 的 `XR_VM_CANCELLED` 完成流程，可能与 sibling 正在运行或即将完成的正常路径并发写同一组 task/scope/coro 状态，造成 `scope->count` / `wait_count` / waiter 唤醒顺序不一致，Windows 调度交错下表现为 hang 或 AV。
- **修复**：
  1. linked scope 取消 sibling 改为 cooperative cancellation：只设置 `XR_CORO_FLG_CANCEL_REQUESTED` 并强制 reductions 归零，让 sibling 在自身 worker 的 safepoint 进入 `XR_VM_CANCELLED`。
  2. `XR_VM_CANCELLED` 统一清理 `CANCEL_REQUESTED | READY | BLOCKED | RUNNING` 状态位，避免取消完成的 coroutine 仍保留可运行/阻塞状态。
  3. `xr_coro_run_on_worker` 入口先识别 `CANCEL_REQUESTED`，保证未开始或刚被重新调度的 sibling 也走统一取消完成路径。
- **验证**：
  - macOS Debug ctest 113/113 通过；`1148_scope_race_stress.xr` 4/4 通过。
  - Win11 Debug ctest 105/105 通过。
  - Win11 Debug linked-only 临时拆分测试修复前 hang，修复后 426ms PASS。
  - Win11 Debug `1148_scope_race_stress.xr` 修复后 5×hammer 全部 4/4 通过（约 1.1–1.3s/次）。

### 2026-05-12 — Windows 端 `0881_nested_iterator_combo.xr` x64 JIT codegen buffer overflow  ✅ 已修复

- **崩溃表象**：Win11 Debug 上 `xray.exe test tests/regression/08_oop/0881_nested_iterator_combo.xr` 稳定失败 (exit=3)，3/3 hammer 全 fail。stderr 依次打：
  ```
  [DEBUG] [jit-bg] codegen failed for ?: x64 codegen: unsupported opcode or regalloc error
    (src/jit/xjit_compile_queue.c:134)
  [FATAL] src/jit/xm_x64.h:132: Check failed: x64_emit8: buffer overflow
  ```
  macOS arm64 走 arm64 backend 不受影响。
- **真正根因**：x64 codegen 的 code buffer 容量走了 `total_xm_ins * 40 + 1024` 的估算，对于含大量 call-shaped op（`OP_INDEX_GET/SET` 走 `call_c_stub`、`OP_CALL_C` 伴 caller-save spill/restore、`OP_CALL_KNOWN` 带 deopt 分支 …）的 nested iterator 函数不够，实际发射出的字节远超估算。`x64_emit8` 以前是 `XR_DCHECK(buf->pos < buf->capacity, "x64_emit8: buffer overflow")` —— Debug 下直接 fatal abort 进程，连 fallback 到解释器的机会也没有。
- **修复**：
  1. `src/jit/xm_x64.h`：给 `X64Buf` 增一个 sticky `bool overflow`。`x64_emit8 / x64_emit32 / x64_emit64` 在 `pos > capacity` 时不再 fatal，而是 set `overflow = true` 且 silently 跳过本次写入，让 codegen 管道继续走到 graceful failure 点。Patch helpers 只写已记录的 position，不会走到这些 emit 接口。
  2. `src/jit/xm_codegen_x64.c`：emit 主流程（blocks + stubs）跑完、`x64_patch_branches` 之前加 `if (ctx.buf.overflow) { result.error = "x64 codegen: code buffer overflow"; goto cleanup; }`。这样 background worker 看到 `result.success=false` 会退回解释器，不发布不完整的 JIT 代码。
  3. 同文件把 buffer 估算从 `total_xm_ins * 40 + 1024` 提到 `total_xm_ins * 96 + 4096`，覆盖 call-shaped op 的实际 worst case + 所有 stubs。这样在 0881 这种函数上 buffer 能装下所有 emit，JIT 能走完；即使未来又有函数超预算，也只是 graceful 退回解释器不再崩溃。
- **验证**：
  - macOS arm64 Debug ctest 113/113 不变，回归未变。
  - Win11 Debug ctest 105/105 不变。
  - `0881_nested_iterator_combo.xr` 修复前 3/3 hammer fatal abort、修复后 3/3 hammer 17/17 子用例全过。
  - Win11 全量回归从 `pass=290 fail=2 timeout=1` 变为 `pass=291 fail=1 timeout=1`（0881 从 fail 列表消失）。

### 2026-05-12 — Windows 端 `1136_task_link.xr` flaky  ✅ 已修复

- **崩溃表象**：Win11 Release 上 `1136_task_link.xr` 之前 5 次 hammer 约 2/5 失败，是 cross-worker channel + linked go expression 场景。
- **真正根因**：`bab4002` 修复了 coroutine task state 的 race。`xtask` 的 state 转换 + monitor channel 生命期之前依赖隐含的顺序，在 IOCP 调度下被打乱 → task linking 拿到错误状态。
- **验证**：`scripts/win_pd_test.sh --no-sync --no-build --xray-test ...` 对 1136 跑 5+5 hammer，全 10/10 通过。
- 关联 commit：`bab4002`。
- **注**：同一测试族中的 `1131_linked_go_expr.xr` 后续确认是 scope barrier 与显式 await 复用 task waiter 的独立 race，已在 2026-05-12 修复。

### 2026-05-11 — Windows 端 `1420_datetime_basic.xr` line 24 断言失败  ✅ 已修复

- **崩溃表象**：Win11 Release VM 上 `xray.exe test C:\workspace\xray\tests\regression\10_stdlib\1420_datetime_basic.xr` 稳定失败，`<anonymous>: assertion failed at line 24: values not equal`，13/14 子测试通过。macOS arm64 Debug 上 14/14 通过。
- **真正根因**：测试中 `datetime.create(1970, 1, 1, 0, 0, 0)` 默认本地时间。在 Asia/Shanghai (UTC+8) 这条 wall-clock 对应的 UTC `time_t` 是 -28800。MSVC CRT 的 `mktime` / `_mkgmtime` / `localtime_s` / `gmtime_s` 全都拒绝任何 `time_t < 0`，前两个返回 `(time_t)-1`、后两个让输出 `struct tm` 处于未指定状态。结果 `dt.year()` 拿到的 `tm_year` 不是 70 而是 `_mkgmtime` 失败后的 `tm` 残留值，断言 `year == 1970` 失败。glibc/musl/macOS 没有这个限制，故仅在 Windows 触发。
- **修复**：在 `stdlib/datetime/datetime.c` 引入基于 Howard Hinnant `civil_from_days` / `days_from_civil` 的 portable 实现（`portable_timegm` / `portable_gmtime`），与平台 CRT 完全解耦，覆盖 `int` 范围内所有年份。三个调用点改造为：(a) `datetime_mktime` 内部用 `portable_timegm` + `local_offset_at(wall_t)` 完成 local→UTC 转换；(b) `xr_datetime_to_tm` 先把 `tz_offset*60` 加回 timestamp 再用 `portable_gmtime` 解读；(c) `local_offset_at` 在 Windows 上对 `t < 86400` 改用 1970-01-02 UTC 作为 probe，避免 `localtime_s`/`gmtime_s` 拒绝负 `time_t`。`xr_datetime_parse` 的直接 `mktime` 调用也并入统一 helper。
- **验证**：
  - macOS Debug ctest 113/113 不变，回归 293 文件 / 2688 用例全过，0 倒退。
  - Win11 Release ctest 105/105 不变；`1420_datetime_basic.xr` 修复前稳定 13/14、修复后 5×hammer 全 14/14；`1421_datetime_format.xr` 13/13、`1422_datetime_calc.xr` 21/21 不受影响。

### 2026-05-11 — Windows 端 JIT x64 codegen RT-opcode fallback 漏 NOP → `0xC0000374`  ✅ 已修复

- **崩溃表象**：Win11 Release VM 上 `1207_gc_stress.xr` 偶发以 exit code `-1073740940` (`0xC0000374` = `STATUS_HEAP_CORRUPTION`) 退出，5 次约 1-2 次复现。stderr 反复打印 `[WARNING] [x64-cg] RT opcode 83 should use CALL_C path (xm_codegen_x64_ins.c:1263)`。macOS arm64 上同样 warning 但稳定通过。
- **真正根因**：`XM_RT_PRINT/ARRAY_LEN/INDEX_GET/INDEX_SET` 是 AOT-only opcode，JIT builder 不应该生成；当 builder 出 bug 让它漏到 codegen 时，arm64 后端的 fallback 会 emit 一条 `a64_nop()` 保持指令流长度，x64 后端漏了这一步只打 warning 不发任何字节。结果 codegen pass 的偏移记账（branch patches、jump 目标、跨块 fixup）按"每个 XmIns 至少 1 条机器指令"的隐含约定跑，导致后续分支/调用 displacement 跳到指令中间或越过指令，多次跑后栈/堆元数据被覆盖 → `STATUS_HEAP_CORRUPTION`。
- **修复**：在 `xm_codegen_x64_ins.c` 与 `xm_codegen_x64_mem.c` 两个 x64 fallback 路径里都加 `x64_nop(&ctx->buf)`，与 arm64 行为对齐。这是 defence-in-depth 修复；更深的 root cause（JIT builder 为何生成 AOT-only opcode）需要在 lowering pipeline 上单独处理。
- **验证**：
  - macOS Debug ctest 113/113 不变。
  - Win11 Release ctest 105/105 不变。
  - Win11 Release 回归 288/293 → **291/293**；`1207_gc_stress.xr` 5/5 hammer 全部通过（修复前 5/5 中至少 1 次 heap corruption）。
- 关联 commit：`8b06997`。

### 2026-05-11 — Windows 端 `test_vm_api` SEGFAULT (`0xC0000005`)  ✅ 已修复

- **崩溃表象**：Parallels Win11 ARM (x64 target, MSVC, Release) 跑 `test_vm_api` 在 `vm_interpret_proto_null_proto_returns_error` 上 access violation。Linux/macOS Debug 不复现，因为该测试本身 `#ifdef NDEBUG`。
- **真正根因**：`xr_vm_interpret_proto` 用 `XR_DCHECK(proto != NULL, ...)` 验证 proto。`XR_DCHECK` 在 Debug 是 abort、Release 是 no-op。Release 下 NULL proto 一路被传到 `xr_closure_new(isolate, NULL, ...)`，第一次解引用 `proto->...` 就 crash。测试本身的 `#ifdef NDEBUG` 注释明确写了契约："In Release the DCHECK is a no-op; entry must still surface an error rather than dereferencing the NULL proto."
- **修复**：对照 `xr_vm_call_closure` 的 hardening 模式（DCHECK 用于"调用方 bug 永不应出现"的契约违反，对运行时 NULL 用显式 `if return`），把 `proto == NULL` 改为显式分支 `return XR_VM_RUNTIME_ERROR`。`isolate` 仍保留 DCHECK（没有合理 fallback）。
- **验证**：Win11 Release ctest 36/36 → PASS，macOS Debug 不受影响。
- 关联 commit：`9905c96`。

### 2026-05-11 — Windows 端 `test_xi_compare` 触发 `/GS` stack cookie 破坏 (`0xC0000409`)  ✅ 已修复

- **崩溃表象**：测试在第一个 case 跑完后 `STATUS_STACK_BUFFER_OVERRUN`，`__fastfail(FAST_FAIL_STACK_COOKIE_CHECK_FAILURE)`。
- **真正根因**：`execute_and_capture` 用 `freopen("/tmp/xi_cmp_capture.txt", "w", stdout)` 和 `freopen("/dev/tty", "w", stdout)` 重定向 stdout。两条 POSIX 路径在 Windows 默认安装上都不存在（无 `/tmp` 目录，无 `/dev/tty` 设备）。`freopen` 的 destructive 语义是 **先关闭原 stream**，失败时才返回 NULL —— 也就是说调用失败那一刻，stdout 的 CRT FILE\* 已经处于无效状态。后续任何 `printf`/`fprintf(stdout, ...)` 走的就是损坏的 CRT 缓冲区，触发 MSVC `/GS` 检测到栈 cookie 被改写。
- **修复**：换成可移植的 `dup` + `tmpfile` + `dup2` 模式：保存原 stdout fd，把 stdout 重定向到 `tmpfile()`（自动处理临时目录与清理），跑完后 `dup2` 还原。POSIX 的 `dup`/`dup2`/`close`/`fileno` 在 Windows 上是带下划线版本，通过 4 个 `xi_cmp_*` 别名按 `_WIN32` 切换。同时去掉 `g_capture_path` 全局与 `read_capture` 函数，直接从 tmpfile 读回。
- **验证**：macOS Debug 0.54s PASS；Win11 Release 0.83s PASS（全部 ~120 cases）。
- 关联 commit：`1d1055e`。

### 2026-05-11 — `1206_gc_enhanced.xr` / `1207_gc_stress.xr` 在 `--jit-force` 下偶发 SIGSEGV  ✅ 已修复

- **崩溃表象**：`xr_gc_destroy_logger` 解引用 `ref->logger == 0x3` 触发 fault；type 字节看似是 `XR_TLOGGER (28)` 但任何分配点都从未产生过这样的对象。
- **真正根因**：`mark_coro_roots` 在没有匹配 PC 的 bytecode stackmap 时退化为保守扫描，把残留的栈槽误读成一个 `XrValue{tag=PTR, heap_type=MAP, ptr=<obj+8>}`。`xr_coro_gc_markobject` 只用 `obj->extra` 的 bit 0 判断 "shared"，错位指针落在下游 TBLOB body 的某字节上恰好命中 bit 0，从而把错位指针塞进 `shared_refs[]`。下一轮 GC 的 `xr_shared_decref` 对该指针做 `atomic_fetch_sub(&obj->gc_next, 1)`——实际是把真实 TBLOB header 的 `type|marked|extra|objsize` 这 8 字节减 1。多次减 1 后 type 从 30 (TBLOB) 滚到 28 (TLOGGER)，sweep 把它当 Logger 销毁 → 解引用 garbage 崩溃。
- **修复**：`xr_coro_gc_markobject` 的 shared 路径加入 16 字节对齐校验——所有真实 shared 对象都来自 `xr_malloc` / `xr_mem_map`，必然 ≥16 字节对齐。错位指针不可能是真 shared 对象，直接 silently drop。
- **验证**：
  - `1206_gc_enhanced.xr`：Debug 50/50 通过，ASAN 20/20 通过（修复前 ASAN 60-80% 失败率）。
  - `1207_gc_stress.xr`：Debug 50/50 通过（同根因被一并修复）。
  - 完整回归 293 文件 / 2688 用例全过；JIT diff 298 用例全过。
- 关联 commit：`d94c4a8`。
