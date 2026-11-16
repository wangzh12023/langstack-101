# 端到端实验：从 C 到 LLVM IR 到目标码

用同一份递归 `fibonacci` 走完 `clang → LLVM IR → opt/llc → 目标码`，对照 `-O0` 与 `-O2`。读本页前应已浏览 [LLVM IR 核心概念](llvm-ir.md) 与 [SSA 形式与 φ 节点](ssa.md)；本实验把那些概念钉到具体文件上。

## 源码

```c
/* fib.c */
int fib(int n) {
  if (n < 2)
    return n;
  return fib(n - 1) + fib(n - 2);
}
```

递归双调用便于观察：优化前每个调用点清晰；优化后可能内联、尾部形态变化或仍保留双 `call`（取决于版本与旗标）。迭代版对照可回看 [手写 fibonacci](../ch02/fib-asm-lab.md)。

## 导出 IR

!!! example "实验步骤：IR"

    ```bash
    clang -O0 -emit-llvm -S -o fib.O0.ll fib.c
    clang -O2 -emit-llvm -S -o fib.O2.ll fib.c
    ```

在 `fib.O0.ll` 中查找：`alloca`、对 `n` 的 `store`/`load`、`icmp`、两个 `call`、`add`、基本块标签。局部量几乎都在栈槽上，SSA 名较少，接近「带类型的栈机」。

在 `fib.O2.ll` 中对比：`n` 是否仍以 `alloca` 出现；φ 是否出现在合流块；两个递归调用是否还在；是否多了属性或简化后的控制流。名字不再对应「源码里的一个变量一行」，而是数据流图上的定义。

也可用管道拆开前端与中端（效果随版本接近直接 `-O2`）：

```bash
clang -O0 -emit-llvm -S -o fib.raw.ll fib.c
opt -O2 -S -o fib.opt.ll fib.raw.ll
```

## 降到汇编与目标文件

!!! example "实验步骤：llc 与 objdump"

    ```bash
    llc -O0 -o fib.O0.s fib.O0.ll
    llc -O2 -o fib.O2.s fib.O2.ll
    clang -O0 -c -o fib.O0.o fib.c
    clang -O2 -c -o fib.O2.o fib.c
    objdump -d fib.O0.o
    objdump -d fib.O2.o
    ```

在 `fib.O0.s` / `objdump` 里指认：序言是否建栈帧、参数是否立刻存栈、`call fib` 是否成对出现、返回值是否在 `%eax`。在 `fib.O2.s` 里看：栈帧是否缩小或消失、寄存器是否直接承担 `n` 与临时和、调用序列是否仍遵守 [SysV](../ch02/sysv-amd64-abi.md)（参数 `%edi`、返回 `%eax`）。

`llc` 吃的是 IR，输出汇编；`clang -c` 走完整驱动。两者在同等级优化下应能对上同一套 ABI 与大致指令形态；若只改 IR 再 `llc`，等于跳过 clang 驱动里的部分机器通道开关，对比时以「是否同优化意图」为准。

## 观察清单

| 观察点 | `-O0` 常见现象 | `-O2` 常见现象 |
| --- | --- | --- |
| 局部 `n` | 栈槽 + load/store | 寄存器或 SSA 名，槽可能消失 |
| 控制流 | 块多、与源码行接近 | 块合并、判断可能被改写 |
| 递归调用 | 两个清晰 `call` | 仍可能保留双调用；或伴随其他简化 |
| 调试（加 `-g`） | 变量位置稳定 | 易出现 optimized out，见 [DWARF](debug-dwarf.md) |

合法性提醒：优化在有定义行为上遵守 as-if；本函数对正常 `n` 有定义，对照的是表示形式而非「算错结果」。中端类别说明见 [中端优化管线](midend-passes.md)；后端如何把虚拟寄存器收成物理寄存器见 [指令选择与寄存器分配](backend-isel-ra.md)。
