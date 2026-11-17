# 指令级程序


要把「进程里正在跑的东西」说清楚，得先能读写 CPU 真正执行的那种程序：它待在哪些段里、用哪些寄存器、地址怎么算、分支怎么写、机器字里的比特如何对应助记符。这一站停在「单线程、用户态、还没谈函数约定」之前；下一站才把 `call` 变成有 ABI 的故事。


汇编源文件同时描述两类事物：指令与数据的**内容**，以及它们在目标文件里落入哪一**段**、对外可见性如何、对齐约束是什么。汇编器把段指令翻译成 ELF 节区（section）与符号表项；链接器再按节区属性合并、重定位并映射到可装载段（segment）。本页处理静态汇编里最常用的一组指令与数据伪指令；ELF 头与程序头的完整字段见 [ELF 文件结构](elf.md)。

### 三类内容段

可重定位目标文件里，与用户代码直接对应的内容通常落在三类节区。分类轴是「装载后是否占文件空间、是否可写」。

`.text` 存放机器指令，装载后一般映射为可读可执行、不可写。函数体、跳转目标标签默认写在这里：

```asm
    .text
    .globl  add1
    .type   add1, @function
add1:
    leaq    1(%rdi), %rax
    ret
```

`.data` 存放已初始化的可写数据。全局/静态变量的初值以字节形式进入目标文件，装载时从文件拷贝到可写页：

```asm
    .data
    .p2align 3
    .globl  g_count
    .type   g_count, @object
g_count:
    .quad   42
```

`.bss` 存放未初始化（或显式为零）的可写数据。节区在文件中不占对应的数据字节，只记录需要的尺寸；装载器在进程里分配并清零。GAS 用 `.comm` / `.lcomm` 或切换到 `.bss` 后用 `.zero` / `.skip` 声明：

```asm
    .bss
    .p2align 3
buf:
    .zero   4096

    .comm   shared_buf, 4096, 8    # 通用符号，对齐 8
    .lcomm  local_buf, 256         # 局部 bss
```

只读常量常落在 `.rodata`（字符串字面量、`const` 全局量、跳转表）。它与 `.text` 一样不可写，但通常不可执行。读写与可执行属性由节区标志决定，检视时以 `readelf -S` 的 `W`/`A`/`X` 为准。

GAS 也允许 `.section name, "flags", @type` 显式建节，例如 `.section .rodata.str1.1,"aMS",@progbits,1` 存放合并字符串；编译器输出里常见，手写实验多数用简写伪指令即可。

### 数据伪指令

在 `.data` / `.rodata` 中写入具体字节的常用伪指令：

| 伪指令 | 宽度 / 含义 |
| --- | --- |
| `.byte` | 8 位 |
| `.word` / `.short` | 16 位 |
| `.long` / `.int` | 32 位 |
| `.quad` | 64 位 |
| `.ascii "hi"` | 原始字节，无尾零 |
| `.asciz "hi"` / `.string` | 带尾零的 C 字符串 |
| `.zero n` / `.skip n` | n 字节零填充 |
| `.rept` / `.endr` | 重复块 |

```asm
    .section .rodata
msg:
    .asciz  "hello\n"
nums:
    .quad   1, 2, 3, 4
```

多标签可指向同一段内容的不同偏移，用于结构体字段式布局或跳转表项。位置计数器 `.` 在每条伪指令后前进相应字节数。

### 符号与可见性

标签在汇编里是地址的名字。汇编器为每个标签生成符号表项；链接器按绑定与类型解析跨文件引用。

`.globl name`（GAS 亦接受 `.global`）把 `name` 标为全局绑定：本目标文件定义、其他目标文件可引用。缺少 `.globl` 的标签默认局部（STB_LOCAL），链接阶段对外不可见，可与其他文件中的同名局部符号共存。

`.type name, @function` / `@object` 写入符号类型，供链接器、调试器与某些分析工具区分函数与数据。`.size name, .-name` 用「当前位置减去符号起点」写出符号大小，常见于函数或对象末尾；`nm` / `readelf -s` 的 size 列由此而来。

