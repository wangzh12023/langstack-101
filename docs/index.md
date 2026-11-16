# 语言栈 101

<div class="grid cards" markdown>

-   :material-layers-triple: **机器与表示**

    ---

    比特、浮点、流水线、缓存、虚存、内存序与 ISA

    [:octicons-arrow-right-24: 进入第一章](ch01/index.md)

-   :material-chip: **汇编与二进制**

    ---

    从寄存器、ABI、ELF 到装载与手写实验

    [:octicons-arrow-right-24: 进入第二章](ch02/index.md)

-   :simple-llvm: **编译器与 LLVM**

    ---

    IR、SSA、优化、后端 lowering 与语义边界

    [:octicons-arrow-right-24: 进入第三章](ch03/index.md)

-   :material-axis-arrow: **语言共性与分论**

    ---

    用同一组轴读类型、内存、并发与各语言落点

    [:octicons-arrow-right-24: 进入第四章](ch04/index.md)

</div>

## 这本书写什么

本站从硬件契约往上讲到语言实现：先建立「一条语句如何变成机器上的状态变化」的模型，再解释 LLVM 一类工具链在中间做了什么，最后用同一套轴去读 C、Rust、Go、JVM、Python、JavaScript 等语言。触发点可以是各类语言排行或 [naive fibonacci 对照](https://benjdd.com/languages/)，正文关心的是机制，而不是名次。

完成后的能力标准写在 [口试量规](Spec/oral-rubric.md)：能和导师把 ABI、UB、链接、内存序、派发和基准读法谈清楚。

## 阅读顺序

主线按章节编号前进。第一章的内存序与 ISA 两节是后文前置；语言分论页建立在第四章的轴上，分论只写落点与差异。

详细依赖与实验环境见 [阅读说明](guide.md)；目录修订与阶段计划见 [课程大纲与验收](Spec/outline.md)。

## 体例

主题配色与版式对齐 [Linux 101](https://101.lug.ustc.edu.cn/) 使用的 Material for MkDocs（indigo）。写作约束见 [编写规范](appendix/writing-spec.md)。
