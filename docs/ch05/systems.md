# C、C++、Rust、Go


这四门都贴着机器与 ABI 说话，却把内存与并发的责任放在不同地方。按 C → C++ → Rust → Go 读，可以看到从「程序员与抽象机直接成交」走到「类型系统或运行时接管一部分成交」。细节机制能在第四章找到的，这里不重讲。


![C](../assets/svg/c.svg){ width="28" }

C 的实现几乎贴着平台 ABI：源级过程调用默认就是 [System V AMD64 ABI](../ch02/sysv-amd64-abi.md) 上的 C 调用约定；对象布局由实现定义的 `sizeof`/`alignof` 与结构体填充规则给出；没有隐藏的虚表、GC 或绿色线程。语义却不贴着具体 CPU——标准写在**抽象机**上，优化合法性由「程序不触发未定义行为」支撑。轴的定义见 [UB 与抽象机](../ch04/ub-abstract-machine.md)、[对象布局](../ch04/object-layout.md)、[求值与执行模型](../ch04/evaluation.md)。

### 实现谱系与执行管线

主流路径是预处理 → 编译前端（clang / gcc）→ 中端优化 → 目标码 → 汇编 / 集成汇编器 → 目标文件 `.o` → 链接得到可执行文件或共享库。clang 把 C 降到 LLVM IR 后走共享中端与后端，见 [clang 与其他前端](../ch03/frontends-on-llvm.md)；gcc 有自己的 GIMPLE/RTL 管线，落到同一类 ELF 与调用约定上。默认是 [AOT](../ch04/evaluation.md)：基准测的是编译后的机器码稳态，而不是解释或 warmup。

从翻译单元到 `.o` 的距离短：一个 `.c` 通常对应一个可重定位目标文件，符号、重定位与节布局在链接期与装载期解释（[ELF](../ch02/elf.md)、[重定位](../ch02/relocation.md)、[静态链接](../ch02/static-linking.md)）。`-O0` 到 `-O2`/`-O3` 改变的是抽象机程序在 as-if 规则下的改写强度；带进 IR 的 `nsw`、`noalias` 等旗标如何约束优化，见 [语言语义在 LLVM 边界上的损失](../ch03/llvm-semantics-limits.md)。

### 内存与对象表示

分配谱系是手动管理：`malloc`/`free`、栈对象、静态存储期。RAII 不在语言里；寿命由存储期与程序员约定的释放点决定，轴见 [内存管理](../ch04/memory-management.md)。结构体按实现的 ABI 与对齐规则排布字段与 padding；联合体与「把同一存储当另一类型读」受有效类型 / 严格别名规则约束——违反时进入 UB，而不是「按机器上的位型解释」。

标量与指针宽度、端序、`char` 是否有符号，属于实现定义；跨编译器或跨架构传递二进制布局时，要以双方的 ABI 与 `#pragma pack` / 属性为准。字符串是以空字节结尾的 `char` 序列，长度不在对象头里；越界读写没有语言级检查。

### 并发与运行时调度

C11 起有 `_Atomic`、线程库（如 `threads.h` 或 POSIX `pthread`）与互斥原语；执行实体是 OS 线程，没有语言内置的 M:N 调度器。数据竞争在 C 里属于未定义行为（与 C++ 同类）；`volatile` 只约束对**该对象**的抽象机访问不被优化掉，不提供多线程同步，也与 Java 的 `volatile` 不是同一契约。轴与硬件序见 [并发模型](../ch04/concurrency.md)、[内存序](../ch01/memory-ordering.md)。

### 与 C / ABI 的边界

对平台而言，C 就是 ABI 的参照语言：导出给其他语言的 `extern "C"` 符号、参数寄存器、结构体分类与返回约定，默认对齐 SysV（或目标平台的 C ABI）。从 C 再调 C 几乎零编组——没有所有权协议要翻译，也没有异常人格要穿越。与汇编互操作时，遵守同一调用约定与红区/栈对齐即可，见 [内联汇编](../ch02/inline-asm.md)。

代价是：所有边界问题（谁 `free`、指针活多久、回调在哪个线程）都落在文档与约定上，语言不强制。动态库符号与 PIC 形态见 [位置无关代码与动态链接](../ch02/pic-dynamic.md)。

### 影响正确性与性能的语义

