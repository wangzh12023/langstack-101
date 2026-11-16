# 汇编与二进制

本章把「源文件里的符号」接到「进程映像里可执行的字节」。主线是 Linux x86-64、System V AMD64 ABI、ELF：先写可读的汇编，经汇编器与链接器变成可加载对象，再用反汇编与 ELF 检视工具核对字节是否与约定一致。第一章给出的寄存器宽度、内存与 ISA 契约在这里变成可检验的机器事实；第三章起再把这些事实接到编译器 lowering。

读完本章后，应能手写或阅读一段带函数调用的 AT&T 汇编、对照 `objdump` 说出返回地址与参数寄存器落点，并解释一条静态重定位如何改写目标字节。口试量规第 1 题（SysV 调用路径）与第 3、4 题（PLT/GOT、进程入口）的正文落点都在本章。

## 章节路径

本章按三条并行线索推进，阅读顺序以前两条为主干，第三条在能读简单 `objdump` 之后接入。

| 线索 | 页面 | 产出 |
| --- | --- | --- |
| 指令级读写 | [段与指令](sections-directives.md) → [寄存器](registers.md) → [寻址](addressing.md) → [编码与 RIP-relative](encoding-rip-relative.md) → [控制流](control-flow.md) → [栈帧](stack-frames.md) → [SysV ABI](sysv-amd64-abi.md) | 能解释一次 `call`/`ret` 与参数传递 |
| 运行时边界 | [crt 入口](crt-entry.md) → [系统调用](syscalls.md) → [TLS](tls.md) → [原子](atomics-x86.md) → [信号](exceptions-signals.md) → [内联汇编](inline-asm.md) | 能把用户态入口、syscall、异步帧与汇编约束说清 |
| 目标文件与装载 | [ELF](elf.md) → [静态链接](static-linking.md) → [重定位](relocation.md) → [PIC 与动态链接](pic-dynamic.md) → [装载](loading.md) | 能指着节区与重定位记录说明字节如何被改写 |
| 端到端核对 | [手写 fibonacci](fib-asm-lab.md) | 产出可运行 ELF，并自述反汇编 |

前几页默认 AT&T 语法、GAS/`clang -S` 风格；AArch64 只在 [SysV 页末的 AAPCS64 对照](sysv-amd64-abi.md#aarch64-aapcs64-对照)出现，不平行铺开第二套全书。

## 工具链角色

四个命令覆盖「源 → 目标 → 可执行 → 检视」的最小闭环。它们回答的问题彼此不同，不要混用输出形态。

`as`（或 `clang -c` 内嵌的汇编器）把汇编文本变成可重定位目标文件：解析段指令、解析立即数与符号引用、按目标 ISA 写出机器码与重定位记录。单独使用时，输入是 `.s`，输出通常是 `.o`。

`ld`（或 `clang` 驱动调用的链接器）把一个或多个 `.o` 与库合成可执行文件或共享库：解析未定义符号、应用重定位、安排段到虚拟地址、写入程序头与动态段。日常构建更常经编译器驱动完成；直接调 `ld` 时必须显式给出入口、crt 与库搜索路径。

`objdump` 面向「指令与机器字节」。常用形态：

```bash
objdump -d -M intel=0 a.out    # 反汇编；本站默认 AT&T，即不加 -M intel
objdump -d -r foo.o            # 反汇编并显示重定位
objdump -s -j .text a.out      # 倾倒节区原始十六进制
```

读输出时先对齐地址列与字节列，再对助记符；[编码页](encoding-rip-relative.md)用同一路径示范如何从字节还原 ModR/M 与 RIP-relative 位移。

`readelf` 面向「ELF 元数据与装载视图」。常用形态：

```bash
readelf -h a.out               # ELF 头：入口、机器类型、端序
readelf -S a.out               # 节头表
readelf -l a.out               # 程序头 / 装载段
readelf -r foo.o               # 重定位表
readelf -s a.out               # 符号表
```

`nm` 可作符号表的轻量替代；涉及动态段、哈希、版本依赖时仍以 `readelf -d` 为准。工具安装与版本差异见 [工具链附录](../appendix/toolchain.md)。

## AT&T 与 Intel 语法

同一条 x86-64 指令有两套常见文本写法。本站正文与代码块默认 AT&T（GNU `as`、`objdump` 默认）：操作数顺序为**源在前、目的在后**；立即数带 `$`；寄存器带 `%`；内存操作数写作 `disp(%base,%index,scale)`；操作数宽度由助记符后缀 `b`/`w`/`l`/`q` 或显式操作数体现。

Intel 语法（NASM、MASM、`objdump -M intel`）相反：目的在前、源在后；立即数无 `$`；寄存器无 `%`；内存用 `[base+index*scale+disp]`；宽度常用 `dword ptr` 一类前缀。对照同一条加法：

```asm
# AT&T
addq $1, %rax

; Intel
add rax, 1
```

阅读他人笔记或 IDE 反汇编时，先确认语法再谈语义。本站示例不混用两套记号；若某段必须用 Intel，会在该段标明。

## 本章终点

满足下列可检验条件，即可进入第三章，把同一段机器行为接到 LLVM IR 与 `-O2` 改写：

1. 能画出一次函数调用前后的栈布局：返回地址、对齐约束、红区，并能指出整数与浮点参数所在寄存器（见 [栈帧](stack-frames.md)、[SysV ABI](sysv-amd64-abi.md)）。
2. 能对一小段 `.o` 或可执行文件使用 `objdump -d` / `readelf -r`，把一条 PC-relative 引用或重定位记录对到具体字节。
3. 能说明用户态入口为何通常不是 `main`，以及一次 `syscall` 与一次普通 `call` 在控制转移上的差别（见 [crt 入口](crt-entry.md)、[系统调用](syscalls.md)）。

[手写 fibonacci 实验](fib-asm-lab.md)把上述三点收成一份可运行产物；实验页依赖前序页的术语，不单独引入第二套约定。
