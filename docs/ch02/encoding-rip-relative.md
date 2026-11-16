# 指令编码与 RIP 相对寻址

x86-64 指令是变长字节序列：可选前缀、操作码、ModR/M、SIB、位移、立即数按需拼接。读 `objdump -d` 时，左边十六进制是真实编码，右边助记符是解码结果。本页给出足以解释常见 `mov`/`lea`/`call`/`jcc` 的轮廓，并专门说明 PIC 下全局量使用的 RIP-relative 形式。

## 字节流轮廓

一条典型指令的字段顺序：

```text
[前缀...]  操作码  [ModR/M]  [SIB]  [位移]  [立即数]
```

前缀包括操作数宽度覆盖、地址宽度覆盖、段覆盖、`LOCK`/`REP`，以及 64 位模式下的 **REX**（`0x40`–`0x4F`）。REX 的四位含义：

| 位 | 名 | 作用 |
| --- | --- | --- |
| W | REX.W | 1 表示 64 位操作数宽度 |
| R | REX.R | 扩展 ModR/M 的 reg 字段 |
| X | REX.X | 扩展 SIB 的 index |
| B | REX.B | 扩展 ModR/M 的 r/m 或 SIB 的 base |

无 REX 时只能编码旧的 8 个寄存器；访问 `%r8`–`%r15` 或强制 64 位宽度时必须出现 REX。`objdump` 字节列里若以 `48` 开头，常表示 `REX.W`。

## ModR/M 与 SIB

ModR/M 一个字节拆为 `mod`（2 bit）、`reg`（3 bit）、`r/m`（3 bit）：

- `reg` 常选寄存器操作数，或作为操作码扩展。
- `mod` 与 `r/m` 一起描述另一个操作数是寄存器还是内存，以及位移宽度（0/8/32 位）。
- 当 `mod ≠ 11` 且 `r/m = 100` 时，下一条字节是 SIB。

SIB 拆为 `scale`（2 bit）、`index`（3 bit）、`base`（3 bit），对应

\[
\mathrm{EA} = \mathrm{base} + \mathrm{index}\times 2^{\mathrm{scale}} + \mathrm{disp}
\]

`index = 4`（`%rsp`）表示无变址。汇编里写出的 `disp(%base,%index,scale)` 与这三个字段一一对应。

## RIP-relative 寻址

64 位模式下，`mod = 00` 且 `r/m = 101` 被定义为 **RIP-relative**：有效地址为

\[
\mathrm{EA} = \mathrm{RIP}_{\mathrm{next}} + \mathrm{disp32}
\]

其中 \(\mathrm{RIP}_{\mathrm{next}}\) 是**本指令之后**第一条字节的地址，`disp32` 为有符号 32 位位移。GAS 写作 `symbol(%rip)`：

```asm
    movq    g_count(%rip), %rax
    leaq    .L.str(%rip), %rdi
```

链接器把 `disp32` 填成「目标地址 − 下一条指令地址」。相对同一映像内的符号，32 位位移通常足够；跨共享库的全局量则经 GOT 间接，见 [位置无关代码与动态链接](pic-dynamic.md)。

RIP-relative 使代码在不修改指令字节的前提下装载到不同虚拟基址（配合相对重定位），是 x86-64 PIC 的默认形态。绝对 64 位立即数搬移（`movabs $imm64, %reg`）仍然存在，但编译器对局部全局量优先选 RIP-relative。

## 从 objdump 字节还原

取一小段可重定位对象上的反汇编（地址与字节随工具链略有差异，读法相同）：

```text
0:  48 8b 05 00 00 00 00    mov    0x0(%rip),%rax
7:  R_X86_64_PC32   g_count-0x4
```

逐步对齐：

1. `48`：REX.W，64 位操作。
2. `8b`：`mov r/m → reg` 操作码。
3. `05`：ModR/M。二进制 `0000 0101` → `mod=00`，`reg=000`（`%rax`），`r/m=101` → RIP-relative。
4. `00 00 00 00`：占位 `disp32`；重定位记录 `R_X86_64_PC32` 指示链接器按 PC 相对规则回填。`-0x4` 出现在符号修饰里，是因为重定位加数相对于位移字段末尾/下一条指令的约定，与「EA = RIP_next + disp」一致。

再看一条近相对调用：

```text
e:  e8 00 00 00 00          call   f
13: R_X86_64_PLT32  f-0x4
```

`e8` 后跟 32 位相对位移，目标为 `RIP_next + disp32`（经 PLT 时进过程链接表）。无重定位的可执行文件里，`disp32` 已是具体非零值，可手算：

\[
\mathrm{target} = \mathrm{addr}(\mathrm{call下一条}) + \mathrm{disp32}
\]

条件跳转 `74 xx`（`je` 短跳）或 `0f 84 disp32`（近跳）同理，只是位移宽度与操作码不同。

## 阅读练习路径

对任意 `.o`：

```bash
clang -c -o foo.o foo.s
objdump -d -r foo.o
```

选择一条带 `(%rip)` 或 `call` 的指令：先数清前缀与操作码，再判断是否有 ModR/M/SIB，最后把位移与 `-r` 重定位行对上。能独立完成这一步，再读 [控制流](control-flow.md) 与 [重定位](relocation.md) 时就不会把助记符当成不可拆的黑盒。
