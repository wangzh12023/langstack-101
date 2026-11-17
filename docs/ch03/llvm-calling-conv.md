# 调用约定在 LLVM 中的落地

调用约定规定参数与返回值放在哪些寄存器或栈槽、谁保存哪些寄存器、入口栈如何对齐。LLVM IR 里的 `call` 仍使用虚拟形态；真正按 [System V AMD64 ABI](../ch02/sysv-amd64-abi.md)（或 Windows x64、AAPCS64 等）排布，发生在后端 lowering 与指令发射阶段。前端负责选出正确的约定属性；后端负责生成遵守该约定的序言、实参移动与 `call`/`ret` 序列。

## IR 上的调用形态

一条典型调用：

```llvm
%rv = call i32 @foo(i32 %a, ptr %p)
```

这里只固定了类型与实参值的 SSA 名，没有写「`%a` 进 `%edi`」。函数定义与调用点可以带 **calling convention** 标识（如 `ccc` 表示 C 默认约定，`fastcc`、`tailcc`、`webkit_jscc` 等表示其他约定），以及参数属性：`zeroext` / `signext`、`byval`、`sret`、`inreg`、`align` 等。这些属性把 ABI 里「窄整数如何扩展」「结构体走隐藏指针还是拆寄存器」提前说清楚，供后端选对移动指令。

返回 `sret` 结构体时，IR 常表现为额外的指针参数；前端按目标 ABI 决定是否使用该形式。变参函数还需在约定要求时生成「浮点参数个数」一类信息（SysV 下对应 `%al`），具体由前端与目标 lowering 协作完成。

## 从 call 到机器码

后端在指令选择前后把每个实参放到 ABI 指定位置：前几个整数进 `%rdi`…`%r9`，多余的进栈，浮点进 `%xmm0`…。跨 `call` 仍存活的 caller-saved 值被存到栈槽或改放 callee-saved 寄存器。被调函数若使用 `%rbx` 等，序言 push、尾声 pop。栈对齐（SysV 下 `call` 前 `%rsp ≡ 0 (mod 16)`）由帧 lowering 与调用序列共同保证。

尾调用在约定与目标支持时，可生成不增长栈的跳转式调用；IR 上的 `musttail` / `tail` 标记表达前端意图，后端再决定能否兑现。读汇编时看到的 `jmp` 代替 `call`，往往对应这一路径。

## 与语言前端的交界

C/clang 默认 `ccc`，与平台 C ABI 对齐，便于与手写汇编、其他 C 目标文件互链。Rust、Swift 等对内部函数可能使用更快的私有约定，在对外 `extern "C"` 边界再切回 C ABI；这种切换表现为不同函数上的约定属性，由前端显式标出。异常与人格（personality）相关的调用（`invoke`）还要与 unwind 表一致，细节超出本页，但同样属于「约定 + 运行时」而不只是寄存器表。

[SysV 寄存器表](../ch02/sysv-amd64-abi.md) 回答「机器上怎么传」；本页回答「LLVM 如何在 IR 属性里声明、在后端落实」；[语言语义边界](llvm-semantics-limits.md) 回答「别名等非布局类信息如何另附」。