有符号整数溢出、空/悬空/越界指针解引用、已结束生命周期对象的使用、违反严格别名、无序的冲突并发访问，都在抽象机里没有定义结果；编译器可以删除「只有在 UB 路径上才有意义」的检查。口试常用 `x+1 > x` 一类例子：在 `int` 上可被折成常量，改成无符号类型后回绕有定义，优化空间不同——细节与分类见 [UB 与抽象机](../ch04/ub-abstract-machine.md)。

性能上，C 接近「你写下的抽象机操作经优化后的机器码」：调用开销就是 ABI 调用，字段访问就是地址运算加 load/store，没有隐式写屏障或接口装箱。别名信息（`restrict`）、内联与 LTO/PGO 决定优化器能走多远；测 fib 一类微基准时，要确认整数宽度、是否尾调用、以及优化是否改写了算法本身（[读基准](../ch06/reading-benchmarks.md)）。

### 导师对话笔记

- C 语义在抽象机上；具体 CPU 上「这次跑通」不反驳 UB 下的优化合法性。
- 默认执行模型是 AOT 到机器码；`.c` → `.o` → 链接，无字节码/JIT 层。
- 对象无隐藏头；布局由 ABI 与对齐填充决定，`sizeof`/`alignof` 可观测。
- 与平台 C ABI 的距离近零：其他语言的 FFI 通常以「能调 C」为公约。
- 数据竞争与有符号溢出等属 UB；`volatile` 不是内存序原语。
- 内存管理是手动谱系；泄漏、use-after-free、double-free 是正确性成本，不是运行时停顿成本。


![C++](../assets/svg/cplusplus.svg){ width="28" }

C++ 在 C 的抽象机与 UB 传统上叠加**对象模型**（构造、析构、生命周期、基类子对象）、**模板单态化**与可选的异常展开。Itanium C++ ABI（Linux 上 clang/gcc 的事实标准）规定名字修饰、虚表与 RTTI 布局、异常人格与类型信息；对 C 导出的边界仍可通过 `extern "C"` 收成零开销的 SysV 调用。类型与派发轴见 [类型系统](../ch04/typing.md)、[派发与胖指针](../ch04/polymorphism-repr.md)；寿命与手动谱系见 [内存管理](../ch04/memory-management.md)。

### 实现谱系与执行管线

工具链与 C 同族：clang++ / g++，AOT 到目标码与 ELF。前端负责模板实例化、重载决议、对象模型 lowering；clang 路径上同样进入 LLVM IR（[前端接入](../ch03/frontends-on-llvm.md)）。模板与 `constexpr` 在编译期展开或求值，运行时看到的是单态化后的具体函数与类型布局——这是典型的[单态化](../ch04/typing.md)策略，码膨胀与可内联特化成对出现。

异常开启时，编译器为可能展开的栈帧生成 LSDA / call-site 表，运行时（如 libstdc++ / libc++abi）按 Itanium 异常处理 ABI 完成 catch 匹配与栈展开。`-fno-exceptions` 去掉这条路径与表体积。RTTI（`typeid` / `dynamic_cast`）依赖类型信息与 vtable 旁的元数据；可一并裁剪。

### 内存与对象表示

存储期仍分静态、线程、自动、动态；动态侧是 `new`/`delete` 与分配器，属手动管理上的结构化写法。**RAII** 把资源释放绑到析构函数与作用域结束，不改变「谁负责释放」的谱系归属。对象生命周期从构造完成开始、到析构开始结束；在生命周期外访问、或未启动生命周期就当对象用，落入 UB。

带虚函数的类对象通常含 **vptr**；虚调用路径是 load vptr → 取函数指针 → 间接 call，见 [派发机制](../ch04/polymorphism-repr.md)。多重继承与虚继承引入 thunk、`this` 调整与更复杂的 vtable / VTT 布局——口试级描述以「单继承虚调用 + this 调整存在」为底线，细节以 Itanium ABI 原文为准。成员布局遵循 ABI 的基类与成员排列规则，padding 与尾填充影响 `sizeof`；标准布局（standard-layout）类型才适合与 C 结构体在 FFI 上对穿。

### 并发与运行时调度

`std::thread` 映射到平台线程；`std::mutex`、`std::atomic`、内存序枚举把 [硬件内存序](../ch01/memory-ordering.md) 收成可移植接口。[数据竞争](../ch04/concurrency.md) 在 C++ 中定义为 UB。C++20 起的协程是状态机变换，执行仍落在调用方提供的调度器/执行器上，语言不附带 Go 式 M:N 运行时。

