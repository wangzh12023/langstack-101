# 控制流与条件码

控制流指令改写 `RIP`：无条件转移、条件转移、调用与返回、以及以寄存器/内存为目的的间接转移。条件码由 `cmp`/`test` 与算术逻辑指令写入 `RFLAGS`，再由 `jcc`/`cmovcc`/`setcc` 读取。本页把这些机制接到 `if`/`switch` 的常见降低形态。

## 比较与测试

`cmp src, dst` 按 \(dst - src\) 更新标志，不写回结果。随后的条件跳转解释这些标志：

```asm
    cmpq    %rsi, %rdi      # 标志 ← rdi - rsi
    jl      .L_lt           # 有符号 rdi < rsi
```

`test src, dst` 按 \(dst \land src\) 更新标志，常用形态：

```asm
    testq   %rax, %rax      # 与自身与，置 ZF 若 rax==0
    jz      .L_zero
    testl   $1, %edi        # 测最低位
    jnz     .L_odd
```

有符号与无符号谓词使用不同标志组合：`jl`/`jle`/`jg`/`jge` 看 `SF`/`OF`/`ZF`；`jb`/`jbe`/`ja`/`jae` 看 `CF`/`ZF`。同一组标志上选错助记符，等价于把有符号比较做成无符号比较。

## 条件跳转

短跳（8 位位移）与近跳（32 位位移）语义相同，编码长度不同。常见助记符：

| 助记符 | 条件 |
| --- | --- |
| `je` / `jz` | `ZF=1` |
| `jne` / `jnz` | `ZF=0` |
| `jl` / `jnge` | 有符号小于 |
| `jle` / `jng` | 有符号小于等于 |
| `jg` / `jnle` | 有符号大于 |
| `jge` / `jnl` | 有符号大于等于 |
| `jb` / `jnae` | 无符号低于 |
| `ja` / `jnbe` | 无符号高于 |
| `js` / `jns` | 符号位 |
| `jo` / `jno` | 溢出位 |

`jmp` 无条件转移。向前/向后标签由汇编器计算位移；跨节区或外部符号则留下重定位。

`setcc r8` 把条件写成 0/1 到 8 位寄存器，用于无分支的布尔物化；`cmovcc` 在条件成立时做寄存器间传送，失败路径仍顺序执行，避免部分分支预测代价。

## 间接跳转与调用

直接 `jmp label` / `call label` 的目标在编码里（或经重定位）固定。间接形式从寄存器或内存取目标地址：

```asm
    jmpq    *%rax
    jmpq    *(%rax)
    jmpq    *table(,%rdi,8)
    callq   *%rax
```

`ret` 从栈顶弹出返回地址到 `RIP`；`ret $imm` 在弹出后再把 `%rsp` 加上立即数，用于清理调用方已推入的栈参数（SysV 下少见，stdcall 等约定中更常见）。

间接跳转是 `switch` 跳转表、函数指针调用、C++ 虚调用、动态链接 PLT 首跳等机制的共同底座。目标地址在寄存器里时，静态反汇编只能看到 `jmpq *%rax`，实际目的地取决于运行时值。

## 从 if 到基本块

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

编译器在优化后可能改成 `cmovg` 等无分支形式，控制流图从菱形收成直线，但标志来源仍是比较。读 `-O0` 与 `-O2` 输出时，先识别「比较 → 条件 → 汇合」，再看是否被重写成条件传送。

短路逻辑 `a && b` 变成顺序基本块：第一个条件失败则跳过第二个的求值；`||` 同理。顺序与副作用是否执行，由跳转边决定，不是由单一布尔指令决定。

## 从 switch 到跳转表

稠密整型 `switch` 常降低为「规范化下标 + 查表间接跳」：

```asm
    # 假定 index 已在 %edi，且 0..3 已分出 default
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

稀疏 `switch` 可能落成二分比较链或哈希跳转。无论哪种，入口条件检查（范围、default）与间接跳转的目标表是阅读重点。表项本身是地址常量，在 PIC 下可能存相对偏移再加基址，细节随编译器版本变化；以具体 `objdump` 为准。

## 调用作为控制流

`call` 同时完成两件事：把返回地址（下一条指令地址）压栈，并把 `RIP` 设为目标。`ret` 逆操作。这与 `jmp` 的差别仅在于返回地址是否入栈——ABI 层面的参数、栈对齐、保存责任建立在这一对原语之上，下一页 [过程调用与栈帧](stack-frames.md) 展开帧布局，再下一页 [SysV ABI](sysv-amd64-abi.md) 给出寄存器契约。
