# 链接、装载与进程映像


单独一个 `.s` 可以拼出玩具程序；真实软件是许多目标文件与库。链接决定符号变成地址，装载决定这些字节如何进地址空间，启动代码决定你为什么很少从 `main` 的第一条指令开始跑。读跨语言「谁启动慢」时，动态链接与初始化常常就藏在这里。


ELF（Executable and Linkable Format）是 Linux 上可重定位目标、可执行文件与共享库的容器格式。同一套头结构服务两种视图：**链接视图**用节头表描述 `.text`、`.data`、重定位与符号；**执行视图**用程序头表描述装载段与解释器。本页建立字段级地图，装载步骤见 [装载](loading.md)，重定位见 [重定位](relocation.md)。

### 文件头

文件起始为 `Elf64_Ehdr`。检视：

```bash
readelf -h a.out
```

关键字段：`e_type`（`ET_REL` 可重定位、`ET_EXEC` 可执行、`ET_DYN` 共享库或 PIE）、`e_machine`（`EM_X86_64`）、`e_entry`（入口虚拟地址；`ET_REL` 常为 0）、`e_phoff`/`e_phnum`（程序头）、`e_shoff`/`e_shnum`（节头）。口试「入口为何不是 main」时，`e_entry` 指向的值通常是 `_start`，见 [crt 入口](crt-entry.md)。

### 节头表：链接视图

```bash
readelf -S foo.o
```

每节有名字、类型、标志、文件偏移、地址（在可重定位对象里常为 0）、大小、对齐。常见节：

| 节 | 角色 |
| --- | --- |
| `.text` | 机器码 |
| `.rodata` | 只读数据 |
| `.data` | 已初始化可写数据 |
| `.bss` | 未初始化可写；文件中不占数据字节 |
| `.symtab` / `.strtab` | 符号与字符串（可剥离） |
| `.dynsym` / `.dynstr` | 动态链接用符号 |
| `.rela.text` 等 | 重定位记录 |
| `.dynamic` | 动态段标签（`DT_*`） |

节名字是约定；权限以标志位（`SHF_WRITE`、`SHF_ALLOC`、`SHF_EXECINSTR`）为准。汇编侧如何写入这些节见 [段与指令](sections-directives.md)。

### 程序头表：执行视图

```bash
readelf -l a.out
```

`PT_LOAD` 描述映射到内存的段：文件偏移、虚拟地址、文件大小、内存大小、权限（`rwx`）。`p_memsz > p_filesz` 的部分在运行时补零，覆盖 `.bss` 一类需求。`PT_INTERP` 给出动态链接器路径；`PT_DYNAMIC` 指向动态节；`PT_TLS` 描述 TLS 模板；`PT_GNU_STACK` 等表达栈可执行性等 GNU 扩展。

链接器把许多节打进少数 `PT_LOAD`：例如只读的 `.text` 与 `.rodata` 常共享同一 `r-x` 段。读装载问题时以程序头为准；读重定位与符号问题时以节头为准。

### 符号表

```bash
readelf -s foo.o
nm foo.o
```

符号绑定（`LOCAL`/`GLOBAL`/`WEAK`）、类型（`FUNC`/`OBJECT`）、所在节索引，决定静态链接能否解析以及动态链接是否导出。`UND` 表示本文件未定义、待链接器或加载器解析。

### 节视图与段视图

编译器/汇编器产出 `ET_REL`，几乎只有节视图。链接器输出 `ET_EXEC` 或 `ET_DYN`，两种表都在：节表方便调试与 `objdump`，程序头驱动 `mmap` 式装载。剥离（`strip`）可去掉 `.symtab` 等，不影响 `PT_LOAD` 装载，但影响调试与部分静态分析。


静态链接把若干可重定位目标与静态库中的成员合成一个可执行文件（或共享库映像中由链接期决定的部分）：解析符号引用、合并节区、按链接脚本布置地址，并应用重定位。动态链接把一部分绑定留到装载期；本节只谈链接期完成的工作。字节如何被改写见 [重定位](relocation.md)，GOT/PLT 见 [PIC 与动态链接](pic-dynamic.md)。