### 与 C / ABI 的边界

**Itanium C++ ABI 轮廓（Linux）**：符号经名字修饰（mangling）编码类型与重载；虚函数经 vtable；`type_info` 与异常类型匹配走 RTTI 结构；异常抛出走 `__cxa_throw` 一类入口与人格函数，中间帧需有匹配的展开信息。与 C 互操作时：`extern "C"` 关闭修饰、按 C 调用约定发射；可平凡复制、标准布局的类型按 C 聚合规则传递。虚函数、非平凡析构、或可能抛出的边界，不能假装成纯 C。

「零成本」边界成立的条件是：两侧约定同一 SysV 调用、无异常穿越未准备的帧、所有权与寿命在文档中对齐，且 C++ 侧不把带 vptr / 非平凡特殊成员的对象按 C `struct` 乱解析。成本来自编组与契约，而不是语言强制插入的 GC 或装箱层。轴见 [FFI](../ch04/ffi.md)。

### 影响正确性与性能的语义

继承 C 的抽象机与 UB 类别，并加上对象生命周期、严格别名在类型层次上的推论、以及数据竞争。移动语义与拷贝消除改变的是资源所有权与可观测副作用次数；优化器在 as-if 下仍可删除无副作用路径。异常路径上的表与可能的展开成本，使「是否抛」成为 ABI 与性能的一部分：热路径标 `noexcept` 可去掉部分运行时义务。

模板实例化决定最终码形：同一算法对不同 `T` 生成不同指令序列，可内联到接近手写 C；头文件实现导致的重复实例由链接期 COMDAT / ICF 等机制合并。虚调用在去虚化（devirtualization）或 `final` 成功时可退回直接 call；讨论零成本抽象时，要看最终 asm 是否还留下 vptr 负载。

### 导师对话笔记

- 对象生命周期与构造/析构次序是正确性轴；生命周期外访问是 UB，不是「析构后内存里还有位」。
- 模板默认单态化：特化与码体积同一枚硬币；与 Java 式擦除不可直接比基准。
- Linux 上 clang/gcc 的 C++ ABI 以 Itanium 为纲：mangling、vtable、RTTI、异常表。
- 虚调用：对象内 vptr → vtable → 间接 call；与 Rust fat pointer、Go interface 的间接层对照见第四章派发页。
- `extern "C"` + 标准布局可得与 C 同约定的边界；异常与非平凡对象不能安全假装成 C。
- 数据竞争是 UB；`std::atomic` 的 `memory_order` 才是多线程可见性契约。


![Rust](../assets/svg/rust.svg){ width="28" }

Rust 把[所有权与借用](../ch04/memory-management.md)放进编译期类型系统：安全子集里拒绝一批在 C/C++ 里会变成 UB 的别名与寿命错误，并把**数据竞争**挡在类型与 `Send`/`Sync` 边界之外。代码生成默认经 rustc 自有 IR（含 MIR）再接到 **LLVM** 后端，落成与 C 同台的机器码与 ELF；对外 `extern "C"` 按平台 C ABI 发射。轴见 [类型系统](../ch04/typing.md)、[UB 与抽象机](../ch04/ub-abstract-machine.md)、[FFI](../ch04/ffi.md)。

### 实现谱系与执行管线

`rustc` 完成解析、类型检查、借用检查与 MIR 上的语言专用变换，再 lowering 到 LLVM IR，其后与 clang 共享中端/后端能力（[前端接入](../ch03/frontends-on-llvm.md)）。另有实验性后端（如 Cranelift）用于更快的调试构建，生产默认仍是 LLVM。执行模型是 [AOT](../ch04/evaluation.md)；泛型默认**单态化**，单态化后的具体函数可被 LLVM 内联与特化。

`cargo`/`rustc` 产出 `rlib`、静态库或 `cdylib`/`dylib`；与 C 链接时符号与重定位规则仍是 ELF 世界的规则。调试信息走 DWARF；优化等级与 LTO 影响跨 crate 内联边界。

### 内存与对象表示

堆分配由库类型显式给出（`Box`、`Vec`、`String` 等），释放点由所有者离开作用域决定——仍是确定性析构，不是追踪式 GC。借用检查保证：共享借用（`&T`）期间无可变借用；可变借用（`&mut T`）独占。生命周期参数描述的是引用之间的约束关系，运行时通常不留下「生命周期对象」。

