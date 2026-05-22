---
id: spec.17_compilation_pipeline
order: 018
---

<!-- xr-spec:cn -->
---

## 17. 编译流水线 (Compilation Pipeline)

> 真值源：`src/frontend/`、`src/vm/`、`src/jit/`、`src/aot/`、`docs/rules/architecture.md`。

### 17.1 阶段总览

```
源码 (.xr)
    ↓ lexer
Token Stream
    ↓ parser
AST
    ↓ analyzer (语义分析、类型检查、scope/capture/generic)
Typed AST
    ↓ ssa-gen
SSA IR
    ↓ optimize（const fold、DCE、inline、TCO、escape analysis）
Optimized SSA
    ↓ codegen
Bytecode  →  AOT (machine code)
    ↓ VM
    ↓ Profiler → JIT (machine code)
执行
```

### 17.2 词法分析 (Lexer)

- 真值源：`src/frontend/lexer/xlexer.c`。
- 输出 `XrToken` 流，每个 token 含 `kind`、`value`、`pos(line, col)`。
- 处理：字符串插值（产生 `${...}` 拼接序列）、原始字符串、正则字面量。

### 17.3 语法分析 (Parser)

- 真值源：`src/frontend/parser/`（分文件：expr、stmt、decl、match）。
- 风格：手写 Pratt parser（表达式）+ 递归下降（声明 / 语句）。
- 错误恢复：遇到错误后跳到下一同步点（`;` `}` `)`），尽量继续解析。
- 输出：`XrAstNode*` 根（即 module）。

### 17.4 语义分析 (Analyzer)

- 真值源：`src/frontend/analyzer/xanalyzer_*.c`（按主题拆分）。
- **作用域**：嵌套符号表、变量解析、shadowing 检查。
- **类型检查**：双向类型推断、union 收窄、Json 结构匹配。
- **泛型**：monomorphization、约束检查、调用点重写。
- **闭包分析**：upvalue 标记、`go` 闭包捕获禁令。
- **错误码**：`XR_ERR_ANALYZE_*` 系列。

### 17.5 SSA 与优化

- 真值源：`src/ir/xi_opt*.c`、`src/ir/xi_pass.h`、`src/jit/`。
- **常量折叠**：编译期求值。
- **DCE**（dead-code elimination）：删除未使用代码。
- **inlining**：小函数内联。
- **TCO**（tail-call optimization）：accumulator 风格尾递归转循环。
- **escape analysis**：栈分配 vs 堆分配决策。

### 17.6 字节码与 VM

- 真值源：`src/vm/`、`include/xray_opcodes.h`。
- 寄存器栈混合 VM。
- IC（inline cache）加速属性访问与方法分派。

### 17.7 JIT 与 AOT

- **JIT**（运行时）：热函数被 profiler 选中后 → 编译为本地机器码。源码：`src/jit/`。
- **AOT**（提前）：`xray build --aot` → 整个模块编译为 native binary。源码：`src/aot/`。
- 共享 SSA IR；后端选择不同（解释 / JIT / AOT）。
<!-- /xr-spec:cn -->

<!-- xr-spec:en -->
---

## 17. Compilation Pipeline

Pipeline:

```text
source
  -> lexer
  -> tokens
  -> parser
  -> AST
  -> analyzer/type checker
  -> typed AST
  -> IR/lowering
  -> bytecode
  -> VM / JIT / AOT
```

Primary implementation areas:

| Area | Directory |
|--|--|
| Lexer/parser | `src/frontend/lexer`, `src/frontend/parser` |
| Analyzer | `src/frontend/analyzer` |
| IR/lowering | `src/ir` |
| VM | `src/vm` |
| JIT | `src/jit` |
| AOT | `src/aot` |
| Runtime objects | `src/runtime` |
| Stdlib native modules | `stdlib` |
<!-- /xr-spec:en -->
