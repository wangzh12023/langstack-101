# 从汇编到正在运行的进程

本章按执行顺序写 Linux x86-64 用户态程序从汇编到进程的路径。工具用 `as`/`clang`、`ld`、`objdump`、`readelf`；调用约定用 System V AMD64 ABI；汇编语法默认 AT&T。

1. [指令级程序](instructions.md)：段与符号、寄存器、寻址、编码、控制流  
2. [函数调用与 ABI](calling.md)：栈帧与 SysV 传参/返回  
3. [链接、装载与进程映像](link-load.md)：ELF、静态链接、重定位、PIC/GOT/PLT、装载、`_start` 到 `main`  
4. [内核边界上的接口](kernel-boundary.md)：系统调用、TLS、原子指令、信号、内联汇编  

[实验：手写 fibonacci](fib-asm-lab.md) 用可运行的 ELF 核对前三节。第三章讨论编译器优化之后，上述对应关系会如何变化。
