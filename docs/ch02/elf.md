# ELF 文件结构

ELF（Executable and Linkable Format）是 Linux 上目标文件、可执行文件与共享库的常见容器。读 `readelf` / `objdump` 时，先分清两套表：**节（section）** 面向链接与调试，**段（segment / program header）** 面向装载。

## 文件种类

- `ET_REL`：可重定位目标文件（`.o`），含重定位记录，待链接。
- `ET_EXEC` / `ET_DYN`：可执行文件；现代发行版上 PIE 常为 `ET_DYN`。
- 共享库亦为 `ET_DYN`。

文件头给出类别、入口点（对可执行有意义）、节表/程序头表位置与体系结构。

## 节与段

常见节：`.text` 代码，`.data` 已初始化数据，`.bss` 未初始化数据（文件中不占实体空间），`.rodata` 只读常量，`.symtab` / `.dynsym` 符号，`.strtab` 字符串，`.rel*` / `.rela*` 重定位。调试信息常在 `.debug_*`。

装载器按 **program header** 把若干节打包成可映射的段（如 `PT_LOAD`），带上读写执行权限。链接脚本与编译器决定节如何落入段；汇编里的 `.section` 最终要与这些权限一致。

## 符号

符号表项把名字绑到值（地址或偏移）与节索引。全局/弱/局部绑定影响链接时的解析规则。动态符号表是运行期解析的子集。`nm` 与 `readelf -s` 是查看符号的快捷方式。

重定位如何在链接或装载时改写节内容，见 [重定位](relocation.md)。装载路径见 [可执行文件装载](loading.md)。
