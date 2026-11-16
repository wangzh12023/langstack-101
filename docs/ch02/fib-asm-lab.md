# 端到端实验：手写 fibonacci

在 Linux x86-64 上用汇编实现迭代版 `fib`，再与 C 在 `-O0` / `-O2` 下的产物对照。目标是把 [SysV ABI](sysv-amd64-abi.md)、[栈帧](stack-frames.md) 与 `objdump -dr` 读法串成一次可复现实验。

## 迭代版 fib（SysV）

参数 `n` 在 `%edi`（`int`），返回值在 `%eax`。

```asm
# fib.s — AT&T，Linux x86-64
.globl fib
.text
fib:
    movl    %edi, %ecx          # n
    cmpl    $2, %ecx
    jl      .Lsmall
    movl    $0, %eax            # a = 0
    movl    $1, %edx            # b = 1
.Lloop:
    movl    %eax, %esi          # t = a
    addl    %edx, %eax          # a = a + b
    movl    %esi, %edx          # b = t
    decl    %ecx
    cmpl    $1, %ecx
    jg      .Lloop
    ret
.Lsmall:
    movl    %edi, %eax
    ret
```

```asm
# main.s — 调用 fib(10)，用 syscall 写出错误码式退出码
.globl _start
.text
_start:
    movl    $10, %edi
    call    fib
    movl    %eax, %edi          # exit status = fib(10) = 55
    movl    $60, %eax           # sys_exit
    syscall
```

!!! note "实验步骤"

    ```bash
    as -o fib.o fib.s
    as -o main.o main.s
    ld -o fib main.o fib.o
    ./fib; echo $?    # 55
    objdump -d fib
    objdump -dr fib.o
    ```

## 与 C 对照

```c
/* fib.c */
int fib(int n) {
  if (n < 2) return n;
  int a = 0, b = 1;
  for (int i = n; i > 1; --i) {
    int t = a;
    a = a + b;
    b = t;
  }
  return a;
}
```

```bash
clang -O0 -c -o fib_c0.o fib.c
clang -O2 -c -o fib_c2.o fib.c
objdump -d fib_c0.o
objdump -d fib_c2.o
clang -O2 -emit-llvm -S -o fib.ll fib.c
```

观察：`-O0` 是否把局部量放在栈上；`-O2` 是否只剩寄存器循环；递归写法在 `-O2` 下是否仍保留双调用。递归版主要压调用开销，与本页迭代版不是同一压力，读基准时要分开，见 [性能数字的读法](../ch06/reading-benchmarks.md)。

## 检查清单

- `call fib` 前后 `%rsp` 对齐与返回地址位置能否指认  
- `fib.o` 的 `objdump -dr` 中对外部符号的重定位（若有）  
- 入口若改用 crt + `main`，对比纯 `_start` 版本的差异，见 [进程入口与 C 运行时](crt-entry.md)
