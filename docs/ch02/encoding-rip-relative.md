# 指令编码与 RIP 相对寻址

x86-64 指令是变长字节序列：可选前缀、操作码、ModR/M、SIB、位移、立即数按需拼接。读 `objdump -d` 时，左边十六进制是真实编码，右边助记符是解码结果。本页给出足以解释常见 `mov`/`lea`/`call`/`jcc` 的轮廓，并专门说明 PIC 下全局量使用的 RIP-relative 形式；目标是能对着字节列独立还原一条简单指令。

## 字节流轮廓

一条典型指令的字段顺序：

```text
[前缀...]  操作码  [ModR/M]  [SIB]  [位移]  [立即数]
```

并非每条指令都具备全部字段。`ret` 可以是单字节 `c3`；`call rel32` 是操作码加四字节位移；寄存器到寄存器的 `movq` 通常含 REX、操作码与 ModR/M；带比例变址的内存操作数再插入 SIB 与可选位移。

前缀包括操作数宽度覆盖（`66`）、地址宽度覆盖（`67`）、段覆盖、`LOCK`（`f0`）、`REP`/`REPE`/`REPNE`，以及 64 位模式下的 **REX**（`0x40`–`0x4F`）。REX 的四位含义：

| 位 | 名 | 作用 |
| --- | --- | --- |
| W | REX.W | 1 表示 64 位操作数宽度 |
| R | REX.R | 扩展 ModR/M 的 `reg` 字段（选 `%r8`–`%r15`） |
| X | REX.X | 扩展 SIB 的 `index` |
| B | REX.B | 扩展 ModR/M 的 `r/m` 或 SIB 的 `base` |

无 REX 时只能编码旧的 8 个通用寄存器；访问 `%r8`–`%r15` 或强制 64 位操作数宽度时必须出现 REX。字节列以 `48` 开头时常为 `REX.W=1` 且 R/X/B 均为 0；`4c` 表示 `REX.W` 与 `REX.R` 同时置位（例如目的寄存器为 `%r8`）。

旧式单字节操作码里，部分指令把寄存器编号编码在操作码低三位（如 `b8+rd` 的 `mov $imm32, %r32`）。64 位立即数装入寄存器使用 `REX.W` + `b8+rd` 的 10 字节形式，反汇编助记符常写作 `movabs`。

## ModR/M 与 SIB

ModR/M 一个字节拆为 `mod`（2 bit）、`reg`（3 bit）、`r/m`（3 bit）：

| `mod` | 含义（`r/m` 为内存时） |
| --- | --- |
| `00` | 无位移（例外：`r/m=101` 为 RIP-relative 或 `disp32` 特例） |
| `01` | 8 位位移 |
| `10` | 32 位位移 |
| `11` | `r/m` 也是寄存器，无内存、无 SIB |

`reg` 常选寄存器操作数，或作为操作码扩展（此时操作码组靠 `reg` 区分 `add`/`or`/`adc` 等）。当 `mod ≠ 11` 且 `r/m = 100` 时，下一字节为 SIB。

SIB 拆为 `scale`（2 bit）、`index`（3 bit）、`base`（3 bit），对应

\[
\mathrm{EA} = \mathrm{base} + \mathrm{index}\times 2^{\mathrm{scale}} + \mathrm{disp}
\]

`index = 4`（编码上对应 `%rsp`）表示无变址。`base = 5` 且 `mod = 00` 时表示无基址、仅有 `disp32`（再经 REX.B 扩展另论）。汇编里写出的 `disp(%base,%index,scale)` 与这三个字段一一对应；`(%rsp)` 之所以总带 SIB，是因为 `r/m=100` 被征用为「后跟 SIB」的逃逸码。

## RIP-relative 寻址

64 位模式下，`mod = 00` 且 `r/m = 101`（且不走 SIB 路径）被定义为 **RIP-relative**：有效地址为

\[
\mathrm{EA} = \mathrm{RIP}_{\mathrm{next}} + \mathrm{disp32}
\]