### 输入与输出

输入：`ET_REL` 的 `.o`，以及 `.a` 静态库（每个成员仍是 `.o`）。输出：带程序头的 `ET_EXEC`，或 PIE/共享库所用的 `ET_DYN`。

```bash
clang -c a.c b.c
clang -o prog a.o b.o           # 驱动拉入 crt 与默认库
clang -static -o prog a.o b.o   # 尽量静态进映像（视工具链与库而定）
```

直接调 `ld` 时必须显式给出入口、crt 对象与库搜索路径；日常构建经编译器驱动完成。

### 符号解析规则

每个全局符号在链接过程中处于未定义、已定义或弱定义等状态：

- 两个强定义冲突 → 多重定义错误。
- 强定义覆盖弱定义。
- 多个弱定义可并存，由实现挑选其一。
- 最终仍未定义 → 未定义引用错误。

C++ 的模板实例、部分 `inline` 与 COMDAT 节允许「多份相同定义、最终保留一份」；这与「两个不同强定义」不是同一类错误。局部符号（汇编未 `.globl`、C 的 `static` 函数）不参与跨文件解析。

静态库按**当前未定义符号集合**按需抽取成员：未引用到的 `.o` 可不进入最终映像。抽出的成员可能引入新的未定义符号，链接器继续扫描。传统单向前向扫描导致库顺序敏感——引用方应写在提供方之前；循环依赖可用重复列库或 `--start-group` / `--end-group`。

```bash
nm a.o b.o
ld --trace-symbol=foo -o prog a.o b.o ...
```

### 节合并与地址指派

同名兼容节合并进输出节，对齐取更严者。默认或显式链接脚本决定输出节顺序、VMA、以及入口符号（通常 `_start`，不是 `main`，见 [crt 入口](crt-entry.md)）。`readelf -S` / `-l` 看到的地址布局由此而来。

解析完成后，链接器按重定位记录改写机器码与数据中的地址字段。对仅在本映像内解析的符号，`call` 常落成直接 `rel32`；仍需运行时定位的符号留下动态节与 GOT/PLT 相关重定位。

### 静态解析与动态边界

即使用默认的动态链接可执行文件，**本程序内各 `.o` 之间**仍先做静态解析：内部调用不必经 PLT。只有仍对外部动态符号保持 `UND` 的引用，才进入 [PIC](pic-dynamic.md) 路径。`-static` 把所需库代码也链进映像，可去掉 `PT_INTERP`，代价是体积与补丁粒度。

同一份源码：静态进主程序的函数调用可成直接 `call`；放入共享库再被其他模块调用时，往往经 PLT/GOT。用 `nm`、`readelf -s` 与 `objdump -d` 对照「是否局部 / 是否经 `@plt`」即可看见这条边界。

### 检视

```bash
nm prog | grep ' [TtBbDdUu] '
readelf -s prog
readelf -d prog          # 动态时看 NEEDED 等
```

`T`/`t` 为代码中的定义，`U` 为未定义。静态完全解析的闭包里，业务符号不应再以 `U` 留在静态符号表中；动态符号表（`.dynsym`）另有导出与导入集合。


重定位记录告诉链接器（或加载器）：某节区内某一偏移上的字节，必须按指定公式改写成与符号地址相关的值。汇编器在尚不知最终地址时留下占位与记录；静态链接阶段大多被应用掉，动态情况则保留到装载或懒绑定。本页读 `objdump -dr` / `readelf -r` 的输出。

### 重定位记录的字段

一条重定位大致包含：偏移（相对节区）、类型、符号、加数（addend）。`REL` 把加数存在被改写的位置里；`RELA`（x86-64 ELF 常用）把加数存在记录中。

```bash
objdump -d -r foo.o
readelf -r foo.o
```

`objdump -dr` 在指令下挂着重定位行，便于和占位立即数对齐：

