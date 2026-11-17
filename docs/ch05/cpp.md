# C++

![C++](../assets/svg/cplusplus.svg){ width="28" }

C++ 在 C 的抽象机与 UB 传统上叠加**对象模型**（构造、析构、生命周期、基类子对象）、**模板单态化**与可选的异常展开。Itanium C++ ABI（Linux 上 clang/gcc 的事实标准）规定名字修饰、虚表与 RTTI 布局、异常人格与类型信息；对 C 导出的边界仍可通过 `extern "C"` 收成零开销的 SysV 调用。类型与派发轴见 [类型系统](../ch04/typing.md)、[派发与胖指针](../ch04/polymorphism-repr.md)；寿命与手动谱系见 [内存管理](../ch04/memory-management.md)。

## 实现谱系与执行管线

工具链与 C 同族：clang++ / g++，AOT 到目标码与 ELF。前端负责模板实例化、重载决议、对象模型 lowering；clang 路径上同样进入 LLVM IR（[前端接入](../ch03/frontends-on-llvm.md)）。模板与 `constexpr` 在编译期展开或求值，运行时看到的是单态化后的具体函数与类型布局——这是典型的[单态化](../ch04/typing.md)策略，码膨胀与可内联特化成对出现。

异常开启时，编译器为可能展开的栈帧生成 LSDA / call-site 表，运行时（如 libstdc++ / libc++abi）按 Itanium 异常处理 ABI 完成 catch 匹配与栈展开。`-fno-exceptions` 去掉这条路径与表体积。RTTI（`typeid` / `dynamic_cast`）依赖类型信息与 vtable 旁的元数据；可一并裁剪。

## 内存与对象表示

存储期仍分静态、线程、自动、动态；动态侧是 `new`/`delete` 与分配器，属手动管理上的结构化写法。**RAII** 把资源释放绑到析构函数与作用域结束，不改变「谁负责释放」的谱系归属。对象生命周期从构造完成开始、到析构开始结束；在生命周期外访问、或未启动生命周期就当对象用，落入 UB。

带虚函数的类对象通常含 **vptr**；虚调用路径是 load vptr → 取函数指针 → 间接 call，见 [派发机制](../ch04/polymorphism-repr.md)。多重继承与虚继承引入 thunk、`this` 调整与更复杂的 vtable / VTT 布局——讨论时至少要能说明「单继承虚调用 + this 调整存在」为底线，细节以 Itanium ABI 原文为准。成员布局遵循 ABI 的基类与成员排列规则，padding 与尾填充影响 `sizeof`；标准布局（standard-layout）类型才适合与 C 结构体在 FFI 上对穿。

## 并发与运行时调度

`std::thread` 映射到平台线程；`std::mutex`、`std::atomic`、内存序枚举把 [硬件内存序](../ch01/memory-ordering.md) 收成可移植接口。[数据竞争](../ch04/concurrency.md) 在 C++ 中定义为 UB。C++20 起的协程是状态机变换，执行仍落在调用方提供的调度器/执行器上，语言不附带 Go 式 M:N 运行时。

## 与 C / ABI 的边界

**Itanium C++ ABI 轮廓（Linux）**：符号经名字修饰（mangling）编码类型与重载；虚函数经 vtable；`type_info` 与异常类型匹配走 RTTI 结构；异常抛出走 `__cxa_throw` 一类入口与人格函数，中间帧需有匹配的展开信息。与 C 互操作时：`extern "C"` 关闭修饰、按 C 调用约定发射；可平凡复制、标准布局的类型按 C 聚合规则传递。虚函数、非平凡析构、或可能抛出的边界，不能假装成纯 C。

「零成本」边界成立的条件是：两侧约定同一 SysV 调用、无异常穿越未准备的帧、所有权与寿命在文档中对齐，且 C++ 侧不把带 vptr / 非平凡特殊成员的对象按 C `struct` 乱解析。成本来自编组与契约，而不是语言强制插入的 GC 或装箱层。轴见 [FFI](../ch04/ffi.md)。

## 影响正确性与性能的语义

继承 C 的抽象机与 UB 类别，并加上对象生命周期、严格别名在类型层次上的推论、以及数据竞争。移动语义与拷贝消除改变的是资源所有权与可观测副作用次数；优化器在 as-if 下仍可删除无副作用路径。异常路径上的表与可能的展开成本，使「是否抛」成为 ABI 与性能的一部分：热路径标 `noexcept` 可去掉部分运行时义务。

模板实例化决定最终码形：同一算法对不同 `T` 生成不同指令序列，可内联到接近手写 C；头文件实现导致的重复实例由链接期 COMDAT / ICF 等机制合并。虚调用在去虚化（devirtualization）或 `final` 成功时可退回直接 call；讨论零成本抽象时，要看最终 asm 是否还留下 vptr 负载。

## 导师对话笔记

- 对象生命周期与构造/析构次序是正确性轴；生命周期外访问是 UB，不是「析构后内存里还有位」。
- 模板默认单态化：特化与码体积同一枚硬币；与 Java 式擦除不可直接比基准。
- Linux 上 clang/gcc 的 C++ ABI 以 Itanium 为纲：mangling、vtable、RTTI、异常表。
- 虚调用：对象内 vptr → vtable → 间接 call；与 Rust fat pointer、Go interface 的间接层对照见第四章派发页。
- `extern "C"` + 标准布局可得与 C 同约定的边界；异常与非平凡对象不能安全假装成 C。
- 数据竞争是 UB；`std::atomic` 的 `memory_order` 才是多线程可见性契约。
