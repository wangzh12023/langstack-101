# 原子指令与内存序指令

多线程下，普通 `mov`/`add` 不保证「读改写」对其他核表现为一步，也不自动提供跨线程的发布/获取语义。x86-64 用带 `lock` 前缀的读改写、`xchg`，以及 `mfence`/`lfence`/`sfence` 等栅栏表达这些约束。语言层的 `memory_order` 最终要落到这类指令或对其省略的证明上；硬件序的直觉见 [存储缓冲与内存序](../ch01/memory-ordering.md)。

## LOCK 前缀与读改写

`lock` 前缀作用于后随的读改写内存指令，使该操作在缓存一致性协议下对其他观察者表现为原子：

```asm
    lock addq $1, (%rdi)
    lock cmpxchgq %rsi, (%rdi)
    lock xaddq %rax, (%rdi)
```

`xchg` 与内存操作数搭配时，即使不写 `lock` 也有锁总线/锁缓存行语义。`cmpxchg` 比较 `%rax` 与目的内存：相等则写入源寄存器值并置 `ZF`，否则把内存旧值装入 `%rax` 并清 `ZF`——CAS 循环的底座。

无 `lock` 的普通 `addq $1, (%rdi)` 在单线程语义下正确，在并发计数上可能丢失更新。

## 内存序指令

x86-64 对普通用户态存储/加载提供较强的总存储序（TSO）外观：商店缓冲仍允许「自己的读看到自己未引退的写」等局部效应，但对多数编程模型已比弱序架构少补栅栏。显式栅栏仍出现在：

```asm
    mfence                  # 全栅栏：读写均不可跨越
    lfence
    sfence
```

编译器为 C++/`std::atomic` 的 `seq_cst` 存储等可能插入 `mfence` 或改用 `xchg`；`acquire`/`release` 在 x86 上常仅靠普通加载/存储加编译器屏障即可，因为硬件已禁止多数乱序组合。把「语言序」等同于「一定生成 mfence」会误判 `-O2` 输出。

## 与编译器屏障的分工

内联汇编里的 `memory` clobber 阻止编译器跨过该语句重排对内存的缓存假设，但不生成 CPU 栅栏。CPU 栅栏阻止其他核对访问的可见性重排。两者都需要时，既要 `volatile`/atomic 语义或 `memory` clobber，又要 `mfence` 一类指令（或等价的 `lock` 操作）。扩展语法见 [内联汇编](inline-asm.md)。

## 阅读练习

对一段含 `std::atomic::fetch_add` 的 C++，分别编译：

```bash
clang++ -O2 -S -o - atom.cpp | sed -n '1,80p'
```

核对是否出现 `lock`、是否出现 `mfence`，并对照所选 `memory_order`。同一源码在 AArch64 上通常看到更多显式栅栏——对照可用于理解「硬件默认序」与「语言序」的分工，而不要求本页平行展开第二套 ISA。
