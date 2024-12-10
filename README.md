
## 索引


* 上一篇：无
* 下一篇：待更新


## 正文开始


RyuJIT \- 即 .NET 的 JIT 编译器，负责将 IL 代码编译为最终用于执行的机器代码。


本系列为 RyuJIT 教程，将分为多篇进行更新发布，旨在给对 .NET 编译器有兴趣、以及希望参与 .NET JIT 编译器开发工作的人提供一些参考资料。


说是教程其实也只是我在社区中从事 RyuJIT 相关开发工作的一些经验和见解，抛砖引玉。


如有疏漏请见谅。


## 写在前面


RyuJIT，最初也叫做 JITBlue，是微软在大概 2014 年前后为 .NET 发布的新一代 JIT 编译器。“Ryu”则取自日语中龙（竜）的发音，也算是致敬编译原理的经典书籍——“龙书”。


而 .NET 作为上世纪末就诞生的平台，原有的 JIT 在生产中用了将近 20 年，为什么需要一个新的 JIT 呢？这就需要了解一些历史。


### 一些历史


.NET 最初的 JIT 编译器也叫做 JIT32，顾名思义这是给 32 位架构设计的编译器，最早可以追溯到 1996 年。在当时 32 位是主流架构，并且人们也只关心 32 位架构。JIT32 设计上是一个非常轻量的编译器，并且拥有良好的代码生成质量。


然而进入 21 世纪之后 IA64 架构的诞生，.NET 需要运行在 64 位的服务器上。这个时候代码生成质量成了关键，而服务器并不在乎编译时间，并且内存也很大，.NET 选择了采用 C\+\+ 编译器后端（UTC，Universal Tuple Compiler）作为优化器，带来的结果就是支持了 IA64 架构，但同时编译器也用到了大量的 O(N^X) 复杂度的优化算法，并且很吃计算机资源。


然后 AMD64 诞生了，这个时候简单的把 IA64 移植到 AMD64 架构上，就带来了此后一直沿用了十年的 JIT64。然而此时 64 位架构不再是服务器的专属，个人电脑也开始用 64 位架构了。


2010 年的时候由于 Windows RT 在 ARM 设备上的尝试，.NET 需要支持 ARM32 架构。由于个人终端设备的资源很有限，.NET 选择了将 JIT32 移植到 ARM32 上。然而 ARM32 和 x86 虽然都叫做 32 位，实际上几乎没有任何相同之处。虽然花费了大量的精力做了移植，此时的 JIT32 for ARM 的代码质量实际上相当的糟糕，主打一个能用就行。


而到了 2012 年的时候，ARM64 要来了。此时我们回顾一下历史，就会发现在这个时候：


* 用于 x86 的 JIT32 现在跟不上时代
* 用于 x64 的 JIT64 编译速度很慢而且资源消耗大
* 用于 ARM32 的 JIT32 代码质量很差且难以解决


那 ARM64 应该用哪个实现呢？很显然无论哪个都难以满足现代 JIT 的需求。


于是此时 RyuJIT（JITBlue）项目启动了，既然要做一个新的玩意，那自然要跟上最新的架构。于是 RyuJIT 的目标自然就是：


* 生成的代码质量要高（性能好）
* 吞吐量要高（编译快）
* 在所有架构上都有一致的可预测的性能


要做到这些，自然要用上现代的编译器架构：


* 采用基于 SSA（Static Single Assignment）的优化算法
* 能够充分利用类型信息的 VN（Value Numbering）
* 单一代码库支持各种新特性，比如 SIMD 等
* 架构相关的部分（lowering、codegen）相互隔离
* 等等...


### RyuJIT


RyuJIT 复用了 JIT32 的树状 IR 结构，重写大部分前端，然后在 Rationalization 这个步骤将树形 IR 转换为线性 IR 后交给新写的后端。后端的 register allocator 这次用上了 LSRA 而不是 JIT64 那样的图着色来提升编译速度。


这里有一个有趣的小插曲，RyuJIT 最初是打算只重写大部分前端，然后 Rationalization 后就直接扔给 JIT32 去做代码生成的，结果做着做着发现这完全就是个错误的决定，于是最后把后端也重写了。


老的 JIT64 的 IR 结构是线性的，毕竟 UTC 顾名思义就是主要在线性 IR 上做文章的编译器，而把 IL 导入成线性 IR 的开销非常大。由于 RyuJIT 是将 IL 导入成树形的 IR，这相比导入到线性 IR 要容易得多，并且速度也更快。


另外，换上了现代编译器架构，将基于 lexical 的算法更换为基于 semantic 的算法，用上了基于 SSA 和 VN 的优化，RyuJIT 能编译出质量相当优秀的代码。


多亏了 SSA，让 RyuJIT 能用上各种线性或者近线性的算法，最终编译速度也比 JIT64 快得多，开销也要更小。而 SSA 也成为了构建 VN 的基石，允许 RyuJIT 引入基于 VN 的各种高级优化。


## 架构


RyuJIT 在设计上与 runtime 完全独立，作为一个独立的编译器组件存在，不依赖任何的 runtime 实现。因此你可以很容易地将 RyuJIT 作为一个独立的编译器模块拿去给别的项目使用。


正因此，也诞生了不少有趣的项目，例如：