`.weak name` 声明弱符号：若链接时存在强定义则用强定义，否则保留弱定义或把未解析弱引用置零。动态库与弱引用的解析规则放到 [静态链接](static-linking.md) 与 [PIC](pic-dynamic.md)。

`.extern name` 在 GAS/ELF 目标上通常可省略：对未定义标签的引用会自动产生未定义符号。显式写出有时用于可读性。

### 对齐与填充

对齐约束写在定义之前，作用于随后的位置计数器。`.p2align n` 把地址对齐到 \(2^n\) 字节，语义在跨平台时最明确。`.balign m` 对齐到 \(m\) 字节（参数为字节数）。`.align` 在 x86-64 ELF 的 GAS 上通常按幂次解释，但与部分平台语义不同，可移植宏优先 `.p2align` / `.balign`。

```asm
    .data
    .p2align 3          # 8 字节对齐
vec:
    .quad   1, 2, 3, 4

    .text
    .p2align 4          # 16 字节对齐函数入口，利于取指
    .globl  hotpath
hotpath:
    ...
```

对齐不足时，硬件可能对某些访存产生对齐故障，或在允许未对齐的路径上拆分事务、损失性能。ABI 对栈、向量参数、某些结构体字段另有对齐要求，见 [SysV ABI](sysv-amd64-abi.md)。汇编器在对齐伪指令与下一条内容之间插入填充：代码段常用 `nop` 图案，数据段用零。

### 位置计数器与局部标签

`.` 表示当前位置计数器。`.-foo` 计算从标签 `foo` 到当前位置的距离，常用于 `.size` 与手工距离计算。

GAS 数字局部标签 `1:`、`2:` 可重复出现；引用写成 `1b`（backward）或 `1f`（forward），避免为每个基本块发明全局唯一名：

```asm
    cmpq    %rsi, %rdi
    jle     1f
    movq    %rdi, %rax
    ret
1:
    movq    %rsi, %rax
    ret
```

编译器生成的标签常呈 `.L` 前缀（本地、不进符号表对外名）；手写时亦可遵循同一风格，减少与全局符号冲突。

### 与 ELF 节区的对应

汇编里的「段」在目标文件中首先是**节区**。典型映射：

| 汇编方向 | ELF 节区（常见） | 装载后权限（常见） |
| --- | --- | --- |
| `.text` | `.text` | `r-x` |
| `.data` | `.data` | `rw-` |
| `.bss` | `.bss` | `rw-`（文件尺寸为 0） |
| 只读常量 | `.rodata` | `r--` |
| 符号/重定位 | `.symtab`、`.rela.text` 等 | 链接期使用，通常不进运行时 `PT_LOAD` |

链接器按节区标志合并多个 `.o` 的同名节，再把若干节区打包进程序头里的 `PT_LOAD` 段。口语里的「段」常混用两层意思：汇编器指令说的是逻辑内容桶，装载器看的是虚拟地址连续的 `PT_LOAD`。用 `readelf -S` 看节，用 `readelf -l` 看装载段，两者一起才能对上进程映像。

对 `.text` 里的外部符号引用，汇编器通常留下重定位记录（例如 `R_X86_64_PLT32`、`R_X86_64_PC32`、`R_X86_64_REX_GOTP`），把最终地址或 GOT 槽交给链接器填写。[重定位](relocation.md) 说明这些记录如何改写指令中的位移字段；本页只需建立「符号名 → 节区内偏移 → 待填字节」的链条。

### 最小可汇编单元

把符号、段与对齐放在同一文件里，得到链接器可消费的最小形状：

```asm
    .text
    .globl  _start
    .type   _start, @function
_start:
    movl    $60, %eax       # SYS_exit（Linux x86-64）
    xorl    %edi, %edi      # status = 0
    syscall
    .size   _start, .-_start
```

该例直接以 `_start` 为入口并使用 `syscall`，绕过 C 运行时；正常与 C 混链时入口与参数布置见 [进程入口](crt-entry.md)。组装与检视：

