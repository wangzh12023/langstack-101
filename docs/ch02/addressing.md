# 寻址方式与数据传送

一条 x86-64 指令的操作数来自三类来源：立即数、寄存器、内存。内存操作数由基址、变址、比例与位移组成；硬件在译码阶段把这些字段收成有效地址。本页给出 AT&T 写法、SIB 相关形态与对齐约束，为 [指令编码与 RIP 相对寻址](encoding-rip-relative.md) 提供语义锚点，并说明常见数据传送指令如何落在这些形态上。

## 立即数、寄存器与内存

**立即数**写在指令里，带 `$` 前缀：`$0`、`$0x20`、`$label`（由汇编器或链接器填成常量或重定位加数）。宽度受助记符与编码格式限制：许多算术指令允许符号扩展的 8 位立即数以缩短编码；需要完整 64 位常量时，常用 `movabs` 先装入寄存器再运算。

**寄存器操作数**写作 `%reg`，宽度与助记符一致：

```asm
    movq    %rsi, %rax
    movl    $1, %eax          # 写入 eax，同时清零 rax 高 32 位
    addq    %rdx, %rax
    xorl    %eax, %eax        # 清零 rax 的惯用短编码
```

**内存操作数**的通式为：

```text
disp(%base, %index, scale)
```

有效地址为

\[
\mathrm{EA} = \mathrm{base} + \mathrm{index}\times\mathrm{scale} + \mathrm{disp}
\]

其中 `scale` \(\in \{1,2,4,8\}\)，各项均可缺省。缺省规则在写法上的对应关系：

| 写法 | 含义 |
| --- | --- |
| `(%rdi)` | 基址，无位移 |
| `8(%rbp)` | 基址 + 位移 |
| `(%rdi,%rsi)` | 基址 + 变址（scale=1） |
| `(%rdi,%rsi,8)` | 基址 + 变址×8 |
| `label(%rip)` | RIP-relative，见编码页 |
| `0x400000` | 绝对位移（64 位模式下少用作通用访存） |

```asm
    movq    (%rdi), %rax
    movq    8(%rbp), %rax
    movq    (%rdi,%rsi,8), %rax
    movq    -16(%rbp,%rcx,4), %rax
    leaq    buf(%rip), %rdi
```

`lea`（load effective address）计算 EA 写入寄存器，不访问内存、不修改标志，常用于「乘加式」地址运算、取标签地址、以及把 `base+index*scale+disp` 算进指针。`mov` 在操作数是内存时才发生访存。同一表达式写成 `lea` 还是 `mov`，决定是否解引用。

## SIB 与缺省形式

当有效地址需要 index 或 scale，或 base 为 `%rsp`/`%r12` 等特殊编码时，硬件使用 SIB 字节（Scale-Index-Base）。对汇编书写者，SIB 表现为「能写出的寻址形态」；对读 `objdump` 字节者，SIB 是 ModR/M 之后的那一个字节。压缩规则：

- 仅有 `(%rax)` 一类「基址、无变址、非 rsp/r12」时，通常只有 ModR/M，无 SIB。
- `(%rsp)`、`disp(%rsp)` 必须带 SIB：ModR/M 的 `r/m=100` 表示「后跟 SIB」，SIB 里再用 `index=4` 表示无变址。
- `%r12` 作基址时同样走 SIB 路径（与 `%rsp` 共享需逃逸的编码槽）。
- `(%rip)` 相对形式不走「base=RIP」的通用寄存器编码，而走 ModR/M 的特殊 RIP-relative 槽位，位移相对**下一条指令**。

因此 `%rsp` 作基址、RIP-relative 取全局量、以及 `(%rax,%rbx,4)` 在反汇编字节列上的长度不同。按字段读字节的方法见 [指令编码与 RIP 相对寻址](encoding-rip-relative.md)。

规模因子与元素大小对齐时，数组下标降低很直接：`long` 数组用 `scale=8`，`int` 用 `4`。结构体数组则是「下标×元素大小 + 字段位移」，字段位移进 `disp`，下标进 `index*scale`。

## 数据传送与符号扩展

常用传送类指令：

