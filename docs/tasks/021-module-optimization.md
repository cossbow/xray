# src/module 模块优化计划

> 生成日期: 2026-04-21
> 文件数: 21 (11 .c + 10 .h), 约 6,200 行
> 子系统: 模块加载、字节码序列化、包管理、依赖解析、SemVer、Lockfile

---

## 一、架构 / 设计层面

### OPT-01 文件读取模式高度重复 🟡

`fopen → fseek → ftell → fseek → fread → fclose` 至少出现 6 处:

| 文件 | 函数 |
|------|------|
| xmodule.c:589 | `load_script_extension` |
| xmodule.c:718 | `load_script_module` |
| xbundle.c:45 | `read_file_content` |
| xpkg_client.c:45 | `read_file_content` |
| xlockfile.c:172 | `xr_lockfile_load` |
| xbytecode_io.c:746 | `xr_compile_to_file` |

**建议**: 在 `base/` 层提供 `xr_file_read_all(path, &size)` 工具函数。

### OPT-02 编译器钩子用 static 全局变量，不线程安全 🟡

xmodule.c:44-47 的 `fn_parse_with_source` 等 4 个 static 全局变量在多 Isolate 多线程场景下冲突。

**建议**: 将钩子绑定到 `XrModuleRegistry` 或 `XrayIsolate`。

### OPT-03 `realpath()` 返回 malloc 分配的内存 🔴

xmodule.c:884, xbundle.c:95 — `realpath(path, NULL)` 调用系统 `malloc`，
后续用 `xr_free` 释放。若 `xr_malloc/xr_free` 是自定义分配器则 UB。

**建议**: 封装 `xr_realpath()` → 内部 realpath + xr_strdup + free。

---

## 二、xmodule.c — 核心模块系统

### OPT-04 双重缓存写入 🟡

`load_native_module` line 682 已写入缓存，`xr_module_import` line 855 又写一次。

**建议**: 删除 `xr_module_import` 中冗余写入。

### OPT-05 `load_script_extension` 存在死代码 🟢

`execute:` 标签 (line 626) 从未被 goto 跳转，仅 fall-through。

**建议**: 移除标签和无用的 ast 清理。

### OPT-06 路径提取逻辑重复 3 次 🟢

`strrchr(dir, '/') → *last_slash = '\0'` 模式在 xmodule.c 出现 3 处。

**建议**: 提取 `xr_path_dirname()` 工具函数。

### OPT-07 错误消息直接 fprintf(stderr) 🟢

xmodule.c:864-879 绕过 `xr_log_warning` / `xr_error_throw` 体系。

**建议**: 使用结构化错误报告。

### OPT-08 `normalize_path` 不处理 `..` 段 🟢

只处理 `./`，不处理 `../`。

### OPT-09 内联函数中 `goto slow` 模式可读性差 🟢

xmodule.h 中 4 个内联函数全用 `goto slow`。

**建议**: 重构为 `if/else` 或拆分 `_slow` 辅助函数。

### OPT-10 每次 add_export 都 invalidate 索引 🟡

xmodule.c:194-198 每次添加 export 都 free symbol_to_index，批量添加时性能差。

**建议**: 延迟到 `build_export_index` 时再重建。

---

## 三、xbundle.c — 多文件打包

### OPT-11 AST 遍历不完整 — 漏掉多种节点类型 🔴

xbundle.c:283 的 switch 缺少 SWITCH_STMT, MATCH_EXPR, ENUM_DECL, LAMBDA,
FOR_IN_STMT, INTERFACE_DECL 等节点。嵌套在这些节点内的 import 语句会被遗漏。

**修复**: 补充所有含子节点的 AST 类型的遍历。

### OPT-12 路径缓冲区固定 512 字节 🟢

xbundle.c:74 — 应使用 PATH_MAX 或动态分配。

### OPT-13 编译失败静默跳过 🟢

`visit_node` 中编译失败时不报错。

---

## 四、xresolver.c — 依赖解析

### OPT-14 `xr_depgraph_find` 是 O(n) 线性扫描 🟡

被 `graph_get_or_create_node` 频繁调用。

**建议**: 用 `XrHashMap` 做 name→node 索引。

### OPT-15 拓扑排序前多余的环检测遍历 🟡

图被遍历 3 次。

**建议**: 合并为单次 DFS 同时做环检测 + 拓扑排序。

### OPT-16 依赖 spec 解析重复 🟢

`resolve_node` 中 "name@version" 解析逻辑出现 2 次。

**建议**: 提取 `parse_dep_spec()` 辅助函数。

---

## 五、xlockfile.c — Lockfile