```bash
clang -c -o start.o start.s
objdump -d start.o
readelf -S start.o
```

下一步是弄清指令实际读写哪些寄存器：[寄存器与标志位](registers.md)。


段解决「字节属于哪一类」；运算本身发生在寄存器与条件码上。


x86-64 整数状态对汇编读者可见的部分，主要是十六个通用寄存器、指令指针 `RIP`、栈指针 `RSP`，以及标志寄存器 `RFLAGS` 中与条件码相关的若干位。向量与浮点状态在 `XMM`/`YMM`/`ZMM` 与 MXCSR 中；调用约定如何使用它们见 [SysV ABI](sysv-amd64-abi.md)。本页建立读写这些名字时的默认语义，并预告调用者/被调用者保存划分，为后续寻址、编码与栈帧页提供共同词汇。

### 通用寄存器

十六个 64 位通用寄存器：`RAX`、`RBX`、`RCX`、`RDX`、`RSI`、`RDI`、`RBP`、`RSP`、`R8`–`R15`。每个都可按宽度取子视图；**写 32 位子寄存器时，硬件将对应 64 位寄存器的高 32 位清零**。写 16 位或 8 位子寄存器则保留高位不变。

| 64 位 | 32 位 | 16 位 | 8 位（低） | 8 位（高，历史） |
| --- | --- | --- | --- | --- |
| `%rax` | `%eax` | `%ax` | `%al` | `%ah` |
| `%rbx` | `%ebx` | `%bx` | `%bl` | `%bh` |
| `%rcx` | `%ecx` | `%cx` | `%cl` | `%ch` |
| `%rdx` | `%edx` | `%dx` | `%dl` | `%dh` |
| `%rsi` | `%esi` | `%si` | `%sil` | — |
| `%rdi` | `%edi` | `%di` | `%dil` | — |
| `%rbp` | `%ebp` | `%bp` | `%bpl` | — |
| `%rsp` | `%esp` | `%sp` | `%spl` | — |
| `%r8` | `%r8d` | `%r8w` | `%r8b` | — |
| … | … | … | … | — |
| `%r15` | `%r15d` | `%r15w` | `%r15b` | — |

`%ah` 等高速字节与 REX 前缀不能随意组合，现代编译器输出很少使用它们；阅读旧代码或手工优化片段时仍可能遇到。

历史别名仍出现在机器码与文档里：`%rax` 常作返回值与某些指令的隐式操作数；`%rdx:%rax` 组成除法与宽乘法的 128 位对；`%rcx` 作串指令计数与移位计数；`%rdi`/`%rsi` 作串指令的目的/源指针。这些隐式用法与 ABI 参数分工重叠，读 `objdump` 时两者要一起看：一条 `div` 会改写 `%rax`/`%rdx`，即使源码变量并不叫这个名字。

在 System V AMD64 用户态约定下，整数/指针参数按顺序使用 `%rdi`、`%rsi`、`%rdx`、`%rcx`、`%r8`、`%r9`；返回值主要在 `%rax`（次宽整数可进 `%rdx`）。这是约定层，不是 ISA 强制；换调用约定后同一条 `mov` 的硬件语义不变，改变的是「谁在调用边界上仍然有效」。

### RIP 与 RSP

`RIP` 是下一条将执行指令的地址。x86-64 用户态不能像通用寄存器那样用 `movq %rip, %rax` 直接读写；读取当前 PC 的常见办法是 `leaq label(%rip), %reg`，或依赖 `call` 压栈的返回地址。控制转移指令（`jmp`/`call`/`ret`/`jcc`）通过改写 `RIP` 改变执行位置；编码里相对位移的基准是**下一条指令的地址**，见 [指令编码与 RIP 相对寻址](encoding-rip-relative.md)。

`RSP` 指向栈顶。`push` 先把 `RSP` 减去操作数宽度再写入；`pop` 先读再加回。`call` 等价于把返回地址压栈并跳转；`ret` 弹出到 `RIP`。栈向低地址增长：

