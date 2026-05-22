---
id: spec.appendix_f_glossary
order: 025
---

<!-- xr-spec:cn -->
---

## 附录 F. 词汇表

| 术语 | 定义 |
|--|--|
| **AOT** | Ahead-of-Time 编译：构建时预编译为机器码 |
| **AST** | Abstract Syntax Tree：源码解析后的中间表示 |
| **Arena** | 批量分配器：所有分配同时释放 |
| **Bytes** | 字节缓冲类型（见 §2.4.5） |
| **Channel** | 类型化的协程通信管道（见 §10.5） |
| **closure** | 闭包：捕获外层变量的函数 |
| **coroutine** | 协程：用户态可暂停/恢复的执行流 |
| **defer** | 延迟执行：函数退出前执行（见 §4.9） |
| **enum** | 枚举类型（见 §5.6） |
| **GC** | Garbage Collector：垃圾回收 |
| **GC-safepoint** | GC 安全点：可安全开始 GC 的指令位置 |
| **goroutine** | xray 中称作协程 (coroutine)，启动语法 `go {...}` |
| **hoisting** | 提升：声明在使用前被隐式定义 |
| **IC** | Inline Cache：内联缓存（属性访问/方法分派优化） |
| **interface** | 接口（见 §5.5） |
| **JIT** | Just-In-Time 编译：运行时编译热路径 |
| **lvalue / rvalue** | 左值（可赋值）/ 右值（仅值） |
| **monomorphization** | 单态化：泛型实例化为多个具体类型版本（xray 不做） |
| **move** | 所有权转移：跨协程时强制（见 §7.3） |
| **NaN-boxing** | 用 IEEE-754 NaN 的位空间存放标记值 |
| **nullable** | 可空类型 `T?`：值可以为 null |
| **pattern** | 模式：用于 `match` 与解构（见 §6） |
| **scope** | 作用域 |
| **shared** | 跨协程共享的存储类（见 §5.1.3） |
| **SSA** | Static Single Assignment：每个变量只赋值一次的 IR |
| **struct** | 值类型类（见 §5.4） |
| **TCO** | Tail-Call Optimization：尾调用优化 |
| **trait** | Rust 术语；xray 用 `interface` |
| **truthy** | 真值：控制流中非 `false` / `null` / `0` / `""` / 空集合的值视为真（见 §2.3.3） |
| **monomorphization** | 单态化：泛型类型参数在编译期特化为具体类型，运行时保留类型信息 |
| **union** | 联合类型 `A \| B` |
| **upvalue** | 闭包捕获的外层变量 |
| **VM** | Virtual Machine：xray 字节码虚拟机 |
| **write barrier** | 写屏障：GC 在指针更新时插入的钩子 |
<!-- /xr-spec:cn -->

<!-- xr-spec:en -->
---

## Appendix F. Glossary

| Term | Meaning |
|--|--|
| AOT | Ahead-of-time compilation |
| AST | Abstract syntax tree |
| Channel | Typed coroutine communication pipe |
| Closure | Function value capturing outer variables |
| Coroutine | User-space schedulable execution unit |
| GC | Garbage collection |
| JIT | Just-in-time compilation |
| Nullable | Type that can contain `null` |
| Pattern | Match/destructuring form |
| Prelude | Built-in symbols available without import |
| Safepoint | Point where coroutine/GC scheduling can observe execution |
| SSA | Static single assignment IR |
| VM | Bytecode virtual machine |
<!-- /xr-spec:en -->
