# JVM、.NET、Swift

托管运行时与 ARC 把对象寿命和一部分检查移出源码里的显式 `free`。Kotlin 放在 JVM 一节；Swift 的 ARC 与追踪式 GC 对照着写。

### 实现谱系与执行管线

源码经 `javac` / `kotlinc` 生成 `.class`（或 jar 内同类文件）。类加载器按名解析常量池引用，做链接与初始化；字节码由模板解释器或带 profiling 的解释路径执行。热点方法进入 C1 基线编译，再视热度进入 C2 或 Graal：内联、逃逸分析、锁消除、基于类型剖面的去虚化在此发生。墙钟时间因此拆成**冷启动（加载 + 解释）**、**升温（编译队列与去优化）**与**稳态**三段；短基准常落在前两段。

执行模型上，JVM 是典型的「字节码 VM + 多层 JIT」。AOT 路径（GraalVM Native Image、部分 Android ART 流程）把同一字节码语义提前编成机器码，换掉运行期剖面以换取启动与峰值可预期；对话时要先声明测的是哪条管线。Kotlin 的 `inline` 函数在字节码层展开，协程则编成状态机类；二者改变的是生成代码形态，不另起一套堆与 GC。

```mermaid
flowchart LR
  SRC[Java/Kotlin 源码] --> BC[.class 字节码]
  BC --> INT[解释 / C1]
  INT --> C2[C2 / Graal]
  C2 --> DEOPT[去优化回解释]
  DEOPT --> INT
```

读图顺序：冷路径走解释与 C1；热路径进优化编译；投机失败则去优化回到可再剖面的路径。

### 内存与对象表示

堆对象带 **mark word** 与 **类指针**（压缩普通对象指针 Compressed Oops 开启时更窄），其后是实例字段；数组另有长度。对象头宽度直接抬高「小对象」的下限成本。布局轴见 [对象表示与运行时布局](../ch04/object-layout.md)。

引用类型一律是指向堆对象的句柄（局部变量与操作数栈上的引用）；`int`/`long`/`double` 等原始类型可内联在栈帧与字段中。赋给 `Object`、泛型擦除后的 `T`、或装箱 API（`Integer`）时发生堆分配。HotSpot 的逃逸分析与标量替换可在证明未逃逸后把分配打回寄存器/栈；是否触发决定微观基准是否「突然变快」。

多态派发：虚方法经接收者类的虚方法表（或 itable 路径处理接口）；`invokevirtual` / `invokeinterface` 在解释器里走解析与缓存，在 JIT 里常被类型反馈收成单态或双态内联。间接层计数方式与 C++ vptr、Go interface 对照见 [派发机制与胖指针](../ch04/polymorphism-repr.md)。Kotlin 的接口默认方法、`inline class`（JVM 后端在可行时擦成底层类型）改变的是调用点能否去装箱，不是另一套对象头格式。

GC 为追踪式分代实现（具体收集器：G1、ZGC、Shenandoah 等）。跨代指针与并发标记依赖 [写屏障](../ch04/gc-mechanics.md)；给引用字段赋值的成本有时来自屏障而非赋值本身。

### 并发与运行时调度

平台线程一对一映射到 OS 线程；`synchronized`、`java.util.concurrent` 与 `volatile` 的语义由 **Java Memory Model（JMM）** 约束：数据竞争在 Java 中不是 C++ 式 UB，但仍可得到「有定义但难推理」的值。可见性与 happens-before 是对话时的正确切口，见 [并发模型](../ch04/concurrency.md)。

虚拟线程（Project Loom）把大量轻量任务多路复用到载体线程上：阻塞在受支持的点时卸载载体，而不是为每个任务永久占用一个大栈 OS 线程。与 Go goroutine 同属 M:N 谱系，但 pin 住载体的操作（某些 JNI、持有监视器的阻塞）会改变扩展性叙事。