```text
高地址
  ...
  调用方帧
  返回地址      ← call 之后，(%rsp) 最初指向这里
  被保存的寄存器 / 局部量
  ...
低地址          ← %rsp
```

叶子函数在一定条件下可使用 `%rsp` 下方的红区而不显式减法，规则见 [栈帧](stack-frames.md) 与 [SysV ABI](sysv-amd64-abi.md)。

`RBP` 在未优化代码里常作帧指针：prologue 保存旧 `%rbp` 并把 `%rbp` 设为当前帧基址，局部变量用负偏移、经栈传入的参数用正偏移寻址。`-fomit-frame-pointer`（多为 `-O` 默认）省略帧指针，改用 `%rsp` 相对寻址；此时「局部变量在哪」随 `%rsp` 的实时值变化，调试信息靠 DWARF 位置表达式描述。

### RFLAGS 中的条件码

算术与逻辑指令更新 `RFLAGS` 中的条件码；条件跳转、条件传送与 `setcc` 读取这些位。与整数比较直接相关的四位：

| 标志 | 含义（典型） |
| --- | --- |
| `CF` | 无符号进位/借位 |
| `ZF` | 结果为零 |
| `SF` | 结果最高位为 1 |
| `OF` | 有符号溢出 |

另有 `PF`（低 8 位奇偶）、`AF`（辅助进位）等，高级语言整数控制流很少直接依赖。`DF`（方向标志）影响串操作地址增减，`cld`/`std` 可改写；ABI 要求在调用边界 `DF` 清零，编译器生成的函数可假设入口 `DF=0`。

`cmp a, b`（AT&T：`cmp src, dst`）按 \(dst - src\) 更新标志但不写回差。`test a, b` 按按位与更新标志，常用于测位或与零比较。条件跳转助记符编码对这些标志的谓词，例如 `je`/`jz` 看 `ZF`，`jl` 看有符号小于，`jb` 看无符号低于；完整表见 [控制流与条件码](control-flow.md)。

`lea` 只做地址计算，**不**改标志。`mov` 通常也不改标志。需要在一串算术中间插入「不算标志的地址运算」时，`lea` 与 `add` 不可互换。

### 调用者保存与被调用者保存

跨 `call` 边界，寄存器分成两组（System V AMD64）：

**调用者保存**（caller-saved，又称 volatile）：`%rax`、`%rcx`、`%rdx`、`%rsi`、`%rdi`、`%r8`–`%r11`，以及 `%xmm0`–`%xmm15` 等。被调用方可任意破坏；调用方若在 `call` 之后还需要其中的值，必须自己先存到栈或 callee-saved 寄存器。

**被调用者保存**（callee-saved，又称 non-volatile）：`%rbx`、`%rbp`、`%r12`–`%r15`。被调用方若使用，必须在返回前恢复原值；调用方可以假定它们跨调用存活。

分组降低总溢写：叶子函数多用 caller-saved 可少做 prologue/epilogue；深调用链上的长生命周期值放在 callee-saved，避免每个调用点都保存。完整表与浮点、变参规则写在 [SysV ABI](sysv-amd64-abi.md)；[栈帧](stack-frames.md) 演示 prologue 如何 `push` callee-saved。

### 宽度后缀与零扩展

AT&T 助记符用 `b`/`w`/`l`/`q` 标宽度：`movb`、`movw`、`movl`、`movq`。写 `%eax` 的 `movl` 会清零 `%rax` 的高 32 位；写 `%al` 的 `movb` 只改低 8 位。从窄到宽需要显式扩展：`movzbl`（零扩展到 32 位，并因写 `eax` 而清零高半）、`movsbq`（符号扩展到 64 位）等。

C 里「窄参数在寄存器中如何呈现」由 ABI 规定为已扩展到约定宽度的视图；手写汇编与编译器输出保持同一约定，可避免高位垃圾进入比较或返回值。观测手段：对一小段 C 函数使用 `clang -S -O2`，看入口后是否出现 `movsxd`/`movzx`，再对照 [寻址方式与数据传送](addressing.md) 中的内存操作数写法。

