# 寻址方式与数据传送

一条 x86-64 指令的操作数来自三类来源：立即数、寄存器、内存。内存操作数由基址、变址、比例与位移组成；硬件在译码阶段把这些字段收成有效地址。本页给出 AT&T 写法与对齐约束，为下一页的 ModR/M / SIB / RIP-relative 编码提供语义锚点。

## 立即数、寄存器与内存

**立即数**写在指令里，带 `$` 前缀：`$0`、`$0x20`、`$label`（由汇编器/链接器填成常量）。宽度受助记符与编码格式限制；超范围的立即数需拆成多条指令或经内存中转。

**寄存器操作数**写作 `%reg`，宽度与助记符一致：

```asm
    movq    %rsi, %rax
    movl    $1, %eax
    addq    %rdx, %rax
```

**内存操作数**的通式为：

```text
disp(%base, %index, scale)
```

有效地址为

\[
\mathrm{EA} = \mathrm{base} + \mathrm{index}\times\mathrm{scale} + \mathrm{disp}
\]

其中 `scale` \(\in \{1,2,4,8\}\)，各项均可缺省。AT&T 示例：

```asm
    movq    (%rdi), %rax           # *rdi
    movq    8(%rbp), %rax          # *(rbp+8)
    movq    (%rdi,%rsi,8), %rax    # *(rdi + rsi*8)
    movq    -16(%rbp,%rcx,4), %rax
    leaq    buf(%rip), %rdi        # RIP-relative，见下页
```

`lea`（load effective address）计算 EA 写入寄存器，不访问内存、不改标志，常用于乘法式地址运算与取标签地址。`mov` 在操作数是内存时才发生访存。

## SIB 与缺省形式

当有效地址需要 index 或 scale，或 base 为 `%rsp`/`%r12` 等特殊编码时，硬件使用 SIB 字节（Scale-Index-Base）。对汇编书写者，SIB 表现为「能写出的寻址形态」；对读 `objdump` 字节者，SIB 是 ModR/M 之后的那一个字节。规则压缩如下：

- 仅有 `(%rax)` 一类「基址、无变址」时，通常只有 ModR/M，无 SIB。
- `(%rsp)`、`disp(%rsp)` 必须带 SIB（硬件用 index=`%rsp` 的编码表示「无变址」）。
- `(%rip)` 相对形式不走「base=RIP」的通用寄存器编码，而走 ModR/M 的特殊 RIP-relative 槽位，位移为相对**下一条指令**的有符号偏移。

因此 `%rsp` 作基址、RIP-relative 取全局量、以及 `(%rax,%rbx,4)` 在反汇编字节列上的长度不同。 systematically 读字节的方法见 [指令编码与 RIP 相对寻址](encoding-rip-relative.md)。

## 数据传送与符号扩展

常用传送类指令：

| 指令 | 作用 |
| --- | --- |
| `mov` | 按宽度拷贝；`movl` 到 32 位寄存器时清零高 32 位 |
| `movzx` / `movzbl` 等 | 零扩展 |
| `movsx` / `movsbl` 等 | 符号扩展 |
| `push` / `pop` | 栈上传送并调整 `%rsp` |
| `xchg` | 交换；与内存时隐含锁定总线（历史语义） |
| `cmovcc` | 按条件码条件传送，不分支 |

从内存读入窄值时，选择零扩展还是符号扩展决定高位内容。与 ABI「参数在寄存器中已是扩展后视图」一致时，后续 `cmp`/`test` 的高位行为才与 C 语义对齐。

块拷贝与串操作（`movsq`、`rep movsb` 等）隐式使用 `%rdi`/`%rsi`/`%rcx`；手写或阅读时注意这些寄存器的副作用。

## 对齐

对齐指有效地址对类型自然对齐边界取模为零：`int64_t` 访问地址满足 \(\mathrm{EA} \equiv 0 \pmod{8}\)，16 字节 SSE 访问满足 \(\mathrm{EA} \equiv 0 \pmod{16}\)。x86-64 对多数普通整数/浮点标量访存允许未对齐，代价是可能的拆分访存与性能损失；部分专用指令（如旧的对齐要求严格的 SSE 移动）在未对齐时触发故障。ABI 要求栈在调用边界 16 字节对齐，向量参数与某些结构体字段亦有对齐约束——这是软件契约，靠编译器与手写汇编共同遵守，见 [SysV ABI](sysv-amd64-abi.md)。

编译器为全局量与栈槽插入的对齐填充，可在 `.align` / `.p2align`（见 [段与指令](sections-directives.md)）与 prologue 的 `subq $N, %rsp` 中观察到。读反汇编时，若看到对 `%rsp` 的额外减法使总调整量满足 \(16\) 的倍数，通常是在满足调用边界对齐。

## 与高级语言表达式的对应

把 C 表达式落到寻址形态，便于对照 `clang -S` 输出：

```c
long f(long *a, long i) { return a[i]; }
```

典型汇编：

```asm
    movq    (%rdi,%rsi,8), %rax
    ret
```

结构体字段访问变成「基址 + 常量位移」；取址 `&x` 常变成 `leaq`；全局量在 `-fPIC` 下变成 `名(%rip)`。控制流与调用不改变寻址算术本身，只改变 `%rip`/`%rsp` 与寄存器存活集；下一页进入编码层，说明这些形态如何变成固定字节序列。
