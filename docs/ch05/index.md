# 语言分论

第四章把类型、内存、求值、并发、对象布局、派发、抽象机与 FFI 收成一组**跨语言语义轴**；本章只写各门语言在这些轴上的**落点**。轴的定义、对照表与术语边界留在 [语言共性](../ch04/index.md)，分论页不重讲。

## 阅读顺序

先扫一遍 [第四章概览表](../ch04/index.md)，确认「擦除 / 单态化」「逃逸」「数据竞争」「胖指针」等词已有固定含义。再按语言页阅读：每一页按同一骨架组织——实现谱系与执行管线、内存与对象表示、并发与运行时调度、与 C / ABI 的边界、影响正确性与性能的语义、导师对话笔记——便于横向对照，而不是按特性清单罗列。

系统级四门（C、C++、Rust、Go）默认落在 Linux x86-64、[System V AMD64 ABI](../ch02/sysv-amd64-abi.md) 与 AOT 管线上；托管与动态语言页会标明字节码、JIT 或解释路径，读基准时要先对齐测的是哪一段执行模型（见 [求值与执行模型](../ch04/evaluation.md)）。

## 与第四章的回链方式

遇到「这门语言怎么管内存」一类问题：先在 [内存管理轴](../ch04/memory-management.md) / [GC 机制](../ch04/gc-mechanics.md) 定位策略族，再打开对应语言页看分配器、逃逸、停顿或所有权检查落在何处。派发与虚调用对照固定指向 [派发机制与胖指针](../ch04/polymorphism-repr.md)；UB、as-if 与「机器上能跑」的界限指向 [UB 与抽象机](../ch04/ub-abstract-machine.md)；跨语言调用的约定与寿命指向 [FFI](../ch04/ffi.md)。

口试量规里与语言实现直接相关的条目——抽象机与 UB、派发间接层、GC 与所有权、基准数字拆解——应能从「轴定义页 + 本语言落点页」各指一段作答，见 [口试量规](../Spec/oral-rubric.md)。

## 本章页面

| 页 | 默认管线上的抓手 |
| --- | --- |
| [C](c.md) | 抽象机、UB、到 `.o` 的短路径、与 ABI 近零距离 |
| [C++](cpp.md) | 对象模型与寿命、模板单态化、异常表、Itanium ABI 轮廓 |
| [Rust](rust.md) | 所有权/借用、LLVM 后端、panic、`unsafe`/FFI、数据竞争自由边界 |
| [Go](go.md) | 逃逸分析、GC、GMP、interface 派发、cgo 成本 |
| [Java 与 Kotlin](java.md)、[C#](csharp.md)、[Swift](swift.md) | 托管运行时 / ARC 谱系 |
| [Python](python.md)、[JavaScript 与 TypeScript](javascript.md)、[动态运行时样本](dynamic-runtimes.md) | 解释、字节码与 JIT |
| [跨语言对照表](cross-language-matrix.md) | 同轴并排汇总 |

系统级四门写满后，再读托管与动态页时，用同一骨架核对差异，而不是另起一套分类。
