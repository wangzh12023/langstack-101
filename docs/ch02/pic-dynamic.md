# 位置无关代码与动态链接

共享库与 PIE 可执行文件可被装载到任意基址。位置无关代码（PIC）避免在指令流里写死绝对地址；跨模块符号经 GOT（全局偏移表）与 PLT（过程链接表）间接到达。本节要能说明：一次外部调用的路径、懒绑定前后 GOT 槽内容、`-fPIC` 对寻址形态的改变。

## GOT 与 PLT

同一条 `call printf` 在可重定位对象里通常是相对调用占位；最终 `printf` 住在 libc 的装载基址上，基址要到运行时才知道。若把绝对地址写进调用方的 `.text`，则要么每次装载改代码段（不可分享、与 W^X 策略冲突），要么限制装载地址。

**GOT** 是可写（或经 relro 部分只读）数据表，槽位存放符号的运行时地址。代码通过 RIP-relative 加载 GOT 槽，再间接跳转/调用，`.text` 本身保持可分享。

**PLT** 是一小段可执行跳板：对每个外部函数，调用方 `call foo@plt`，进入对应 PLT 项；PLT 再经 GOT 跳到真实地址。把「首次解析」集中到跳板，便于懒绑定。

## 一次外部调用的路径

编译/链接后，对 `printf` 的调用形如：

```asm
    call    printf@plt
```

PLT 项的典型轮廓（示意，具体字节随实现略有差异）：

```text
printf@plt:
    jmpq    *printf@GOT(%rip)     # 间接跳到 GOT 槽指向的地址
    pushq   $index                # 首次未绑定时才会落到这里
    jmpq    resolver@plt          # 进入动态链接器解析例程
```

**懒绑定前**：`printf` 的 GOT 槽指向 PLT 项里 `pushq`/`jmp resolver` 那一段（即「跳回自己」的下一指令），于是第一次 `call printf@plt` → `jmp *GOT` → 又回到 PLT 后半 → 解析器。

**解析器**（`_dl_runtime_resolve` 一类）按索引找到符号名，在已装载库中查找，把**真实函数地址写入 GOT 槽**，再跳转到该地址。

**懒绑定后**：GOT 槽已是 `printf` 的真实地址；再次 `call printf@plt` → `jmp *GOT` 直接进入 libc，不再走解析器。

可用下列命令对照：

```bash
objdump -d -j .plt a.out
readelf -r a.out          # .rela.plt 中的 JUMP_SLOT
LD_DEBUG=bindings ./a.out # 观察首次解析（调试用）
```

`LD_BIND_NOW=1` 或链接 `-z now` 时，启动阶段即填完 JUMP_SLOT，懒绑定路径不再出现；GOT 在绑定后可通过 RELRO 变为只读。

## -fPIC 对寻址形态的改变

**取函数地址 / 调用外部函数**：非 PIC 的可执行文件里，对外部符号仍常经 PLT；PIC/PIE 下同样经 PLT/GOT，且代码用 RIP-relative 寻址 GOT。差别在于全局**数据**访问：

- 无 PIC 的传统 `ET_EXEC`：可能对全局量使用绝对地址重定位（现今默认 PIE 已少见此形态）。
- `-fPIC` / PIE：访问本模块全局量常用 `symbol(%rip)`；访问可能被拦截或位于其他模块的全局量时，经 `symbol@GOTPCREL(%rip)` 加载地址再访存。

```asm
    movq    g_count(%rip), %rax          # 直接 RIP-relative
    movq    g_count@GOTPCREL(%rip), %rax # 先取 GOT 中的地址
    movq    (%rax), %rax
```

编译器在「符号可能进入动态符号表、允许被预装载库覆盖（copy relocation / 符号介入）」时倾向 GOT；仅内部可见且链接器证明不需要介入时，可收缩为直接 RIP-relative。读 `objdump -dr` 时以重定位类型（`GOTPCREL` vs `PC32`）为准。

## 与静态解析的分界

同一可执行文件内的静态函数或内部联动符号，链接器可直接写成 `R_X86_64_PC32`，无 PLT。只有需要运行时再定位的符号才占用 GOT/PLT 槽。口答第 3 题时按顺序说清：**call → PLT → GOT →（首次）解析器写 GOT → 真函数；再次则 PLT→GOT→真函数**；再补一句 `-fPIC` 下数据经 GOT 或 RIP-relative、代码避免绝对地址写死。