Kotlin 协程是编译期状态机 + 用户选择的调度器（`Dispatchers.Default` 等），不是 JVM 内建绿色线程。挂起点由 `suspend` 函数标出；取消与上下文（`CoroutineContext`）沿状态机传递。把协程与虚拟线程混为一谈时，追问：调度实体是谁、阻塞谁、取消如何传播。

### 与 C / ABI 的边界

出 JVM 堆进入本地代码的主路径是 **JNI**（及后续 Panama/`Linker` API）。本地方法按平台调用约定进入 C；参数侧要把 Java 类型编成 C 可理解的表示，字符串与数组常涉及拷贝或临界区指针。JNI 局部/全局引用钉住对象，防止 GC 在本地侧仍持有时回收或搬迁——这是 [FFI 页](../ch04/ffi.md)「所有权与寿命」在 JVM 上的具体形式。

异常不能从 Java 栈帧「自然」穿过任意 C 帧再回到 Java：本地代码须用 JNI 异常检查 API 显式抛出或清除。反向调用（C 调 Java）要求当前线程已 `AttachCurrentThread`。基准里一层 JNI 封装的成本多来自编组、引用管理与安全点交互，而不是单条 `call` 指令。

Kotlin/Native 另有与 LLVM 的互操作与不同的内存模型；本页默认指 **Kotlin/JVM**。若对话落到多平台后端，先声明目标（JVM / Native / JS），再谈 ABI。

### 影响正确性与性能的语义

[类型系统轴](../ch04/typing.md) 上，Java 泛型是擦除：运行时看不到 `List<String>` 与 `List<Integer>` 的 `T`；桥接方法与装箱由此而来。Kotlin 在前端加强空安全与部分内联优化，但擦除边界仍在字节码处。名义类型与单继承 + 接口的派发结构决定 itable/vtable 形态。

整数：`int`/`long` 为二的补码定宽；语言层定义溢出回绕，没有 C 有符号溢出那种 [UB](../ch04/ub-abstract-machine.md)。`Integer` 缓存、自动装箱在循环里制造隐藏分配。`strictfp` 与浮点关联历史细节在现代默认路径上较少成为热点，但跨 JVM 讨论可复现数值时仍可能被问到。

尾调用不被语言保证优化；深度递归 fib 会砸栈或至少无法当成「与 C 同等调用约定距离」的算法。JIT 内联阈值、逃逸分析是否触发、GC 是否在测量窗口内停顿，都会让同构源码的排名抖动。

### 导师对话笔记

- 「Java 比 C 慢多少」要拆成：冷启动 vs 稳态、是否已 C2、有无装箱、GC 是否计入墙钟；不要用单次短跑回答。
- 解释 `synchronized` 时走到 JMM happens-before，而不是「加锁就变慢」的印象句。
- Kotlin 协程 vs 虚拟线程：前者是语言编译产物 + 库调度器，后者是 JVM 调度实体；可组合，不可等同。
- JNI 指针与 Java 引用的寿命：谁 `DeleteLocalRef`、谁钉住、临界区指针在 `Release*` 之后不可再用。
- 泛型擦除的后果：不能 `new T()`、数组协变与运行时 `ClassCastException` 检查落在边界上。
- Compressed Oops、对象头大小如何影响「创建大量小对象」的基准；追问堆参数与收集器名。

