# 语言共性

第五章会按语言写实现落点；本章先固定一组**跨语言语义轴**，避免把每门语言读成互不相关的特性清单。

| 轴 | 专页 | 对话时用来干什么 |
| --- | --- | --- |
| 类型 | [类型系统轴](typing.md) | 擦除 vs 单态化、静态/动态/渐进的成本落在哪 |
| 内存 | [内存管理轴](memory-management.md)、[GC 机制](gc-mechanics.md) | 谁分配、谁回收、停顿与循环引用 |
| 执行 | [求值与执行模型](evaluation.md) | 解释 / 字节码 / JIT / AOT 测基准时在测哪一段 |
| 并发 | [并发模型](concurrency.md) | 线程、async 状态机、CSP、actor 与内存可见性 |
| 表示 | [对象布局](object-layout.md)、[派发与胖指针](polymorphism-repr.md) | 虚调用、interface、`dyn Trait` 差几层间接 |
| 正确性 | [UB 与抽象机](ub-abstract-machine.md) | 优化合法性、与「机器上能跑」的界限 |
| 边界 | [FFI](ffi.md) | 调用约定、所有权与异常能否跨界 |

读语言分论时，用上表当目录：先在本章找到轴的定义，再去该语言页看落点。轴的定义只在本章展开一次；分论页不重讲。
