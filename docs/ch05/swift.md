![Swift](../assets/svg/swift.svg){ width="40" }

# Swift

Swift 默认走 **LLVM** 后端的 AOT 管线（也可在 REPL/Playground 等场景下即时编译），堆上对象的主路径是 **ARC**（自动引用计数），而不是 JVM/CLR 式追踪式 GC。语言把值语义、写时复制与协议存在性（existential）放进类型系统；与 C / Objective-C 的边界是日常互操作面。共性轴见 [内存管理轴](../ch04/memory-management.md)、[求值与执行模型](../ch04/evaluation.md)、[FFI](../ch04/ffi.md)。

## 实现谱系与执行管线

`swiftc` 前端做类型检查与 SIL（Swift Intermediate Language）生成；SIL 上做所有权相关优化、泛型特化、ARC 流量削减（保留/释放消除、移动），再经 LLVM IR 进入与 C/C++/Rust 同族的选指令与寄存器分配，见第三部分 LLVM 链。发布构建是 AOT：墙钟通常不含 JIT 升温，但含模块序列化、增量编译与链接时间；Whole Module Optimization 改变内联与特化边界。

执行模型以本地机器码为主。动态特性（`AnyObject` 消息发送、Objective-C 运行时桥接）在热点上仍可能走消息派发与运行时查找；纯 Swift 泛型默认倾向特化，也可保留抽象调用以控制码尺寸。基准对比 JIT 语言时，不要把「无 warmup」误读成「绝对更快」——要比的是同阶段语义成本。

## 内存与对象表示

类实例在堆上，对象头含类型元数据与引用计数等；赋值与传参对类是共享引用并调整计数。结构体与枚举是值类型：复制语义下默认拷贝，大结构体经 LLVM 按 ABI 决定直接传递或隐藏指针。`Array`/`Dictionary`/`String` 等以写时复制（COW）缓冲实现：读共享缓冲，写时若唯一则原地改，否则复制——表面值语义，底层是引用计数缓冲，见 [对象布局](../ch04/object-layout.md) 与 RC 对照。

协议存在性容器（`any P`）携带值（或堆分配的盒子）与协议见证表/类型元数据，形态接近胖指针：数据 + 表。泛型特化成功时见证调用可变成直接调用；未特化则间接。与 `dyn Trait` / interface 的间接层对照见 [派发与胖指针](../ch04/polymorphism-repr.md)。

ARC 在引用写入时插入 `retain`/`release`（及 `unowned`/`weak` 变体）。环由 `weak`/`unowned` 打断；否则泄漏。追踪式 GC 的写屏障问题换成了赋值路径上的原子性计数与弱引用表，见 [GC 与 RC 对照](../ch04/gc-mechanics.md)。优化器在证明所有权转移后可消除成对 retain/release；微观基准对优化等级敏感。

## 并发与运行时调度

结构化并发以 `async`/`await` 与 `Task`/`Actor` 为中心。异步函数编成由合作式运行时调度的状态机；actor 串行化对隔离状态的访问，跨 actor 调用表现为异步消息。数据竞争在安全模型里靠隔离检查在编译期收紧；与「任意共享内存 + 锁」的 Java 线程模型切口不同，见 [并发模型](../ch04/concurrency.md)。

底层仍可创建线程、使用 GCD（Dispatch）队列；与 async 任务混用时要分清隔离域与队列亲和。原子操作与锁仍存在，用于与 C 互操作或绕过 actor 的场景；那一步重新打开数据竞争表面。

## 与 C / ABI 的边界

Swift 定义稳定的 C 互操作：导入 Clang 模块、把 `Int`/`UnsafePointer` 等与 C 类型对应，函数按平台 ABI（Apple 平台上的 ARM64 变体、x86_64 System V 等）调用。`@objc` 与 Objective-C 运行时桥接带来消息发送、自动桥接的 `NSString`/`String` 等；ARC 与 Objective-C ARC 交织，所有权注解（`__bridge` 等）错误会双释放或泄漏。

把 Swift 对象指针交给可能长期持有的 C 代码时，必须明确延长寿命或改用非托管/持久引用 API，否则 release 后悬空——同一 [FFI 寿命](../ch04/ffi.md) 问题。Swift 错误（`throws`）与 C 错误码、Objective-C `NSError**` 之间有桥接约定；不要假设异常可穿过任意 C 帧。

Swift 调用约定在寄存器选用与所有权传递上相对 C 有扩展；对 C 导出时通常收敛到 C ABI。对话里「能否直接把 Swift struct 扔给 C」要先对布局与 `@in`/`@out` 一类约定，而不是假定与 C `struct` 字节相同。

## 影响正确性与性能的语义

类型系统静态、名义与协议导向；泛型默认单态化/特化倾向，接近 [类型系统轴](../ch04/typing.md) 上的单态化端，同时保留抽象派发出口。`Optional` 是枚举，不是空指针语法糖；桥接到 Objective-C 可空引用时语义要单独对齐。

整数：`Int` 宽度随平台（32/64）；溢出默认在 debug trap、release 可回绕（可用 `&+` 等显式包装运算）。这与 C 有符号 UB、与 Java 恒定回绕都不同——跨语言 fib 必须声明用的哪套溢出语义。尾调用无普遍语言保证；递归深度与 ARC 流量一起决定朴素递归是否可读。

COW 使「看起来像拷贝」的赋值在唯一引用时变成指针拷贝；若在循环里触发写共享缓冲，复杂度与分配会突变。`escaping` 闭包把捕获提升到堆并纳入 ARC；异步续体同样是分配与生命周期源。

## 导师对话笔记

- 对比 Java/C#：主回收机制是 ARC 还是追踪 GC；环如何处理；赋值路径成本从哪来。
- 值类型 + COW：何时真拷贝缓冲区；`isUniquelyReferenced` 一类问题在对话里怎么陈述。
- `any P` 与泛型 `<T: P>`：存在性容器的分配与见证表间接，相对特化后的直连调用。
- 与 Objective-C 混编时的桥接流量：消息发送、自动桥接分配、线程上的自动释放池。
- 溢出语义：debug trap vs release 回绕；基准与正确性讨论必须固定构建配置。
- Actor 隔离解决什么：可发送性与共享可变状态，而不是「比锁更快」的口号。