![C#](../assets/svg/csharp.svg){ width="40" }

C# 落在 CLR（.NET 运行时）上：源码编成 CIL（MSIL），由 RyuJIT 等在加载或首次执行时编成机器码，堆由追踪式 GC 管理。与 JVM 同属「托管字节码 + JIT + GC」谱系，但值类型、泛型具体化、委托与 P/Invoke 的落点不同；轴定义回链 [求值与执行模型](../ch04/evaluation.md)、[GC 机制](../ch04/gc-mechanics.md)、[FFI](../ch04/ffi.md)。

### 实现谱系与执行管线

`csc` / `dotnet` 工具链产出程序集（PE 内嵌 CIL 与元数据）。运行时加载程序集后，方法可在首次调用时 JIT，也可走 ReadyToRun / Native AOT 等预编译路径。RyuJIT 使用分层编译：快速质量较差的码先上，再按热度重编译；剖面驱动内联与去虚化。测量时同样要声明**启动、升温、稳态**落在哪一段，见执行模型轴。

语言层的 `async`/`await` 由编译器改写成状态机类型，在线程池或同步上下文上续跑；这与 JVM 上 Kotlin 协程同属「状态机 + 执行器」，只是类型与调度库不同。表达式树、动态调用站点（`dynamic` / DLR）走另一条缓存绑定路径，微观基准若碰到会偏离「普通虚调用」直觉。

Native AOT 把可到达代码编成平台机器码并裁剪元数据：启动快、反射与动态加载能力收缩。对话中「.NET 快不快」要先钉住 CoreCLR JIT 还是 Native AOT。

### 内存与对象表示

引用类型实例在 GC 堆上，带方法表指针等对象头；布局细节随运行时版本变化，对话抓住「头 + 字段、引用可被搬迁」即可，见 [对象布局](../ch04/object-layout.md)。**值类型（`struct`）**可内联在栈、数组与其它结构中，不经堆对象头——这是相对 Java「几乎一切皆堆上引用类型」的关键差异。值类型赋给接口或 `object` 时装箱成堆对象；泛型在 CLR 上对值类型**具体化**，可避免 Java 式强制装箱，代价是部分码膨胀。

数组与字符串是运行时内建类型；字符串不可变，切片型 API（`Span<T>` / `Memory<T>`）把连续内存视图做成可在栈上传递的胖视图，用于减少拷贝与分配。`ref struct` 限制逃逸出栈，把一部分安全契约放进编译期。

派发：虚方法经方法表；接口调用有自己的槽与可能的歧义解析。委托是可调用对象 + 目标，组合成多播时调用路径变长。与 C++/Go/Rust 间接层对照见 [派发与胖指针](../ch04/polymorphism-repr.md)。

GC 为分代追踪式（工作站/服务器模式、后台回收等变体）。写屏障与分配上下文影响字段写入与小对象分配成本，见 [写屏障](../ch04/gc-mechanics.md)。`stackalloc`、native 堆（`NativeMemory`）与 pinned 指针把「非 GC 堆」嵌进托管世界，成本出在边界钉住与泄漏责任上。

### 并发与运行时调度

`Thread` 映射到 OS 线程；`Task` / 线程池负责任务排队与续体执行。`async` 方法在 `await` 处挂起，返回 `Task`/`ValueTask`；同步上下文（UI 线程亲和）若捕获不当会造成死锁或多余封送。内存模型由 ECMA CLI / .NET 规范给出：`volatile`、`Interlocked`、`lock` 建立释放-获取类关系，数据竞争语义与 C++ UB 不同，与 Java 一样要按语言 MM 说，见 [并发模型](../ch04/concurrency.md)。

近年来的用户态协作（如部分场景下的 `Task` 调度优化）不改变「async 是状态机」这一层；若被问到绿线程，先对齐讨论的是运行时实验特性还是语言默认模型。

### 与 C / ABI 的边界

**P/Invoke** 按平台调用约定（Windows 上常见 `stdcall`/`cdecl` 历史包袱；.NET 在 Unix 上对齐 System V）进入本地库。结构体布局由 `[StructLayout]`、打包与字符集决定；字符串编组可选拷贝或固定。**C++/CLI** 与 **COM** 互操作是另一条 Windows 重路径；跨平台项目更常谈 P/Invoke 与 `LibraryImport` 源生成。

GC 堆对象传给可能持有指针的本地代码前必须钉住（`fixed`、`GCHandle`），否则搬迁导致悬空，见 FFI 页的寿命讨论。托管异常与 SEH/本地异常的交叉需要明确边界；默认做法是边界上换成错误码或 HRESULT 风格，而不是让异常穿透未知帧。

反向回调：把托管委托传给 C 时，委托必须保持存活，否则回调进已收集的入口。基准里「包一层 native」的开销拆成编组、钉住、调用约定转换三项看。

### 影响正确性与性能的语义

类型上，C# 静态名义类型为主；可空引用注解（nullable reference types）主要是编译期流分析，运行时仍是普通引用，与 Kotlin 空安全的擦除/检查落点类似地要问「运行时还剩什么」，见 [类型系统轴](../ch04/typing.md)。泛型对值类型具体化，使 `List<int>` 可连续存放；对引用类型共享实现。这直接影响「同构泛型容器」与 Java 的分配对比。

整数与浮点：定宽补码与 IEEE 浮点；`checked`/`unchecked` 上下文决定溢出是抛异常还是回绕。默认上下文下溢出回绕，与 C 有符号 UB 不同。`decimal` 为十进制 128 位量级表示，算术成本高于硬件二进制浮点。

闭包与 `async` 状态机是隐藏分配源：捕获变量提升到堆上显示对象；`ValueTask` 与结构体状态机优化试图收回一部分。JIT 是否内联、是否去虚、栈分配是否成功，会让同一算法的分配计数跳变。尾递归无语言级保证。

### 导师对话笔记

- 对比 JVM 时优先谈：**值类型 + 泛型具体化**、P/Invoke 与 JNI 的编组模型、async 状态机与线程池。
- 「struct 一定在栈上」不成立：字段在堆对象里、装箱、数组中的值类型元素，都要分开说。
- `Span<T>` 解决什么：连续内存视图、减少拷贝；`ref struct` 为何不能进堆上字段。
- 测 fib 时声明：`int` vs `long`、是否 `checked`、是否 Debug、是否已分层 JIT 升温。
- 钉住与 `GCHandle` 泄漏是托管/本地混合程序的常见正确性故障。
- Native AOT 下反射与动态加载的裁剪失败模式：编译期看得见的调用图 vs 运行期字符串激活。

![Swift](../assets/svg/swift.svg){ width="40" }

Swift 默认走 **LLVM** 后端的 AOT 管线（也可在 REPL/Playground 等场景下即时编译），堆上对象的主路径是 **ARC**（自动引用计数），而不是 JVM/CLR 式追踪式 GC。语言把值语义、写时复制与协议存在性（existential）放进类型系统；与 C / Objective-C 的边界是日常互操作面。共性轴见 [内存管理轴](../ch04/memory-management.md)、[求值与执行模型](../ch04/evaluation.md)、[FFI](../ch04/ffi.md)。

### 实现谱系与执行管线

`swiftc` 前端做类型检查与 SIL（Swift Intermediate Language）生成；SIL 上做所有权相关优化、泛型特化、ARC 流量削减（保留/释放消除、移动），再经 LLVM IR 进入与 C/C++/Rust 同族的选指令与寄存器分配，见第三部分 LLVM 链。发布构建是 AOT：墙钟通常不含 JIT 升温，但含模块序列化、增量编译与链接时间；Whole Module Optimization 改变内联与特化边界。

执行模型以本地机器码为主。动态特性（`AnyObject` 消息发送、Objective-C 运行时桥接）在热点上仍可能走消息派发与运行时查找；纯 Swift 泛型默认倾向特化，也可保留抽象调用以控制码尺寸。基准对比 JIT 语言时，不要把「无 warmup」误读成「绝对更快」——要比的是同阶段语义成本。

### 内存与对象表示

类实例在堆上，对象头含类型元数据与引用计数等；赋值与传参对类是共享引用并调整计数。结构体与枚举是值类型：复制语义下默认拷贝，大结构体经 LLVM 按 ABI 决定直接传递或隐藏指针。`Array`/`Dictionary`/`String` 等以写时复制（COW）缓冲实现：读共享缓冲，写时若唯一则原地改，否则复制——表面值语义，底层是引用计数缓冲，见 [对象布局](../ch04/object-layout.md) 与 RC 对照。

协议存在性容器（`any P`）携带值（或堆分配的盒子）与协议见证表/类型元数据，形态接近胖指针：数据 + 表。泛型特化成功时见证调用可变成直接调用；未特化则间接。与 `dyn Trait` / interface 的间接层对照见 [派发与胖指针](../ch04/polymorphism-repr.md)。

ARC 在引用写入时插入 `retain`/`release`（及 `unowned`/`weak` 变体）。环由 `weak`/`unowned` 打断；否则泄漏。追踪式 GC 的写屏障问题换成了赋值路径上的原子性计数与弱引用表，见 [GC 与 RC 对照](../ch04/gc-mechanics.md)。优化器在证明所有权转移后可消除成对 retain/release；微观基准对优化等级敏感。

### 并发与运行时调度

结构化并发以 `async`/`await` 与 `Task`/`Actor` 为中心。异步函数编成由合作式运行时调度的状态机；actor 串行化对隔离状态的访问，跨 actor 调用表现为异步消息。数据竞争在安全模型里靠隔离检查在编译期收紧；与「任意共享内存 + 锁」的 Java 线程模型切口不同，见 [并发模型](../ch04/concurrency.md)。

底层仍可创建线程、使用 GCD（Dispatch）队列；与 async 任务混用时要分清隔离域与队列亲和。原子操作与锁仍存在，用于与 C 互操作或绕过 actor 的场景；那一步重新打开数据竞争表面。

### 与 C / ABI 的边界

Swift 定义稳定的 C 互操作：导入 Clang 模块、把 `Int`/`UnsafePointer` 等与 C 类型对应，函数按平台 ABI（Apple 平台上的 ARM64 变体、x86_64 System V 等）调用。`@objc` 与 Objective-C 运行时桥接带来消息发送、自动桥接的 `NSString`/`String` 等；ARC 与 Objective-C ARC 交织，所有权注解（`__bridge` 等）错误会双释放或泄漏。

把 Swift 对象指针交给可能长期持有的 C 代码时，必须明确延长寿命或改用非托管/持久引用 API，否则 release 后悬空——同一 [FFI 寿命](../ch04/ffi.md) 问题。Swift 错误（`throws`）与 C 错误码、Objective-C `NSError**` 之间有桥接约定；不要假设异常可穿过任意 C 帧。

Swift 调用约定在寄存器选用与所有权传递上相对 C 有扩展；对 C 导出时通常收敛到 C ABI。对话里「能否直接把 Swift struct 扔给 C」要先对布局与 `@in`/`@out` 一类约定，而不是假定与 C `struct` 字节相同。

### 影响正确性与性能的语义

类型系统静态、名义与协议导向；泛型默认单态化/特化倾向，接近 [类型系统轴](../ch04/typing.md) 上的单态化端，同时保留抽象派发出口。`Optional` 是枚举，不是空指针语法糖；桥接到 Objective-C 可空引用时语义要单独对齐。

整数：`Int` 宽度随平台（32/64）；溢出默认在 debug trap、release 可回绕（可用 `&+` 等显式包装运算）。这与 C 有符号 UB、与 Java 恒定回绕都不同——跨语言 fib 必须声明用的哪套溢出语义。尾调用无普遍语言保证；递归深度与 ARC 流量一起决定朴素递归是否可读。

COW 使「看起来像拷贝」的赋值在唯一引用时变成指针拷贝；若在循环里触发写共享缓冲，复杂度与分配会突变。`escaping` 闭包把捕获提升到堆并纳入 ARC；异步续体同样是分配与生命周期源。

### 导师对话笔记

- 对比 Java/C#：主回收机制是 ARC 还是追踪 GC；环如何处理；赋值路径成本从哪来。
- 值类型 + COW：何时真拷贝缓冲区；`isUniquelyReferenced` 一类问题在对话里怎么陈述。
- `any P` 与泛型 `<T: P>`：存在性容器的分配与见证表间接，相对特化后的直连调用。
- 与 Objective-C 混编时的桥接流量：消息发送、自动桥接分配、线程上的自动释放池。
- 溢出语义：debug trap vs release 回绕；基准与正确性讨论必须固定构建配置。
- Actor 隔离解决什么：可发送性与共享可变状态，而不是「比锁更快」的口号。