```text
0:  48 8b 05 00 00 00 00    mov    0x0(%rip),%rax
7:  R_X86_64_PC32   g_count-0x4
```

占位 `disp32` 为 0；链接时按类型写入最终位移。编码与 `-0x4` 加数的几何意义见 [指令编码与 RIP-relative](encoding-rip-relative.md)。

### 常见 x86-64 类型

| 类型 | 典型用途 | 公式直觉 |
| --- | --- | --- |
| `R_X86_64_64` | 绝对 64 位地址 | `S + A` |
| `R_X86_64_PC32` | 相对调用/RIP 位移 | `S + A - P` |
| `R_X86_64_PLT32` | 经 PLT 的相对调用 | `PLT(S) + A - P` |
| `R_X86_64_GOTPCREL` | 经 GOT 的 RIP-relative 加载 | `GOT(S) + A - P` |
| `R_X86_64_32` / `32S` | 截断绝对地址 | 窄即时/位移 |

其中 `S` 为符号地址，`A` 为加数，`P` 为被改写位置的地址（对 PC 相对类型与「下一条指令」的约定一起体现在 `A` 里）。TLS 另有 `TLSGD`、`TPOFF` 等类型，见 [TLS](tls.md)。

### 静态应用与保留到动态

链接同一可执行文件内已解析符号时，`R_X86_64_PC32` 可直接写成最终位移，记录不必留给加载器。对共享库导出符号或 `-fPIC` 下需经 GOT 的全局量，链接器把引用改成 GOT/PLT 形态，并在动态重定位表（如 `.rela.dyn`、`.rela.plt`）里留下装载期/绑定期记录。

因此「重定位」不是只发生在 `ld` 那一次：静态链接消掉一批，动态链接器在启动（及懒绑定时）再消掉一批。

### 手工核对步骤

1. 在 `.o` 上 `objdump -dr`，找到带 `R_X86_64_*` 的指令。  
2. 数清被改写的字节字段（常是 4 字节位移或 8 字节绝对地址）。  
3. 链接出可执行文件后对同一逻辑点再 `objdump -d`，确认占位已成具体立即数。  
4. 对 `PC32`：手算 `target = next_ip + disp32`，与符号地址对照。

能独立完成这四步，口试里解释「链接器改了哪几个字节」就不会停在术语层。


静态链接在链接期改完多数地址；共享库与 PIE 把一部分绑定推迟到装载甚至第一次调用。


共享库与 PIE 可执行文件可被装载到任意基址。位置无关代码（PIC）避免在指令流里写死绝对地址；跨模块符号经 GOT（全局偏移表）与 PLT（过程链接表）间接到达。本页覆盖口试第 3 题：一次外部调用的路径、懒绑定前后 GOT 槽内容、`-fPIC` 对寻址形态的改变。

### 为何需要 GOT 与 PLT

同一条 `call printf` 在可重定位对象里通常是相对调用占位；最终 `printf` 住在 libc 的装载基址上，基址要到运行时才知道。若把绝对地址写进调用方的 `.text`，则要么每次装载改代码段（不可分享、与 W^X 策略冲突），要么限制装载地址。

**GOT** 是可写（或经 relro 部分只读）数据表，槽位存放符号的运行时地址。代码通过 RIP-relative 加载 GOT 槽，再间接跳转/调用，`.text` 本身保持可分享。

**PLT** 是一小段可执行跳板：对每个外部函数，调用方 `call foo@plt`，进入对应 PLT 项；PLT 再经 GOT 跳到真实地址。把「首次解析」集中到跳板，便于懒绑定。

### 一次外部调用的路径

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

### -fPIC 对寻址形态的改变

**取函数地址 / 调用外部函数**：非 PIC 的可执行文件里，对外部符号仍常经 PLT；PIC/PIE 下同样经 PLT/GOT，且代码用 RIP-relative 寻址 GOT。差别在于全局**数据**访问：