### XMM 寄存器预告

`%xmm0`–`%xmm15` 各 128 位，可存放标量 `float`/`double` 或打包向量；在 AVX 下扩展为 `%ymm`，在 AVX-512 下扩展为 `%zmm`。System V 用 `%xmm0`–`%xmm7` 传递前八个标量浮点实参，返回值主要在 `%xmm0`。整数通用寄存器与 XMM 在调用约定里是两套并行资源：同一调用可以同时占用 `%rdi` 与 `%xmm0` 而不互相替换。MXCSR 控制舍入与浮点异常掩码，普通整数汇编很少改写它；语言运行时与数值库才会显式触及。完整传参表见 [SysV ABI](sysv-amd64-abi.md)；本页只需建立「看见 `movsd`/`movss` 就切换到 XMM 命名空间」的阅读习惯。


多数有趣的程序不是纯寄存器：要按一定方式从内存取数。


一条 x86-64 指令的操作数来自三类来源：立即数、寄存器、内存。内存操作数由基址、变址、比例与位移组成；硬件在译码阶段把这些字段收成有效地址。本页给出 AT&T 写法、SIB 相关形态与对齐约束，为 [指令编码与 RIP 相对寻址](encoding-rip-relative.md) 提供语义锚点，并说明常见数据传送指令如何落在这些形态上。

### 立即数、寄存器与内存

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

### SIB 与缺省形式

当有效地址需要 index 或 scale，或 base 为 `%rsp`/`%r12` 等特殊编码时，硬件使用 SIB 字节（Scale-Index-Base）。对汇编书写者，SIB 表现为「能写出的寻址形态」；对读 `objdump` 字节者，SIB 是 ModR/M 之后的那一个字节。压缩规则：

- 仅有 `(%rax)` 一类「基址、无变址、非 rsp/r12」时，通常只有 ModR/M，无 SIB。
- `(%rsp)`、`disp(%rsp)` 必须带 SIB：ModR/M 的 `r/m=100` 表示「后跟 SIB」，SIB 里再用 `index=4` 表示无变址。
- `%r12` 作基址时同样走 SIB 路径（与 `%rsp` 共享需逃逸的编码槽）。
- `(%rip)` 相对形式不走「base=RIP」的通用寄存器编码，而走 ModR/M 的特殊 RIP-relative 槽位，位移相对**下一条指令**。

因此 `%rsp` 作基址、RIP-relative 取全局量、以及 `(%rax,%rbx,4)` 在反汇编字节列上的长度不同。按字段读字节的方法见 [指令编码与 RIP 相对寻址](encoding-rip-relative.md)。

规模因子与元素大小对齐时，数组下标降低很直接：`long` 数组用 `scale=8`，`int` 用 `4`。结构体数组则是「下标×元素大小 + 字段位移」，字段位移进 `disp`，下标进 `index*scale`。

### 数据传送与符号扩展

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

### 对齐

对齐指有效地址对类型自然对齐边界取模为零：8 字节访问满足 \(\mathrm{EA} \equiv 0 \pmod{8}\)，16 字节 SSE 访问满足 \(\mathrm{EA} \equiv 0 \pmod{16}\)。x86-64 对多数普通整数与标量浮点访存允许未对齐，代价是可能拆成多次总线事务并损失性能；部分要求对齐的 SIMD 加载/存储在未对齐时触发故障，较新的指令则区分对齐与未对齐助记符。

ABI 要求栈在**调用边界** 16 字节对齐；向量参数、某些结构体字段与通过栈传递的实参槽亦有对齐约束——这是软件契约，靠编译器与手写汇编共同遵守，见 [SysV ABI](sysv-amd64-abi.md)。全局量与栈槽的对齐填充，可在 `.p2align`（见 [段与指令](sections-directives.md)）与 prologue 的 `subq $N, %rsp` 中观察到：若 \(N\) 的选择使下一次 `call` 前 `%rsp` 回到 16 的倍数，通常就是在满足调用对齐。

### 与高级语言表达式的对应

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

### 读写宽度与部分更新

