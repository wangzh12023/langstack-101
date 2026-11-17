# 语言栈 101

跨语言排行和 [naive fibonacci 对照](https://benjdd.com/languages/) 很容易让人停在「谁更快」上。要接着问下去——快在调用、分配、整数语义，还是 JIT 根本还没升温——就得手里有一张从源码到机器行为的地图。本站按这张地图写，不按语言名词表写。

<div class="grid cards" markdown>

-   :material-cpu-64-bit: **机器在算什么**

    ---

    先固定：比特、存储层次、指令契约。后面所有「变量 / 调用 / 优化」都落在这上面。

    [第一章](ch01/index.md)

-   :material-chip: **从汇编到进程**

    ---

    全书脊梁：指令怎么写、函数怎么交接、分离编译的碎片怎样变成正在跑的进程。

    [第二章](ch02/index.md)

-   :simple-llvm: **编译器改写路径**

    ---

    LLVM 一类工具在中间改写你在第二章建立的对应关系；优化合法与否取决于语言抽象机。

    [第三章](ch03/index.md)

-   :material-axis-arrow: **语言差异落在哪**

    ---

    用几条轴读各语言，再看 C/Rust/JVM/Python 等如何选点；最后回到基准数字怎么读。

    [第四章](ch04/index.md)

</div>

读法与依赖见 [怎么读](guide.md)。能和导师把机制谈清楚的自测标准见 [口试量规](Spec/oral-rubric.md)。