布局：`#[repr(Rust)]` 允许编译器重排字段；`#[repr(C)]` 对齐 C ABI 布局以便 FFI。切片与 `str`、以及 `&dyn Trait` / `Box<dyn Trait>` 使用**胖指针**（数据指针 + 长度或 vtable 指针），对象本身可不内嵌 vptr，见 [派发与胖指针](../ch04/polymorphism-repr.md)、[对象布局](../ch04/object-layout.md)。枚举的判别式与 payload 布局受 `repr` 影响；`Option<&T>` 一类空指针优化是表示层的落点。

### 并发与运行时调度

`std::thread` 是 OS 线程。所有权与 `Send`/`Sync` 约束哪些类型能跨线程转移或共享；安全代码中数据竞争被类型系统排除——这是语言给出的**数据竞争自由**边界，不是「不可能有竞态条件」：逻辑上的 TOCTOU、死锁、业务层时序错误仍在。`async`/`.await` 把函数编译成状态机（`Future`），需执行器驱动；取消与 `Pin` 是异步状态机特有的表示约束。轴见 [并发模型](../ch04/concurrency.md)。

### 与 C / ABI 的边界

`extern "C"` 函数按 C 调用约定生成；`#[repr(C)]` 结构体与 C 头文件对齐。从安全 Rust 进入 FFI 必须经过 `unsafe`：编译器不再证明对端遵守借用与别名规则。所有权约定（谁 `free`、指针别名是否等于 `&mut`）写在 API 契约里；违反时回到语言级 UB。

`panic` 默认可展开栈（类似人格与落地实现相关）；展开穿过外语帧或 `panic=abort` 策略下的行为不同。跨 FFI 应把失败收成错误码或显式结果类型，而不是让 panic 穿越 C 栈。回调进 Rust 时还要保证线程与可重入符合对端运行时假设。

### 影响正确性与性能的语义

安全子集把一批空间安全与别名问题前移到编译期；`unsafe` 块用于实现抽象或 FFI，其义务是维持模块对外的安全不变量——漏掉不变量等于重新引入 UB，优化器仍可按「无 UB」假定改写。`&mut` 的不别名保证可下传到 LLVM 的 `noalias` 一类属性，成为优化火力，见 [LLVM 语义边界](../ch03/llvm-semantics-limits.md)。

动态派发（`dyn Trait`）保留间接调用；静态派发（泛型单态化、`impl Trait` 返回位）可内联到接近 C。边界检查在 Debug 与部分 Release 配置下保留；消除检查依赖证明或 `unsafe` 的无检查索引。`panic` 与分配失败策略（abort vs unwind）影响二进制是否依赖展开表及失败路径成本。

### 导师对话笔记

- 安全 Rust：所有权/借用在编译期；数据竞争在安全子集被拒绝。`unsafe` 把契约交回作者。
- 默认经 MIR → LLVM → 机器码；泛型单态化，不是 JVM 式擦除。
- `&dyn Trait` 是引用侧胖指针（data + vtable）；对象可保持朴素布局，便于 `repr(C)` 与嵌入。
- FFI：`extern "C"` + `repr(C)`；寿命与别名必须在边界上人工对齐，panic 不宜穿越 C 帧。
- 「无数据竞争」≠「无竞态条件」；async 是状态机 + 执行器，不是内置 GMP。
- 性能对话先问最终是单态化直连还是 `dyn` 间接，再问边界检查与分配是否还在热路径上。


![Go](../assets/svg/go.svg){ width="28" }

Go 工具链（`gc` 编译器）做 AOT，但带**用户态调度**、**追踪式 GC** 与把具体值装进 interface 时的表示变换。逃逸分析决定值在栈上还是堆上；GMP 模型把大量 goroutine 多路复用到 OS 线程；cgo 跨进 C 时要付线程与调用约定切换成本。轴见 [求值模型](../ch04/evaluation.md)、[内存管理](../ch04/memory-management.md)、[GC 机制](../ch04/gc-mechanics.md)、[并发](../ch04/concurrency.md)、[派发](../ch04/polymorphism-repr.md)、[FFI](../ch04/ffi.md)。

### 实现谱系与执行管线

