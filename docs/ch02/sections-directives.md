# 段、符号与汇编器指令

汇编源文件同时描述两类事物：指令与数据的**内容**，以及它们在目标文件里落入哪一**段**、对外可见性如何、对齐约束是什么。汇编器把段指令翻译成 ELF 节区（section）与符号表项；链接器再按节区属性合并、重定位并映射到可装载段（segment）。本页处理静态汇编里最常用的一组指令与数据伪指令；ELF 头与程序头的完整字段见 [ELF 文件结构](elf.md)。

## 三类内容段

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

## 数据伪指令

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

## 符号与可见性

标签在汇编里是地址的名字。汇编器为每个标签生成符号表项；链接器按绑定与类型解析跨文件引用。

`.globl name`（GAS 亦接受 `.global`）把 `name` 标为全局绑定：本目标文件定义、其他目标文件可引用。缺少 `.globl` 的标签默认局部（STB_LOCAL），链接阶段对外不可见，可与其他文件中的同名局部符号共存。

`.type name, @function` / `@object` 写入符号类型，供链接器、调试器与某些分析工具区分函数与数据。`.size name, .-name` 用「当前位置减去符号起点」写出符号大小，常见于函数或对象末尾；`nm` / `readelf -s` 的 size 列由此而来。

`.weak name` 声明弱符号：若链接时存在强定义则用强定义，否则保留弱定义或把未解析弱引用置零。动态库与弱引用的解析规则放到 [静态链接](static-linking.md) 与 [PIC](pic-dynamic.md)。

`.extern name` 在 GAS/ELF 目标上通常可省略：对未定义标签的引用会自动产生未定义符号。显式写出有时用于可读性。

## 对齐与填充

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

## 位置计数器与局部标签

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

## 与 ELF 节区的对应

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

## 最小可汇编单元

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
