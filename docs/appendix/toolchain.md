# 工具链速查

主线命令以 Linux x86-64 为例。

## 编译与反汇编

```bash
clang -O0 -g -o fib fib.c
clang -O2 -S -o fib.s fib.c
clang -O2 -emit-llvm -S -o fib.ll fib.c
objdump -d -M intel fib          # 需要 Intel 语法时
objdump -d fib                   # 默认 AT&T
objdump -dr fib.o                # 反汇编并显示重定位
readelf -h -l -S -s a.out
nm -C a.out
```

## LLVM 管线

```bash
clang -O0 -emit-llvm -S -o a.ll a.c
opt -O2 -S -o a.opt.ll a.ll
llc -O2 -o a.s a.opt.ll
llc -filetype=obj -o a.o a.opt.ll
```

## 链接与动态符号

```bash
clang -fPIC -shared -o libf.so f.c
clang -o main main.c -L. -lf -Wl,-rpath,'$ORIGIN'
readelf -d main | grep -E 'NEEDED|RPATH|RUNPATH'
objdump -R main                  # 动态重定位
```

## 运行与观测

```bash
./main
strace -e write,exit_group ./main
perf stat ./main
gdb ./main
```

语言专用工具（`javap`、`go tool objdump`、`rustc --emit=llvm-ir` 等）在各语言分论页展开。
