# 导师对话清单

按三个主题域自测。每条回链到正文；不能只背短语。口试题全文见 [口试量规](../Spec/oral-rubric.md)。

## 系统与 ABI

- 画出 `call` 前后 `rsp`、返回地址与 16 字节对齐的关系；说明红区适用场景。[SysV ABI](../ch02/calling.md) · [栈帧](../ch02/calling.md)
- 背出 SysV 前六个整型参数寄存器，以及两个 `double` 的 XMM 位置。[SysV ABI](../ch02/calling.md)
- 区分 caller-saved 与 callee-saved，并说明写错的后果。[寄存器](../ch02/instructions.md) · [SysV ABI](../ch02/calling.md)
- 解释入口不是 `main`：`_start`、libc 初始化、到 `main` 之前的路径。[进程入口](../ch02/link-load.md) · [装载](../ch02/link-load.md)
- 叙述一次外部调用的 PLT/GOT 路径，以及懒绑定前后 GOT 槽内容差异。[PIC 与动态链接](../ch02/link-load.md)
- 说明 `-fPIC` 下访问全局量为何常经 GOT 或 RIP-relative 形态。[PIC](../ch02/link-load.md) · [编码与 RIP-relative](../ch02/instructions.md)

## 内存与并发

- 用一句话区分数据竞争与竞态条件；指出至少一种语言对数据竞争的定义。[并发模型](../ch04/axes-boundary.md)
- 说明 store buffer / 乱序执行为何让「写在前、读在后」的源码顺序不等于全局可见顺序。[内存序](../ch01/machine.md)
- 解释 happens-before 或等价约束在讨论 `atomic` / `volatile`（各语言含义不同）时的位置。[内存序](../ch01/machine.md) · [原子指令](../ch02/kernel-boundary.md)
- 对照 tracing GC 的写屏障与 RC/ARC 的循环引用问题。[GC](../ch04/axes-core.md) · [内存管理轴](../ch04/axes-core.md)
- 说明伪共享：两个逻辑变量如何因同一缓存行而互相拖累。[缓存](../ch01/machine.md)

## 语言实现取舍

- 对给定语言，指出其默认执行模型（AOT / 字节码+JIT / 解释）及基准数字可能测到的阶段。[执行模型](../ch04/axes-core.md) · [跨语言对照表](../ch05/cross-language-matrix.md)
- 举一例类型擦除与一例单态化，并说明对码体积与特化优化的影响。[类型系统轴](../ch04/axes-core.md)
- 画出 C++ 虚调用、Go interface、Rust `dyn Trait` 的间接层差异。[派发与胖指针](../ch04/axes-boundary.md)
- 说明 FFI 边界上：调用约定要对齐什么，异常/panic 为何不能随意穿越。[FFI](../ch04/axes-boundary.md)
- 举一例「语言语义在进入 LLVM IR 后已固定或丢失」（别名、生命周期、provenance 等）。[LLVM 语义边界](../ch03/ir-and-opt.md)

## 基准

- 对朴素递归 fibonacci，列出至少四条会改变跨语言名次解释的因素（整数语义、优化是否改写算法、JIT warmup、调用开销、GC/分配等）。[性能数字的读法](reading-benchmarks.md)
- 把「谁最快」改述成「谁在这条路径上贴近何种机器行为」。[跨语言对照表](../ch05/cross-language-matrix.md) · [源码到硅片](source-to-silicon.md)
