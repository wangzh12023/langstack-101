# 语言栈 101

看 [跨语言 fibonacci 对照](https://benjdd.com/languages/) 一类结果时，时间差通常来自调用开销、整数语义、分配与 GC、JIT 是否已升温，而不是「语言名字」本身。本站按实现路径写这些机制：从机器表示到汇编与进程，再到编译器与各语言运行时。

<div class="grid cards" markdown>

-   :material-cpu-64-bit: **机器侧的表示与执行**

    ---

    整数与浮点、流水线与缓存、虚存、内存序、指令集契约

    [第一章](ch01/index.md)

-   :material-chip: **从汇编到正在运行的进程**

    ---

    指令、调用约定、链接与装载、内核边界

    [第二章](ch02/index.md)

-   :simple-llvm: **编译器对执行路径的改写**

    ---

    LLVM IR、SSA、优化、后端与调试信息

    [第三章](ch03/index.md)

-   :material-axis-arrow: **语言差异与对照**

    ---

    类型、内存、执行与并发等轴，以及各语言实现

    [第四章](ch04/index.md)

</div>

依赖与实验环境见 [阅读说明](guide.md)。自测条目见 [口试量规](Spec/oral-rubric.md)。
