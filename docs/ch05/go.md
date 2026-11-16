# Go

![Go](../assets/svg/go.svg){ width="28" }

Go 工具链（`gc` 编译器）做 AOT，但带**用户态调度**、**追踪式 GC** 与把具体值装进 interface 时的表示变换。逃逸分析决定值在栈上还是堆上；GMP 模型把大量 goroutine 多路复用到 OS 线程；cgo 跨进 C 时要付线程与调用约定切换成本。轴见 [求值模型](../ch04/evaluation.md)、[内存管理](../ch04/memory-management.md)、[GC 机制](../ch04/gc-mechanics.md)、[并发](../ch04/concurrency.md)、[派发](../ch04/polymorphism-repr.md)、[FFI](../ch04/ffi.md)。

## 实现谱系与执行管线

标准实现是 Google 的 `gc`（`compile`/`link`）：前端到 SSA 中间表示，再生成目标码，链出带 Go runtime 的可执行文件。另有 `gccgo` 走 GCC 后端，语义与调度仍依赖 Go 运行时约定。默认 [AOT](../ch04/evaluation.md)，无 JVM 式分层 JIT；PGO 等可增强静态优化，但不改变「发布物是机器码 + runtime」的形态。

包是编译单元；接口满足关系在编译期按方法集检查（[结构型一面](../ch04/typing.md)），运行时 interface 值仍携带具体类型信息以便派发与断言。

## 内存与对象表示

分配器与 GC 管理堆；程序员不写 `free`。**逃逸分析**若证明引用未逃出函数，可把对象留在栈上，避免堆分配与 GC 压力；一旦取地址存入接口、闭包、全局或返回指针等，对象逃逸到堆——微观基准里「多一次 `&` 或赋给 interface」可改变分配曲线。GC 为并发标记清扫一类实现，带写屏障；停顿模型随版本演进，对话时问分代与 STW 时长，而不是品牌名，见 [GC 机制](../ch04/gc-mechanics.md)。

结构体布局由编译器按对齐规则排布；slice 是指针+长度+容量的描述符，string 是指针+长度。interface 值可视为类型信息 + 数据指针（具体布局有优化变体）：把非指针具体值赋给 interface 可能**装箱**到堆上，见 [对象布局](../ch04/object-layout.md) 与 [派发](../ch04/polymorphism-repr.md)。

## 并发与运行时调度

**GMP**：G（goroutine）是用户态轻量任务；M 是 OS 线程；P 是逻辑处理器/调度上下文，限制并行度（`GOMAXPROCS`）。G 在 P 上运行，阻塞或系统调用时 M 与 P 的绑可调整，运行时在停车场、channel、网络轮询点切入调度。channel 提供 CSP 风格通信；共享内存仍合法，竞态检测器（race detector）在调试构建中插桩。goroutine 栈可分段增长，与固定大栈的 OS 线程模型成本不同。

## 与 C / ABI 的边界

**cgo** 允许调用 C 符号并导出 Go 函数给 C。每次进入 C 可能涉及：遵循 C ABI 传递参数、处理 errno、以及调度器对「当前 M 卡在 C 里」的处理——可能占用 OS 线程、影响 G 的调度密度。指针规则严格：把 Go 指针传入 C 后，C 侧持有期间相关对象必须保持对 GC 可见且满足 cgo 指针传递限制；违反会导致堆损坏。回调从 C 进入 Go 要经 cgo 生成的过渡码。

成本往往来自线程占用、调用过渡与拷贝，而不是单次 `call` 指令本身。热路径上避免频繁 cgo；大块数据宜设计成少次跨越、明确所有权。纯 Go 汇编（`TEXT` 等）按 Go 的内部 ABI / 寄存器约定，与 SysV C ABI 不同——与 C 互操作仍应经 cgo 或显式过渡。

## 影响正确性与性能的语义

interface 方法调用：从动态类型取方法表再间接调用；与具体类型上的直接调用相比多一层，且赋值路径可能分配。类型断言与 type switch 依赖运行时类型信息。map、channel、select 的语义含并发安全边界；对 map 的无同步并发写是致命错误（可被检测器抓住），不是 C++ 式静默 UB 分类，但后果同样不可接受。

整数溢出在 Go 里对无符号与有符号是绕回的确定语义（与 C 有符号溢出 UB 不同）；除零等另有运行时 panic。基准要拆：逃逸是否把分配打到堆上、GC 是否进入测量窗口、goroutine 与 channel 是否测到调度、cgo 是否在圈内。朴素递归 fib 的跨语言排名几乎不能直接解释，见 [读基准](../ch06/reading-benchmarks.md)。

## 导师对话笔记

- 逃逸分析：引用逃逸则堆分配；赋给 interface、闭包捕获等是常见逃逸源。
- GC + 写屏障是托管路径成本的一部分；无手动 `free`，有停顿与吞吐模型。
- GMP：G 多路复用到 M，经 P 限制并行；阻塞点由 runtime 调度。
- interface = 类型信息 + 数据；方法调用间接，赋值可能装箱。
- cgo：C ABI + 可能占用 M；指针传递受 GC 与 cgo 规则约束，热路径少跨。
- 与 C/C++ 比：有符号溢出有定义绕回；并发 map 写是运行时可检的错误模式，不是「抽象机 UB」同一套词，但同样禁止。