- 无 PIC 的传统 `ET_EXEC`：可能对全局量使用绝对地址重定位（现今默认 PIE 已少见此形态）。
- `-fPIC` / PIE：访问本模块全局量常用 `symbol(%rip)`；访问可能被拦截或位于其他模块的全局量时，经 `symbol@GOTPCREL(%rip)` 加载地址再访存。

```asm
    movq    g_count(%rip), %rax          # 直接 RIP-relative
    movq    g_count@GOTPCREL(%rip), %rax # 先取 GOT 中的地址
    movq    (%rax), %rax
```

编译器在「符号可能进入动态符号表、允许被预装载库覆盖（copy relocation / 符号介入）」时倾向 GOT；仅内部可见且链接器证明不需要介入时，可收缩为直接 RIP-relative。读 `objdump -dr` 时以重定位类型（`GOTPCREL` vs `PC32`）为准。

### 与静态解析的分界

同一可执行文件内的静态函数或内部联动符号，链接器可直接写成 `R_X86_64_PC32`，无 PLT。只有需要运行时再定位的符号才占用 GOT/PLT 槽。口答第 3 题时按顺序说清：**call → PLT → GOT →（首次）解析器写 GOT → 真函数；再次则 PLT→GOT→真函数**；再补一句 `-fPIC` 下数据经 GOT 或 RIP-relative、代码避免绝对地址写死。


装载把 ELF 文件中的 `PT_LOAD` 段映射进进程地址空间，设置入口寄存器/栈，并把控制交给用户态入口。动态链接程序还要先装入解释器。本页与 [crt 入口](crt-entry.md) 分工：这里写内核与映射，那边写 `_start` 到 `main`。口试第 4 题两边合起来答。

### execve 之后内核做的事

`execve` 成功时，内核替换当前进程的地址空间：

1. 读 ELF 头，确认可执行类型与机器；
2. 按程序头映射 `PT_LOAD`（文件有内容的部分映射到文件，零扩展部分匿名映射）；
3. 若存在 `PT_INTERP`，装载动态链接器，记下其入口；
4. 在新栈上放置 `argc`/`argv`/`envp` 与辅助向量；
5. 把用户态指令指针设为解释器入口（动态）或 `e_entry`（静态），返回用户态。

权限来自程序头：`r-x` 代码段、`rw-` 数据段等。现代发行版默认 PIE（`ET_DYN` 可执行文件），基址由 ASLR 选定；辅助向量里的 `AT_ENTRY`、`AT_PHDR`、`AT_BASE` 反映实际布局。

### 静态与动态两条路径

**静态**：无 `PT_INTERP`。内核直接跳到主程序 `e_entry`（crt `_start`）。所有重定位在链接期已做完（TLS 等少数运行时初始化除外）。

**动态**：内核跳到 `ld.so`。动态链接器：

- 解析主程序 `.dynamic`，装载 `NEEDED` 库；
- 应用 `.rela.dyn` 等重定位，填充 GOT 中非懒绑定项；
- 按策略处理 `JUMP_SLOT`（懒或立即）；
- 跑初始化例程，再转到主程序入口。

因此「进程第一条用户态指令」在动态情况下往往位于 `ld.so`，而不是应用的 `_start`。从 `e_entry` 读到的应用入口仍在，只是由链接器转交。

### 映射与 /proc 可见性

```bash
readelf -l a.out
cat /proc/self/maps
```

`readelf -l` 给文件内的建议虚拟地址与权限；`/proc/self/maps` 给运行时实际映射，包括 `[stack]`、`[vdso]`、`[vvar]`、匿名堆与每个 `.so` 的基址。PIE 下 `a.out` 的文本基址每次可能不同，与 `objdump` 打印的「链接地址」之间差一个运行时 slide；用 RIP-relative 与辅助向量才能在无硬编码基址的情况下工作。

### 与口试第 4 题的衔接

完整短答结构：