其中 \(\mathrm{RIP}_{\mathrm{next}}\) 是**本指令之后**第一条字节的地址，`disp32` 为有符号 32 位位移。GAS 写作 `symbol(%rip)`：

```asm
    movq    g_count(%rip), %rax
    leaq    .L.str(%rip), %rdi
    call    foo@PLT
```

链接器把 `disp32` 填成「目标地址 − 下一条指令地址」。同一可执行映像内的符号，32 位位移通常足够；跨共享库的可互操作全局量则经 GOT 间接取址，见 [位置无关代码与动态链接](pic-dynamic.md)。

RIP-relative 使代码在不改写指令中的绝对地址的前提下，可装载到不同虚拟基址（配合相对重定位），成为 x86-64 PIC 的默认形态。绝对 64 位立即数搬移（`movabs $imm64, %reg`）仍然合法，编码更长，且若立即数是运行时基址相关常量，反而需要额外绝对重定位；编译器对模块内全局量与字符串字面量优先选 RIP-relative。

`leaq symbol(%rip), %reg` 只把地址装进寄存器，不访存；`movq symbol(%rip), %reg` 则读取该地址上的八字节。二者操作码与语义都不同，读字节时不要只看 `(%rip)` 就当成同一种指令。

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
4. `00 00 00 00`：占位 `disp32`；重定位记录 `R_X86_64_PC32` 指示链接器按 PC 相对规则回填。符号旁的 `-0x4` 来自「加数相对于位移字段末尾 / 下一条指令」的约定，与 \(\mathrm{EA} = \mathrm{RIP}_{\mathrm{next}} + \mathrm{disp}\) 一致：位移字段长 4 字节，故记录里常出现相对符号减 4 的写法。

再看一条近相对调用：

```text
e:  e8 00 00 00 00          call   f
13: R_X86_64_PLT32  f-0x4
```

`e8` 后跟 32 位相对位移，目标为 \(\mathrm{RIP}_{\mathrm{next}} + \mathrm{disp32}\)（经 PLT 时先进入过程链接表桩）。无重定位的可执行文件里，`disp32` 已是具体非零值，可手算：

\[
\mathrm{target} = \mathrm{addr}(\text{call 的下一条}) + \mathrm{disp32}
\]

再拆一条寄存器间传送作为对照：

```text
20: 48 89 f8                mov    %rdi,%rax
```

1. `48`：REX.W。
2. `89`：`mov reg → r/m`。
3. `f8`：`mod=11`（寄存器）, `reg=111`（`%rdi`）, `r/m=000`（`%rax`）。无 SIB、无位移、无立即数，整条三字节。

条件跳转：`74 xx` 为 `je` 短跳（8 位位移）；`0f 84 disp32` 为 `je` 近跳。位移同样相对下一条指令。循环与向前跳在未链接的 `.o` 里可能仍是占位零，链接或汇编器定址后才变成真实偏移。

## 长度变化与阅读顺序

同助记符因操作数形态不同而长度悬殊：`movq %rax, %rbx` 三字节级；`movq %rax, -0x20(%rbp)` 带 `disp8`；`movq $0x12345678, g(%rip)` 含立即数与 RIP 位移。阅读顺序固定为：

1. 数清前缀（含 REX），确定操作数宽度与高号寄存器。
2. 取操作码（一字节或 `0f` 逃逸的多字节）。
3. 若格式需要，解 ModR/M，判断是否 SIB、位移宽度、是否 RIP-relative。
4. 读取位移与立即数（小端）。
5. 若 `objdump -r` 在该指令下挂了重定位行，用重定位类型解释为何位移现为 0、链接后应变为什么。

## 最小自检命令

```bash
clang -c -o foo.o foo.s
objdump -d -r foo.o
```

在输出中任选一条带 `(%rip)` 的 `mov`/`lea` 与一条 `call`：写出各字节字段，并核对重定位行的类型与加数。能独立完成这一步，再读 [控制流](control-flow.md) 与 [重定位](relocation.md) 时，助记符就是字节结构的摘要，而不是不可拆的黑盒。
