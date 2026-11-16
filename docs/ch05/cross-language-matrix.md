# 跨语言对照表

同一套朴素算法（以递归 fibonacci 一类为探针）压在不同语言上时，墙钟差来自多层机制叠乘，而不是单一「语言快慢」标量。[求值与执行模型](../ch04/evaluation.md) 已说明基准可能落在启动、升温或稳态；本页把分论页的落点收成可对照的表，供口试第 10 题追问。

公开榜单 [Naïve Fibonacci（benjdd.com/languages）](https://benjdd.com/languages/) 适合当**出题材料**：固定输入（如 40）、多语言并排、方法论公开。它不是实现质量的最终排序——缺整数宽度与溢出约定、缺是否 warmup、缺是否计入运行时启动与 GC，也未保证同构到「同一抽象机操作」。下面表格回答「该追问什么」，不背名次。

## 执行模型与整数语义

| 语言 / 运行时 | 默认执行模型 | 整数语义（fib 相关） | 典型测量阶段陷阱 |
| --- | --- | --- | --- |
| C / C++ / Rust | AOT 机器码 | 定宽；C/C++ 有符号溢出可为 [UB](../ch04/ub-abstract-machine.md)；Rust debug/release 不同 | 优化等级、是否内联递归、是否改成迭代 |
| Go | AOT + 运行时 | 定宽补码回绕 | 逃逸分析是否把小值打到堆；基准框架是否含调度噪声 |
| Java / Kotlin JVM | 字节码 + 多层 JIT | 定宽回绕；无 C 式有符号 UB | 未升温的解释/C1；装箱；栈深 |
| C# / .NET | CIL + JIT（或 Native AOT） | 定宽；`checked` 可改语义 | 分层编译升温；async/闭包误入探针 |
| Swift | AOT（LLVM）+ ARC | 平台 `Int`；debug trap / release 回绕 | 构建配置；ARC 流量是否被消掉 |
| Python CPython | 字节码解释 | 任意精度；自动扩宽 | 实现是否实为 PyPy；是否误测 C 扩展 |
| JavaScript | 解释/基线 + 优化 JIT | `Number` 为双精度；大整数用 `BigInt` | 引擎差异；warmup；`Number` 精度 |
| Ruby YARV | 字节码解释 + IC | 立即数扩 Bignum | GVL 下线程设置；扩展是否释锁 |
| PHP Zend | 操作码（常 + OPCache） | 平台整型与溢出规则随版本 | 是否计入请求引导；OPCache 冷热 |
| Dart | JIT 或 AOT | 语言层任意精度抽象 | 声明 JIT 还是 AOT；isolate 启动 |
| R | 解释 + 向量原语 | 数值以双精度向量为主 | 标量递归与语言设计错位；是否进本地数值库 |

## 分配、调用与距 C 的远近

| 语言 / 运行时 | 分配压力（递归 fib 探针） | 调用约定相对 C | FFI 边界常见成本 |
| --- | --- | --- | --- |
| C | 几乎可仅寄存器/栈 | 即 [SysV](../ch02/sysv-amd64-abi.md) 等平台 ABI | 无（自身） |
| C++ | 默认可无堆；对象语义可引入 | 对外常 `extern "C"`；内部 name mangling | 异常与 RTTI 跨界 |
| Rust | 默认可无堆；`Box` 才堆 | `extern "C"` 收敛；自身 ABI 不稳定 | 所有权与 panic 跨界 |
| Go | 栈为主；逃逸则堆 | cgo 另约定；与 C 有编组 | cgo 线程与拷贝 |
| Java / Kotlin | 帧在栈；对象少则可无堆分配 | JNI / Panama 编组 | 引用钉住、数组临界区 |
| C# | 类似；闭包/异步态可藏分配 | P/Invoke 编组 | 钉住、字符串拷贝 |
| Swift | 栈上值类型；类与 COW 缓冲上堆 | 稳定 C 互操作；Swift ABI 有扩展 | ARC 与桥接、ObjC 消息 |
| Python | 帧对象 + `PyLong` 等堆对象 | C API / ctypes | GIL、借/新引用 |
| JavaScript | 调用帧；闭包环境；极少「裸 int」 | N-API / WASM | 句柄范围、拷贝/移交 |
| Ruby | 帧 + 对象；块闭包 | C 扩展 API | GVL、GC 销账 |
| PHP | 请求内分配；函数帧 | Zend / FFI 扩展 | 请求生命周期 |
| Dart | 视后端；isolate 堆 | `dart:ffi` | GC 与指针寿命 |
| R | SEXP/向量；递归帧贵 | `.Call` 等 | 保护栈、拷贝 |

「调用约定距离」指：热路径是否已是平台 C ABI 上的直接 `call`，还是必须经运行时入口、编组或对象协议。距离大并不自动等于慢——若 fib 被内联成常数折叠，距离问题消失；探针要防止被优化掉语义。

## 基准陷阱清单

下列每一项都能单独让跨语言名次失去解释力；口试时应能从本表指回机制，而不是争第二名差几毫秒。

1. **测到了哪一段**：启动（加载运行时、类加载、OPCache 冷启动）、JIT warmup、还是稳态峰值。
2. **整数是否同一语义**：定宽回绕、UB、任意精度、IEEE 双精度充当整数。
3. **递归是否被改写**：尾调用、内联、常量折叠、备忘录；源码同构不等于 IR 同构。
4. **分配与 GC/RC**：测量窗口是否包含停顿、周期回收、请求结束释放。
5. **是否仍在语言里**：Python/R/Ruby 扩展、JNI、SIMD 库可能把内核换成 C。
6. **并发噪声**：GC 线程、JIT 编译线程、CPU 频率缩放、基准进程亲和。
7. **构建配置**：debug vs release、checked 算术、是否带符号与消毒器。

分论页写各语言落点：[Java 与 Kotlin](java.md)、[C#](csharp.md)、[Swift](swift.md)、[Python](python.md)、[JavaScript 与 TypeScript](javascript.md)、[动态运行时样本](dynamic-runtimes.md)。读法总论见第六章「性能数字的读法」（站点导航）；本页只固定对照维度。

## 导师对话笔记

- 拿到一张 fib 榜，先问方法论：硬件、输入、`/usr/bin/time` 是否含启动、跑几次取什么统计量（benjdd 文中写了三次取最低——这本身改变尾部噪声解读）。
- 用「执行模型 × 整数 × 分配 × 距 C」四列解释数量级差；禁止单因归因。
- C 与托管语言差两个数量级时，指出递归调用与整数表示是否同级，而不是只谈 GC。
- TypeScript 与 JavaScript 同列；Kotlin/JVM 与 Java 同列——擦除/同字节码意味着榜上分列多名常测的是编译器/运行时同一条管线的不同前端。
- 把本表当检查单，指回第四章轴与第五章落点，即满足口试「可指页作答」。
