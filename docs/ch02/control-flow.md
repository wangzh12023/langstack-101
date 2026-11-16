# 控制流与条件码

控制流指令改写 `RIP`：无条件转移、条件转移、调用与返回、以及以寄存器或内存为目的的间接转移。条件码由 `cmp`/`test` 与算术逻辑指令写入 `RFLAGS`，再由 `jcc`/`cmovcc`/`setcc` 读取。本页把这些机制接到 `if`/`switch`/`&&`/`||` 的常见降低形态，并为 [栈帧](stack-frames.md) 中的 `call`/`ret` 提供控制流侧的基础。

## 比较与测试

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

## 条件跳转

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

## 间接跳转与调用

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

## 从 switch 到跳转表

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

## 循环与回边

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

## 调用作为控制流

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