内存操作数的宽度由助记符决定：`movb` 只改一字节，`movl` 改四字节，`movq` 改八字节。对同一地址先 `movb` 再按 `movq` 读出，高位字节来自内存中未写入的旧内容。结构体逐字段赋值在汇编里常表现为一串不同位移、不同宽度的 `mov`；读反汇编时先还原「基址 + 位移」对应的字段，再看宽度是否与类型一致。

只更新 32 位寄存器会清零高 32 位，只更新内存中的 32 位则不影响相邻字节——寄存器与内存的「部分写」规则不同。这会影响别名分析与编译器选择：对内存中的 `int` 成员写，通常生成 `movl`；对该值装入寄存器参与 64 位运算前，再按符号性扩展。

下一页进入编码层，说明这些形态如何变成固定字节序列；再往后的 [控制流](control-flow.md) 与 [栈帧](stack-frames.md) 则把 `%rip`/`%rsp` 的变化接到过程调用。


助记符是给人看的；链接与 PIC 要动的是编码出来的位移与 ModR/M。


x86-64 指令是变长字节序列：可选前缀、操作码、ModR/M、SIB、位移、立即数按需拼接。读 `objdump -d` 时，左边十六进制是真实编码，右边助记符是解码结果。本页给出足以解释常见 `mov`/`lea`/`call`/`jcc` 的轮廓，并专门说明 PIC 下全局量使用的 RIP-relative 形式；目标是能对着字节列独立还原一条简单指令。

### 字节流轮廓

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

### ModR/M 与 SIB

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

### RIP-relative 寻址

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

### 从 objdump 字节还原

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

### 长度变化与阅读顺序

同助记符因操作数形态不同而长度悬殊：`movq %rax, %rbx` 三字节级；`movq %rax, -0x20(%rbp)` 带 `disp8`；`movq $0x12345678, g(%rip)` 含立即数与 RIP 位移。阅读顺序固定为：

1. 数清前缀（含 REX），确定操作数宽度与高号寄存器。
2. 取操作码（一字节或 `0f` 逃逸的多字节）。
3. 若格式需要，解 ModR/M，判断是否 SIB、位移宽度、是否 RIP-relative。
4. 读取位移与立即数（小端）。
5. 若 `objdump -r` 在该指令下挂了重定位行，用重定位类型解释为何位移现为 0、链接后应变为什么。

### 常见操作码速查

阅读字节列时可先靠操作码定位指令大类，再解 ModR/M：

| 字节 | 常见含义 |
| --- | --- |
| `e8` | `call rel32` |
| `e9` | `jmp rel32` |
| `eb` | `jmp rel8` |
| `74` / `75` | `je` / `jne` 短跳 |
| `0f 84` / `0f 85` | `je` / `jne` 近跳 |
| `c3` | `ret` |
| `89` / `8b` | `mov` 存 / 取（方向相反） |
| `8d` | `lea` |
| `68` | `push imm32` |
| `50`–`57` | `push r64`（短编码，无 REX 时对应旧八寄存器） |

带 REX 时同一助记符的首字节变为 `40`–`4f` 再跟操作码。上表不是完整操作码图，只用于在 `objdump` 里快速锚定；遇多字节逃逸（`0f`、`0f 38`、`0f 3a`）需查手册或信任反汇编器并回核操作数。

### 最小自检命令

```bash
clang -c -o foo.o foo.s
objdump -d -r foo.o
```

在输出中任选一条带 `(%rip)` 的 `mov`/`lea` 与一条 `call`：写出各字节字段，并核对重定位行的类型与加数。能独立完成这一步，再读 [控制流](control-flow.md) 与 [重定位](relocation.md) 时，助记符就是字节结构的摘要，而不是不可拆的黑盒。


有了数据通路，还要用条件码把 `if` / 循环变成跳转。