| 指令 | 作用 |
| --- | --- |
| `mov` | 按宽度拷贝；`movl` 到 32 位寄存器时清零高 32 位 |
| `movzx` / `movzbl` 等 | 零扩展窄源到宽目的 |
| `movsx` / `movsbl` / `movslq` 等 | 符号扩展 |
| `push` / `pop` | 栈上传送并调整 `%rsp` |
| `xchg` | 交换；与内存操作数组合时具有隐含总线锁语义 |
| `cmovcc` | 按条件码条件传送，不分支 |
| `lea` | 只计算地址 |

从内存读入窄值时，零扩展与符号扩展决定高位内容。C 的整型提升与 ABI「参数在寄存器中已是扩展后视图」一致时，后续 `cmp`/`test` 才与源语言语义对齐。编译器输出里常见 `movzbl (%rdi), %eax` 再参与运算，而不是先 `movb` 到 `%al` 再依赖高位残留。

块拷贝与串操作（`movsq`、`rep movsb`、`stos` 等）隐式使用 `%rdi`（目的）、`%rsi`（源）、`%rcx`（计数），方向位 `DF` 控制地址增减。手写或阅读时必须把这些副作用算进活跃寄存器集合；它们与 SysV 参数寄存器重叠，调用约定下跨调用本就会失效，函数内部则需避免与仍在使用的指针冲突。

## 对齐

对齐指有效地址对类型自然对齐边界取模为零：8 字节访问满足 \(\mathrm{EA} \equiv 0 \pmod{8}\)，16 字节 SSE 访问满足 \(\mathrm{EA} \equiv 0 \pmod{16}\)。x86-64 对多数普通整数与标量浮点访存允许未对齐，代价是可能拆成多次总线事务并损失性能；部分要求对齐的 SIMD 加载/存储在未对齐时触发故障，较新的指令则区分对齐与未对齐助记符。

ABI 要求栈在**调用边界** 16 字节对齐；向量参数、某些结构体字段与通过栈传递的实参槽亦有对齐约束——这是软件契约，靠编译器与手写汇编共同遵守，见 [SysV ABI](sysv-amd64-abi.md)。全局量与栈槽的对齐填充，可在 `.p2align`（见 [段与指令](sections-directives.md)）与 prologue 的 `subq $N, %rsp` 中观察到：若 \(N\) 的选择使下一次 `call` 前 `%rsp` 回到 16 的倍数，通常就是在满足调用对齐。

## 与高级语言表达式的对应

把 C 表达式落到寻址形态，便于对照 `clang -S` 输出。

```c
long index(long *a, long i) { return a[i]; }
```

典型汇编：

```asm
    movq    (%rdi,%rsi,8), %rax
    ret
```

```c
struct Point { long x, y; };
long get_y(struct Point *p) { return p->y; }
```

```asm
    movq    8(%rdi), %rax      # y 在偏移 8
    ret
```

```c
long *addr_of_slot(long *base, long i) { return &base[i]; }
```

```asm
    leaq    (%rdi,%rsi,8), %rax
    ret
```

全局量在默认 PIC 模型下变成 `名(%rip)`；取址用 `leaq`，读值用 `mov`。控制流与调用不改变寻址算术本身，只改变 `%rip`/`%rsp` 与寄存器存活集。

## 读写宽度与部分更新

内存操作数的宽度由助记符决定：`movb` 只改一字节，`movl` 改四字节，`movq` 改八字节。对同一地址先 `movb` 再按 `movq` 读出，高位字节来自内存中未写入的旧内容。结构体逐字段赋值在汇编里常表现为一串不同位移、不同宽度的 `mov`；读反汇编时先还原「基址 + 位移」对应的字段，再看宽度是否与类型一致。

只更新 32 位寄存器会清零高 32 位，只更新内存中的 32 位则不影响相邻字节——寄存器与内存的「部分写」规则不同。这会影响别名分析与编译器选择：对内存中的 `int` 成员写，通常生成 `movl`；对该值装入寄存器参与 64 位运算前，再按符号性扩展。

下一页进入编码层，说明这些形态如何变成固定字节序列；再往后的 [控制流](control-flow.md) 与 [栈帧](stack-frames.md) 则把 `%rip`/`%rsp` 的变化接到过程调用。
