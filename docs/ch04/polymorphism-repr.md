# 派发机制与胖指针

多态调用的成本取决于：目标地址是否在编译期已知，以及对象旁边还要带多少元数据。下面并排三种常见形态。

## C++ 虚调用

带虚函数的对象通常在对象头附近存放 **虚表指针（vptr）**。调用 `p->f()` 时：加载 vptr → 在 vtable 固定偏移取函数指针 → 间接 `call`。单次调用至少一次额外负载与一次间接跳转；多重继承与虚继承还会加 thunk 与调整 `this` 的偏移。

## Go interface

`interface` 值在实现上可视为「类型信息 + 数据指针」（具体布局随版本优化，对话讲两字段模型即可）。调用接口方法：从类型信息找到方法表 → 间接调用。把具体值赋给接口可能分配并装箱到堆上，使看起来像「值传递」的路径产生分配。

## Rust `dyn Trait`

`&dyn Trait` / `Box<dyn Trait>` 是 **胖指针**：数据指针 + 指向 vtable 的指针（vtable 含方法与尺寸/对齐等）。调用通过 vtable 间接完成。与 Go 类似是「指针+表」；与 C++ 的差异是：C++ vptr 往往存在对象内部，Rust 的 fat pointer 把表指针放在引用一侧，对象本身可以保持朴素布局。

```mermaid
flowchart LR
  subgraph cpp [C++ 对象]
    VP[vptr] --> VT[vtable]
    VT --> F[f()]
  end
  subgraph rust [Rust dyn]
    DATA[data ptr]
    VT2[vtable ptr] --> M[method]
  end
```

## 静态派发对照

模板/泛型单态化、`final`、去虚化分析成功时，调用可变成直接 `call` 甚至内联，间接层消失。讨论「零成本抽象」时，要问最终是否还留下 vptr 负载；讨论 JS 引擎时，内联缓存（IC）用另一套机制把「看似动态」的调用在稳态下打到近直连路径——见 JavaScript 分论。
