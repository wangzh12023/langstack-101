# ELF 文件结构

ELF（Executable and Linkable Format）是 Linux 上可重定位目标、可执行文件与共享库的容器格式。同一套头结构服务两种视图：**链接视图**用节头表描述 `.text`、`.data`、重定位与符号；**执行视图**用程序头表描述装载段与解释器。本页建立字段级地图，装载步骤见 [装载](loading.md)，重定位见 [重定位](relocation.md)。

## 文件头

文件起始为 `Elf64_Ehdr`。检视：

```bash
readelf -h a.out
```

关键字段：`e_type`（`ET_REL` 可重定位、`ET_EXEC` 可执行、`ET_DYN` 共享库或 PIE）、`e_machine`（`EM_X86_64`）、`e_entry`（入口虚拟地址；`ET_REL` 常为 0）、`e_phoff`/`e_phnum`（程序头）、`e_shoff`/`e_shnum`（节头）。口试「入口为何不是 main」时，`e_entry` 指向的值通常是 `_start`，见 [crt 入口](crt-entry.md)。

## 节头表：链接视图

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

## 程序头表：执行视图

```bash
readelf -l a.out
```

`PT_LOAD` 描述映射到内存的段：文件偏移、虚拟地址、文件大小、内存大小、权限（`rwx`）。`p_memsz > p_filesz` 的部分在运行时补零，覆盖 `.bss` 一类需求。`PT_INTERP` 给出动态链接器路径；`PT_DYNAMIC` 指向动态节；`PT_TLS` 描述 TLS 模板；`PT_GNU_STACK` 等表达栈可执行性等 GNU 扩展。

链接器把许多节打进少数 `PT_LOAD`：例如只读的 `.text` 与 `.rodata` 常共享同一 `r-x` 段。读装载问题时以程序头为准；读重定位与符号问题时以节头为准。

## 符号表

```bash
readelf -s foo.o
nm foo.o
```

符号绑定（`LOCAL`/`GLOBAL`/`WEAK`）、类型（`FUNC`/`OBJECT`）、所在节索引，决定静态链接能否解析以及动态链接是否导出。`UND` 表示本文件未定义、待链接器或加载器解析。

## 节视图与段视图

编译器/汇编器产出 `ET_REL`，几乎只有节视图。链接器输出 `ET_EXEC` 或 `ET_DYN`，两种表都在：节表方便调试与 `objdump`，程序头驱动 `mmap` 式装载。剥离（`strip`）可去掉 `.symtab` 等，不影响 `PT_LOAD` 装载，但影响调试与部分静态分析。