控制流指令改写 `RIP`：无条件转移、条件转移、调用与返回、以及以寄存器或内存为目的的间接转移。条件码由 `cmp`/`test` 与算术逻辑指令写入 `RFLAGS`，再由 `jcc`/`cmovcc`/`setcc` 读取。本页把这些机制接到 `if`/`switch`/`&&`/`||` 的常见降低形态，并为 [栈帧](stack-frames.md) 中的 `call`/`ret` 提供控制流侧的基础。

### 比较与测试

`cmp src, dst` 按 \(dst - src\) 更新标志，不写回结果。随后的条件跳转解释这些标志：

```asm
    cmpq    %rsi, %rdi      # 标志 ← rdi - rsi
    jl      .L_lt           # 有符号：rdi < rsi
    jb      .L_below        # 无符号：rdi < rsi
```

同一条 `cmp` 之后，有符号与无符号谓词读取不同标志组合；助记符选错等价于把有符号比较做成无符号比较。

`test src, dst` 按 \(dst \land src\) 更新标志，常用形态：

```asm
    testq   %rax, %rax      # 置 ZF 当且仅当 rax == 0
    jz      .L_zero
    testl   $1, %edi        # 测最低位
    jnz     .L_odd
```

`addq`/`subq`/`andq`/`xorq`/`inc`/`dec` 等也会改标志；`lea` 与多数纯 `mov` 不改标志。需要「在保留标志的前提下计算地址」时用 `lea`；需要「根据结果分支」时用算术或 `cmp`/`test`。

### 条件跳转

短跳（8 位位移）与近跳（32 位位移）语义相同，编码长度不同；汇编器按距离选择，也可在反汇编里看到 `je` 对应 `74 xx` 或 `0f 84 disp32`。常见助记符：

| 助记符 | 条件 |
| --- | --- |
| `je` / `jz` | `ZF=1` |
| `jne` / `jnz` | `ZF=0` |
| `jl` / `jnge` | 有符号小于 |
| `jle` / `jng` | 有符号小于等于 |
| `jg` / `jnle` | 有符号大于 |
| `jge` / `jnl` | 有符号大于等于 |
| `jb` / `jnae` / `jc` | 无符号低于 / `CF=1` |
| `ja` / `jnbe` | 无符号高于 |
| `jbe` / `jna` | 无符号低于等于 |
| `js` / `jns` | `SF` |
| `jo` / `jno` | `OF` |

`jmp` 无条件转移。标签方向由汇编器计算位移；跨翻译单元的目标留下重定位，见 [重定位](relocation.md)。

`setcc r8` 把条件写成 0/1 到 8 位寄存器，用于无分支的布尔物化：

```asm
    cmpq    %rsi, %rdi
    setl    %al             # al = (rdi < rsi) ? 1 : 0
    movzbl  %al, %eax
```

`cmovcc` 在条件成立时做寄存器间传送，两条路径共享同一落点，控制流图呈直线。分支预测代价高且体量小的条件赋值，编译器常选 `cmov`；复杂两侧仍用 `jcc`。

### 间接跳转与调用

直接 `jmp label` / `call label` 的目标在编码或重定位里固定。间接形式从寄存器或内存取目标地址：

```asm
    jmpq    *%rax
    jmpq    *(%rax)
    jmpq    *table(,%rdi,8)
    callq   *%rax
    callq   *(%rax)
```

`ret` 从栈顶弹出返回地址到 `RIP`；`ret $imm` 在弹出后再把 `%rsp` 加上立即数，用于清理调用方已推入的栈参数。SysV 下多数用户态 C 调用靠寄存器传参，`ret` 无立即数是常态；清理栈参数是调用方责任（与某些 stdcall 约定不同）。

间接跳转是 `switch` 跳转表、函数指针、C++ 虚调用、动态链接 PLT 首跳等机制的共同底座。目标在寄存器里时，静态反汇编只显示 `jmpq *%rax`，实际目的地取决于运行时值；调试需结合寄存器或表内容。

### 从 if 到基本块

```c
long max(long a, long b) {
    if (a > b) return a;
    return b;
}
```

一种直接降低：

```asm
    .globl  max
max:
    cmpq    %rsi, %rdi
    jle     .L_b
    movq    %rdi, %rax
    ret
.L_b:
    movq    %rsi, %rax
    ret
```