### OPT-17 手写 TOML 解析器 🟡

项目已有 `stdlib/toml/toml.h`，lockfile 另写了独立的解析器。

**建议**: 复用 `xr_toml_parse`。

### OPT-18 `xr_lockfile_add_dependency` 每次仅增长 1 🟢

**建议**: 倍增策略。

### OPT-19 `parse_string_array` realloc 失败内存泄漏 🔴

realloc 失败时 return NULL 但没释放已解析的元素。

**修复**: 添加 cleanup 逻辑。

### OPT-20 `xr_lockfile_find` 是 O(n) 线性扫描 🟢

---

## 六、xpkg_client.c — 包客户端

### OPT-21 手写 JSON 解析器 🟡

项目已有完整 JSON 模块。

**建议**: 复用 stdlib/json。

### OPT-22 URL 查询参数未编码 🟢

xpkg_client.c:464 — query 未 URL 编码。

### OPT-23 `read_file_content` 重复定义 🟡

与 OPT-01 合并解决。

---

## 七、xproject.c — 项目配置

### OPT-24 `join_path` 不检查 malloc 返回值 🟢

### OPT-25 `fread` 返回值被忽略 🟢

### OPT-26 `collect_files_recursive` realloc 无保护 🟢

---

## 八、xbytecode_io.c — 字节码 IO

### OPT-27 `xr_compile_to_file` 未释放 proto 🔴

编译结果 proto 成功序列化后从未 free，内存泄漏。

### OPT-28 递归 `bc_read_proto` 无深度限制 🟢

恶意字节码可导致栈溢出。

---

## 九、断言密度不足 🟢

| 文件 | 行数 | 断言数 | 密度 |
|------|------|--------|------|
| xmodule.c | 1087 | ~6 | 1/181 |
| xbundle.c | 522 | ~2 | 1/261 |
| xlockfile.c | 501 | ~1 | 1/501 |
| xpkg_client.c | 805 | ~3 | 1/268 |
| xproject.c | 304 | ~0 | 无 |

---

## 实施顺序

### Phase 1 — 高优先级 Bug 修复 ✅
- [x] OPT-11: xbundle.c AST 遍历补全 (14种节点类型)
- [x] OPT-03: realpath → xr_strdup + free (3处)
- [x] OPT-27: xr_compile_to_file proto 泄漏
- [x] OPT-19: parse_string_array realloc 泄漏修复

### Phase 2 — 中优先级性能/可维护性 ✅
- [x] OPT-01: 提取 xr_file_read_all / xr_path_dirname / xr_realpath → base/xfileio.h
- [x] OPT-04: 双重缓存写入
- [x] OPT-05: 死代码 execute: 标签 + 未用 ast 变量
- [x] OPT-06: xr_path_dirname 替换内联 dirname 模式
- [x] OPT-12: 固定缓冲区 512 → PATH_MAX
- [x] OPT-14: depgraph_find O(n)→O(1) hashmap 索引
- [x] OPT-15: 合并环检测+拓扑排序为单次 DFS

### Phase 3 — 低优先级代码质量 ✅
- [x] OPT-08: normalize_path 增加 `..` 段解析 (纯词法路径规范化)
- [x] OPT-09: xmodule.h goto slow → if/else (4个内联函数)
- [x] OPT-13: xbundle.c 编译失败添加 xr_log_warning (parse/compile/serialize)
- [x] OPT-16: 提取 parse_dep_spec() 辅助函数 (消除 2 处重复)
- [x] OPT-18: lockfile add_dependency 倍增策略 + dep_capacity
- [x] OPT-24: join_path NULL 检查 + DCHECK
- [x] OPT-25: xproject.c 用 xr_file_read_all 替代 fopen/fread
- [x] OPT-26: collect_files_recursive realloc 安全中转
- [x] OPT-28: bc_read_proto 递归深度限制 (BC_MAX_NESTING_DEPTH=64)
- [x] 断言密度: xbundle.c, xproject.c 入口校验

### Phase 4 — 架构修正
- [x] OPT-02: 编译器钩子 static 全局 → 移入 XrModuleRegistry (per-Isolate, 线程安全)

### 不实施 (有架构原因)
- OPT-10: 延迟 invalidate 索引 — batch add 后一次 build_index，影响极小
- OPT-17: xlockfile 改用 stdlib/toml — 循环依赖禁止 (src/module ← stdlib/toml → src/module)
- OPT-21: xpkg_client 改用 stdlib/json — 同上 (src/module ← stdlib/json → src/module)
- OPT-22: URL 查询参数编码 — 同上层次依赖问题
