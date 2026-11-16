# Rust

![Rust](../assets/svg/rust.svg){ width="28" }

Rust 把[所有权与借用](../ch04/memory-management.md)放进编译期类型系统：安全子集里拒绝一批在 C/C++ 里会变成 UB 的别名与寿命错误，并把**数据竞争**挡在类型与 `Send`/`Sync` 边界之外。代码生成默认经 rustc 自有 IR（含 MIR）再接到 **LLVM** 后端，落成与 C 同台的机器码与 ELF；对外 `extern "C"` 按平台 C ABI 发射。轴见 [类型系统](../ch04/typing.md)、[UB 与抽象机](../ch04/ub-abstract-machine.md)、[FFI](../ch04/ffi.md)。

## 实现谱系与执行管线

`rustc` 完成解析、类型检查、借用检查与 MIR 上的语言专用变换，再 lowering 到 LLVM IR，其后与 clang 共享中端/后端能力（[前端接入](../ch03/frontends-on-llvm.md)）。另有实验性后端（如 Cranelift）用于更快的调试构建，生产默认仍是 LLVM。执行模型是 [AOT](../ch04/evaluation.md)；泛型默认**单态化**，单态化后的具体函数可被 LLVM 内联与特化。

`cargo`/`rustc` 产出 `rlib`、静态库或 `cdylib`/`dylib`；与 C 链接时符号与重定位规则仍是 ELF 世界的规则。调试信息走 DWARF；优化等级与 LTO 影响跨 crate 内联边界。

## 内存与对象表示

堆分配由库类型显式给出（`Box`、`Vec`、`String` 等），释放点由所有者离开作用域决定——仍是确定性析构，不是追踪式 GC。借用检查保证：共享借用（`&T`）期间无可变借用；可变借用（`&mut T`）独占。生命周期参数描述的是引用之间的约束关系，运行时通常不留下「生命周期对象」。

布局：`#[repr(Rust)]` 允许编译器重排字段；`#[repr(C)]` 对齐 C ABI 布局以便 FFI。切片与 `str`、以及 `&dyn Trait` / `Box<dyn Trait>` 使用**胖指针**（数据指针 + 长度或 vtable 指针），对象本身可不内嵌 vptr，见 [派发与胖指针](../ch04/polymorphism-repr.md)、[对象布局](../ch04/object-layout.md)。枚举的判别式与 payload 布局受 `repr` 影响；`Option<&T>` 一类空指针优化是表示层的落点。

## 并发与运行时调度

`std::thread` 是 OS 线程。所有权与 `Send`/`Sync` 约束哪些类型能跨线程转移或共享；安全代码中数据竞争被类型系统排除——这是语言给出的**数据竞争自由**边界，不是「不可能有竞态条件」：逻辑上的 TOCTOU、死锁、业务层时序错误仍在。`async`/`.await` 把函数编译成状态机（`Future`），需执行器驱动；取消与 `Pin` 是异步状态机特有的表示约束。轴见 [并发模型](../ch04/concurrency.md)。

## 与 C / ABI 的边界

`extern "C"` 函数按 C 调用约定生成；`#[repr(C)]` 结构体与 C 头文件对齐。从安全 Rust 进入 FFI 必须经过 `unsafe`：编译器不再证明对端遵守借用与别名规则。所有权约定（谁 `free`、指针别名是否等于 `&mut`）写在 API 契约里；违反时回到语言级 UB。

`panic` 默认可展开栈（类似人格与落地实现相关）；展开穿过外语帧或 `panic=abort` 策略下的行为不同。跨 FFI 应把失败收成错误码或显式结果类型，而不是让 panic 穿越 C 栈。回调进 Rust 时还要保证线程与可重入符合对端运行时假设。

## 影响正确性与性能的语义

安全子集把一批空间安全与别名问题前移到编译期；`unsafe` 块用于实现抽象或 FFI，其义务是维持模块对外的安全不变量——漏掉不变量等于重新引入 UB，优化器仍可按「无 UB」假定改写。`&mut` 的不别名保证可下传到 LLVM 的 `noalias` 一类属性，成为优化火力，见 [LLVM 语义边界](../ch03/llvm-semantics-limits.md)。

动态派发（`dyn Trait`）保留间接调用；静态派发（泛型单态化、`impl Trait` 返回位）可内联到接近 C。边界检查在 Debug 与部分 Release 配置下保留；消除检查依赖证明或 `unsafe` 的无检查索引。`panic` 与分配失败策略（abort vs unwind）影响二进制是否依赖展开表及失败路径成本。

## 导师对话笔记

- 安全 Rust：所有权/借用在编译期；数据竞争在安全子集被拒绝。`unsafe` 把契约交回作者。
- 默认经 MIR → LLVM → 机器码；泛型单态化，不是 JVM 式擦除。
- `&dyn Trait` 是引用侧胖指针（data + vtable）；对象可保持朴素布局，便于 `repr(C)` 与嵌入。
- FFI：`extern "C"` + `repr(C)`；寿命与别名必须在边界上人工对齐，panic 不宜穿越 C 帧。
- 「无数据竞争」≠「无竞态条件」；async 是状态机 + 执行器，不是内置 GMP。
- 性能对话先问最终是单态化直连还是 `dyn` 间接，再问边界检查与分配是否还在热路径上。