1. **入口不是 main**：`e_entry` → `_start` / crt，主程序逻辑入口是后续才调用的 `main`。  
2. **ld.so**：有 `PT_INTERP` 时内核先进入动态链接器，再进入应用入口。  
3. **辅助向量**：内核放在栈上的 `(type,value)` 表，crt/ld.so 用其找到 PHDR、入口、页大小、随机字节等。  

细节句柄在 [crt 入口](crt-entry.md)；本节提供「映射与谁先执行」的装载侧事实。


装载把映像放进内存；crt 把「内核交出的状态」变成 `main` 可以假设的世界。


ELF 可执行文件的入口点是 ELF 头里的 `e_entry`，通常指向 C 运行时（crt）提供的 `_start`，而不是源码里的 `main`。从内核把控制交给用户态，到第一次执行 `main` 的语句，中间还有动态链接器、辅助向量解析与全局构造。本页按这条时间线展开；装载映射细节见 [可执行文件装载](loading.md)。

### 为何入口不是 main

`main` 是 C 语言约定的「程序逻辑入口」：它假定 `argc`/`argv`/`envp` 已按 ABI 布置好，`stdin`/`stdout` 已可用，`.init_array` 一类构造已跑完，并且返回值会交给 `exit` 一族做清理。这些前提不是 CPU 复位后的硬件状态，而是用户态启动代码准备出来的。

链接器把入口符号默认设为 `_start`（可通过 `-e` 改写）。手写「只有 `_start` + `syscall`」的最小程序可以绕过 crt，此时没有 libc、没有 `main`，也没有对 `argv` 的 C 层包装——[段与指令](sections-directives.md) 末尾的最小示例即属此类。与 C 混链时，保留 crt 是常态。

### 内核交出控制时的栈

`execve` 成功后，用户态栈顶附近已由内核摆好：

```text
高地址
  … 环境字符串与参数字符串 …
  NULL
  envp[n-1] … envp[0]
  NULL
  argv[argc-1] … argv[0]
  argc
低地址 ← 入口指令执行时 %rsp 附近
```

随后是**辅助向量**（auxiliary vector）：形如 `(type, value)` 的定长对，以 `AT_NULL` 结束。常见项包括 `AT_PHDR` / `AT_PHNUM`（程序头位置）、`AT_ENTRY`（应用入口）、`AT_BASE`（解释器基址）、`AT_RANDOM`（栈金丝雀等用的随机字节）、`AT_PAGESZ`。crt 与动态链接器用这些项定位自身与主程序，而不依赖写死的绝对地址。

### ld.so 与解释器

动态链接的可执行文件在程序头里带 `PT_INTERP`，指向动态链接器路径（常见 `/lib64/ld-linux-x86-64.so.2`）。内核先装载解释器，把控制交给解释器入口；解释器再：

1. 定位主程序的动态段与依赖库；
2. 做必要的重定位，填充 GOT 等；
3. 运行自身与库的初始化例程；
4. 跳转到主程序 ELF 头中的应用入口（常为 crt 的 `_start`），或按约定把入口信息交给后续启动代码。

静态链接可执行文件没有 `PT_INTERP`，内核直接跳到 `e_entry`。无论静动态，**`e_entry` 仍很少等于 `main` 的地址**——它指向启动桩。

### 从 _start 到 main

glibc / musl 的启动路径在细节上不同，用户态可见阶段大致为：

1. `_start`（汇编）从栈上取出 `argc`，算出 `argv`/`envp`，处理辅助向量；
2. 调用 `__libc_start_main`（或等价物），传入 `main` 的地址与构造/析构表；
3. 运行 `.init_array` / 构造函数，初始化 TLS、stdio 等；
4. 调用 `main(argc, argv, envp)`；
5. `main` 返回后走 `exit`，运行 `atexit` 与析构，最终 `exit_group` 系统调用。

口试第 4 题的完整答法可以收成三句：入口是 `e_entry`/`_start` 而非 `main`；动态情况下内核先进入 `ld.so`，由它完成库装载与重定位；辅助向量是内核留给用户态的机器可读启动参数，crt 用它找到 PHDR、入口与随机数等，再布置好参数后才调用 `main`。