控制流图为菱形：比较后分出两条路径，在返回前汇合（此处汇合发生在各自的 `ret`）。`-O2` 后可能收成：

```asm
    movq    %rdi, %rax
    cmpq    %rsi, %rdi
    cmovle  %rsi, %rax
    ret
```

标志来源仍是比较，只是用条件传送代替跳转。读优化前后输出时，先识别「比较 → 谓词 → 汇合」，再判断汇合是标签汇合还是 `cmov` 汇合。

短路逻辑按边是否执行来降低：

```c
if (p != 0 && *p > 0) ...
```

```asm
    testq   %rdi, %rdi
    jz      .L_false
    cmpq    $0, (%rdi)
    jle     .L_false
    # 真分支
.L_false:
```

第一个条件失败则跳过第二次求值，从而跳过潜在的空指针解引用。`||` 同理：首次成功则跳过其余。顺序与副作用是否发生，由跳转边决定。

### 从 switch 到跳转表

稠密整型 `switch` 常降低为「规范化下标 + 查表间接跳」：

```asm
    # 假定下标已在 %edi，且已排除越界 / default
    movslq  %edi, %rdi
    leaq    .Ltable(%rip), %rcx
    jmpq    *(%rcx,%rdi,8)

    .section    .rodata
    .p2align    3
.Ltable:
    .quad   .Lcase0
    .quad   .Lcase1
    .quad   .Lcase2
    .quad   .Lcase3
```

稀疏 `switch` 可能落成比较链、二分比较树或哈希跳转。阅读时先找范围检查与 default 边，再找表基址与间接 `jmp`。PIC 下表项可能存相对偏移，运行时再加基址；以具体 `objdump` 为准。

### 循环与回边

`for` / `while` 降低为「入口条件检查 + 循环体 + 回边」。一种常见布局是条件在顶：

```asm
    jmp     .Lcond
.Lbody:
    # 循环体
.Lcond:
    cmpq    %r12, %rbx
    jl      .Lbody
```

另一种是条件在底（do-while 或编译器旋转后的循环）：体先执行，再决定是否跳回。回边是 CFG 中指向已出现基本块的边；优化器的循环分析依赖回边识别，阅读反汇编时可用回边定位循环头。

`loop` 助记符（隐含递减 `%rcx` 并条件跳转）在现代编译器输出中少见，手写演示偶尔出现；语义上等价于对计数器的 `dec` + `jnz`，但编码与标志副作用不同，不宜与显式比较混用而不核对手册。

### 调用作为控制流

`call` 同时完成两件事：把返回地址（下一条指令地址）压栈，并把 `RIP` 设为目标。`ret` 逆操作。与 `jmp` 的差别在于返回地址是否入栈——ABI 层的参数、栈对齐、保存责任建立在这一对原语之上。

```asm
    movq    %rdi, %r12         # 跨调用仍需使用，放入 callee-saved
    movq    $1, %rdi
    call    foo
    movq    %r12, %rdi
    call    bar
```

上例在控制流上是两条顺序调用边；在数据流上依赖 [SysV ABI](sysv-amd64-abi.md) 的保存规则，否则 `%rdi` 在第一次 `call` 后不可再当作旧参数使用。尾调用在满足 ABI 与栈布局约束时可把 `call`+`ret` 收成 `jmp`，从而复用当前帧；能否尾调用取决于参数区是否仍被调用方需要、返回地址是否已由当前帧负责等，细节在编译器后端讨论，阅读时若见到函数末尾的 `jmp` 指向其他函数，应核对是否在做尾调用。

下一页 [过程调用与栈帧](stack-frames.md) 展开帧布局与 prologue/epilogue，把 `%rsp` 的变化画到与 `%rip` 对称的位置。阅读建议：先在 `objdump -d` 里标出所有 `jcc`/`jmp`/`call`/`ret`，画出基本块与边，再回头填每条边上的条件码来源；控制图清楚之后，寄存器与栈槽的解读才有锚点。