标准实现是 Google 的 `gc`（`compile`/`link`）：前端到 SSA 中间表示，再生成目标码，链出带 Go runtime 的可执行文件。另有 `gccgo` 走 GCC 后端，语义与调度仍依赖 Go 运行时约定。默认 [AOT](../ch04/evaluation.md)，无 JVM 式分层 JIT；PGO 等可增强静态优化，但不改变「发布物是机器码 + runtime」的形态。

包是编译单元；接口满足关系在编译期按方法集检查（[结构型一面](../ch04/typing.md)），运行时 interface 值仍携带具体类型信息以便派发与断言。

### 内存与对象表示

分配器与 GC 管理堆；程序员不写 `free`。**逃逸分析**若证明引用未逃出函数，可把对象留在栈上，避免堆分配与 GC 压力；一旦取地址存入接口、闭包、全局或返回指针等，对象逃逸到堆——微观基准里「多一次 `&` 或赋给 interface」可改变分配曲线。GC 为并发标记清扫一类实现，带写屏障；停顿模型随版本演进，对话时问分代与 STW 时长，而不是品牌名，见 [GC 机制](../ch04/gc-mechanics.md)。

结构体布局由编译器按对齐规则排布；slice 是指针+长度+容量的描述符，string 是指针+长度。interface 值可视为类型信息 + 数据指针（具体布局有优化变体）：把非指针具体值赋给 interface 可能**装箱**到堆上，见 [对象布局](../ch04/object-layout.md) 与 [派发](../ch04/polymorphism-repr.md)。

### 并发与运行时调度

**GMP**：G（goroutine）是用户态轻量任务；M 是 OS 线程；P 是逻辑处理器/调度上下文，限制并行度（`GOMAXPROCS`）。G 在 P 上运行，阻塞或系统调用时 M 与 P 的绑可调整，运行时在停车场、channel、网络轮询点切入调度。channel 提供 CSP 风格通信；共享内存仍合法，竞态检测器（race detector）在调试构建中插桩。goroutine 栈可分段增长，与固定大栈的 OS 线程模型成本不同。

### 与 C / ABI 的边界

**cgo** 允许调用 C 符号并导出 Go 函数给 C。每次进入 C 可能涉及：遵循 C ABI 传递参数、处理 errno、以及调度器对「当前 M 卡在 C 里」的处理——可能占用 OS 线程、影响 G 的调度密度。指针规则严格：把 Go 指针传入 C 后，C 侧持有期间相关对象必须保持对 GC 可见且满足 cgo 指针传递限制；违反会导致堆损坏。回调从 C 进入 Go 要经 cgo 生成的过渡码。

成本往往来自线程占用、调用过渡与拷贝，而不是单次 `call` 指令本身。热路径上避免频繁 cgo；大块数据宜设计成少次跨越、明确所有权。纯 Go 汇编（`TEXT` 等）按 Go 的内部 ABI / 寄存器约定，与 SysV C ABI 不同——与 C 互操作仍应经 cgo 或显式过渡。

### 影响正确性与性能的语义

interface 方法调用：从动态类型取方法表再间接调用；与具体类型上的直接调用相比多一层，且赋值路径可能分配。类型断言与 type switch 依赖运行时类型信息。map、channel、select 的语义含并发安全边界；对 map 的无同步并发写是致命错误（可被检测器抓住），不是 C++ 式静默 UB 分类，但后果同样不可接受。

整数溢出在 Go 里对无符号与有符号是绕回的确定语义（与 C 有符号溢出 UB 不同）；除零等另有运行时 panic。基准要拆：逃逸是否把分配打到堆上、GC 是否进入测量窗口、goroutine 与 channel 是否测到调度、cgo 是否在圈内。朴素递归 fib 的跨语言排名几乎不能直接解释，见 [读基准](../ch06/reading-benchmarks.md)。

### 导师对话笔记

- 逃逸分析：引用逃逸则堆分配；赋给 interface、闭包捕获等是常见逃逸源。
- GC + 写屏障是托管路径成本的一部分；无手动 `free`，有停顿与吞吐模型。
- GMP：G 多路复用到 M，经 P 限制并行；阻塞点由 runtime 调度。
- interface = 类型信息 + 数据；方法调用间接，赋值可能装箱。
- cgo：C ABI + 可能占用 M；指针传递受 GC 与 cgo 规则约束，热路径少跨。
- 与 C/C++ 比：有符号溢出有定义绕回；并发 map 写是运行时可检的错误模式，不是「抽象机 UB」同一套词，但同样禁止。