* Pyjion：一个利用 RyuJIT 给 Python 实现了 JIT 的项目
* CoreRT/NativeAOT：把 RyuJIT 当作代码生成引擎的 AOT 编译器
* NativeAOT\-LLVM：一个把 RyuJIT 和 LLVM 组合实现了 IL 编译到原生 WebAssembly 的项目
* 等等...


RyuJIT 凭借其多架构支持、出色的编译速度和良好的代码生成质量成为了高性能编译器的一个很好的选择，可以兼顾编译速度和代码性能。而且 RyuJIT 虽然名字里有 JIT，但得益于其模块化的设计，拿去集成到一个 AOT 编译器里也是完全没有问题！


## 编译阶段


RyuJIT 的编译过程由多个阶段（Phase）组成，整体的编译流程大概如下：


[![overview](https://img2024.cnblogs.com/blog/1590449/202412/1590449-20241209205401097-1694543850.png)](https://github.com)


其中，Importer 到 Rationalization 之前被称为 RyuJIT 的前端，而 Rationalization 到 Code Generation 被称为 RyuJIT 的后端。


### Importer


IL 代码首先会经由 Importer 被导入到 RyuJIT IR。


这个过程会展开各种 intrinsics。


### Inliner


决定导入的方法调用是否应该被内联，这一阶段涉及到了各种玄学 heuristics 计算收益值去决定是否应该内联一个方法。


### Morph


这个过程会对 BB 进行各种变换，对后续的优化阶段做准备。


首先利用指针分析决定对象到底是分配在堆上还是栈上，然后消除死代码和不必要的地址暴露等等，然后构建 liveness 信息得到 use\-def 图，以进行 forward substitution、physical promotion、copy omission 等等，最后插入 GS cookies。


于此同时 `QMARK` 和 `COLON` 也被展开成了块。


### Loop Optimizations


这个过程会识别循环并对循环进行优化。


循环优化包含了一系列优化的组合，例如 loop inversion、loop cloning、loop unrolling 等等。


顺便这个过程还会试图删除没必要的 try\-catch\-finally 块。


### SSA 和 VN\-based Optimizations


这个过程会进行数据流的分析，构建 SSA 和 VN。


然后进行各种 VN\-based 的优化：loop invariants hosting、copy propagation、branch removal、CSE（Common Sub\-expression Elimination）、assertion propagation、bounds check elimination、induction variable optimization 和 dead\-store removal 等等。


这里多亏了 SSA 和 VN 使得这些不需要依赖 lexical\-based 的方法，而可以通过 VN 来判断等价计算从而做到精确的优化。


接着删除不必要的 try\-catch\-finally，并内联类型转换、runtime lookup、static 成员初始化和 thread\-local 访问。


最后做各种 boolean 表达式折叠，以及识别 switch 方便后续展开成 jump table 等等。


### Rationalization


这个阶段构建 IR 的线性表达形式，使得 IR 既可以按照树的形式进行遍历，也可以按照执行顺序的线性形式进行遍历。线性形式将主要用于后端的各阶段。


Rationalization 阶段还会消除掉所有的 `COMMA` 和 statements，从而使得 BB 的执行顺序能够完全被 GenTree 的链表来表达。


顺便一提，这一阶段后的 RyuJIT IR 可以轻而易举地被转换为 LLVM IR。


### Lowering


在这个阶段，会按照执行顺序遍历 IR，展开 jump table，并计算 addressing mode，还会给每个节点标记寄存器的需求和约束，以方便后续 register allocator 分配寄存器。


此阶段是架构相关的，各架构有着独立的实现。


### Register Allocation


这个阶段会进行寄存器分配。这里采用了线性算法（LSRA）来分配寄存器。


### Code Generation


最后来到代码生成。这个阶段同样是架构相关的，各架构有着独立的实现。


这个阶段会决定 frame 布局，遍历各 block 按照执行顺序生成代码、GC 和调试信息，然后生成 prolog 和 epilog。


## 架构隔离设计


目前的 RyuJIT 拥有广泛的架构支持，例如 x86、x64、ARM32、ARM64、LoongArch64 和 RISC\-V64 等等。


还记得我前面说的 RyuJIT 在架构相关的部分相互隔离的设计吗？正是因为这个设计使得给 RyuJIT 添加新的架构和平台支持变得非常简单。


最前面提到的 NativeAOT\-LLVM 项目就是例子之一。


NativeAOT\-LLVM 在 Rationalization 之后把 RyuJIT IR 转换为 LLVM IR 后去调用 LLVM 的编译器，从而使得代码能够被 LLVM 优化；然后按照 Lowering 和 Code Generation 的接口分别实现了 WebAssembly 平台的实现。不仅同时享受到了来自 RyuJIT 和 LLVM 两边的优化，同时还为 RyuJIT 扩展出了 WebAssembly 这一新架构的支持。


## 结尾


这一篇文章就暂时就先写到这里。


JIT 是一个很复杂的项目，还涉及到和 runtime 的各种交互。在之后文章里，我将会首先带着大家了解 RyuJIT 和 runtime 是如何进行交互、如何请求类型系统的，然后讲讲上手 RyuJIT 开发的工具链和流程，再然后带着例子讲讲一些主要的编译阶段，最后再谈一谈一些主要的优化内容。


 本博客参考[wgetCloud机场](https://tabijibiyori.org)。转载请注明出处！
