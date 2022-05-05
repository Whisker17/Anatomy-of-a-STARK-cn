# STARKs 详解，第 1 部分：STARKs 概述

STARKs 是一种交互式证明系统。 但为了更好的理解，你可以将它们视为 SNARKs 的特殊案例：

- 散列函数是唯一的密码学集成
- 算术化基于 AIR (代数中间表示[^1])，将关于计算完整性的声明简化为关于某多项式低阶的声明
- 使用 FRI 作为子协议可以证明多项式的低阶， 并且 FRI 本身用默克树实例化 [^2];
- 零知识是可选的。

本教程的这一部分是要解释 STARKs 定义中的关键术语。

## 交互式证明系统

在计算复杂性理论中，交互式证明系统是至少两个当事方之间的协议，其中一方为验证人，且只要而且只有在该声明是正确的情况下，验证人确信相应的数学声明是正确的。 从理论上讲，声明可以用数学符号，例如 Birch 和 Swinnerton-Dyer 的推测来表述。 $\mathbf{P} \neq \mathbf{NP}$, 或 "第十五个 Fibonacci 数字是 643617." (在一个健全的证明系统中，验证人将拒绝最后一个声明。)

一个密码学证明系统将这种交互式证明系统的抽象概念变成一个打算在现实世界中部署的具体目标。 这种对现实世界应用的限制促成了一些简化：

- 该声明并不涉及数学推测，而涉及某一特定计算的完整性。 如"电路 $C$ 在输入 $x$ 时结果输出 $y$ ", 或 "图灵机 $M$ 在 $T$ 步后输出 $y$ 那么我们就说该证明系统已建立 _计算完整性_。
- 协议中有两种角色，即证明人和验证人。 在不失去通用性的情况下，验证人向证明人发送的信息包含了纯粹的随机性。在这种情况下(所以：几乎总是)，证明系统可以与 _Fiat-Shamir转换_ 不交互。 非交互式证明系统中仅有一条从证明人到验证人的信息。
- 验证人不需要保证完美的安全性，而是可以接受非常小的错误率。 或者，证明系统针对其计算能力受限的证明人提供安全性保证也是可以接受的。 毕竟，所有计算机实际上都有计算界限。 有时作者使用术语 _参数系统_ 来区分协议和提供计算无界限安全性的的证明系统，以及_参数_，用于非交互性转换产生的文本。
- 必须有令人信服的理由说明为什么验证人不能单纯重新运行计算，因为计算完整性声明主张其完整性。 这是因为证明人能够获得验证人无法获得的信息。
  - 当时间受限时，验证人应该比原生系统的重新执行程序快一个数量级。 这样我们就称实现此属性的系统是 _简洁_ 或 _简洁验证_。
  - 简洁验证需要较短的证明，但是一些证明系统，例如 [Bulletproofs](https://eprint.iacr.org/2017/1066.pdf) or [Aurora](https://eprint.iacr.org/2018/828.pdf) 其有紧凑的证明，但仍然验证依然比较慢。
  - 当验证人无法获取证明人提供的秘密信息时， 并且当证明系统确保此秘密的保密性时，证明系统满足了 _零知识性_ 的要求。 验证人相信计算声明的真实性，但却不了解对计算的部分或全部输入。
- 特别是在零知识证明的背景中，计算完整性要求可能需要稍微有些调整。 在有些情况下，仅仅证明声明的正确性是不够的。 证明人还必须另外证明他 _知道_ 额外的秘密输入， 而且也可以不生成证明而直接输出秘密。[^3] 实现这种更强有力的可靠性概念，即知识健全性的证明系统被称为 _知识(或论点)证明_

SNARK是 _一个非交互式的知识论证_。 该 [论文](https://eprint.iacr.org/2011/443.pdf) 构造了 SNARK 一词，使用 _简洁_ 表示高效验证人的证明系统。 然而，近年来该词的含义被淡化，以包括任何紧凑证明的系统。 本教程将还原其原始定义。

## STARK 概述

“STARK”是一个缩写词，指可扩展透明知识论证。 _可扩展_ 是指以下两件事情同时发生：(1)证明人的运行时间在计算量上最多是准线性的， 与在 SNARK 中允许其非常大的复杂性形成对比，(2) 验证时间是计算量上是多对数级别的。 _透明_ 是指所有验证人消息都是公开的。 尤其是，不需要可信初始化来实例化证明系统，因此不存在密码学有害物。 缩写词表示非交互式 STARKs 是 SNARKs 的一个子类，但实际上该词一般用来指用于可扩展透明的 SNARKs 的 _特定的_ 构造。

这一构造的特殊性质在编译过程中得到最好的说明。 根据不同程度的粒度，人们可能会选择将这个过程分成更多或者更少步骤。 我们把编译过程分为四个阶段和三个转换以介绍 STARKs。 本教程稍后将有更详细的过程和图表。

![The compilation pipeline is divisible into three transformation and four stages.](./../../graphics/pipeline.svg "Overview of the compilation pipeline for SNARKs")

### 计算

整个过程中的输入是 _计算_，您可以将其视为一个程序、一个输入或是另一段输出。 这三种方式都以机器友好的格式存在，例如字节列表。 一般而言，程序由决定机器如何操纵其资源的指示组成。 如果正确的指令列表可以模拟任意的图灵机，那么该机器便是图灵完备的。

在这个教程中，程序被硬编码到机器架构中。 因此，允许的计算空间相当有限。 尽管如此，输入和输出依然是可变的。

计算所需的 _资源_ 可能是 _时间_, _内存_ _随机性_, _秘密信息_ 和 _并行性_. 目标是将计算转变为一种格式，使资源有限的验证人能够验证其完整性。 仍然可以研究更多类型的资源，例如缠纠缠的量子比特、 非确定性的或计算特定的黑盒函数的预言机 。 但由此产生的问题通常是计算复杂性理论，而不是密码学实践问题。

### 算术和算术约束系统

过程中的第一个转换称为 _算术_。 在此过程中，在位串上的基本逻辑和算术运算序列被转换为对有限域元素的原生有限域操作序列，使得两者表示相同的计算。 输出是一个算术约束系统，本质上是一堆方程，其系数和变量从有限域中取值。 计算完备是指_当且仅当_约束系统有一个令人满意的解决方案——这意味着，有一个解使得所有方程都成立。

STARK 证明系统算术化进行了以下运算： 在任何时间点，计算状态都包含在 $\mathsf{w}$ 寄存器的元组中，这些寄存器从有限域 $\mathbb{F}$ 中获取值。 机器定义了一个_状态转换函数_ $f : \mathbb{F}^\mathsf{w} \rightarrow \mathbb{F}^\mathsf{w}$，每个周期更新状态。 _代数执行跟踪 (AET)_ 是按时间顺序排列的所有状态元组的列表。

算术约束系统在代数执行轨迹上定义了至少两种类型的约束：

- _边界约束_: 在计算开始或结束时，指定的寄存器有一个给定的值。
- _转换约束_：任意两个连续的状态元组按照状态转换函数演化。

这些约束统称为_代数中间表示_或_AIR_。 高级 STARKs 可以定义更多的约束类型，以便在一个周期内处理内存或寄存器的一致性。

### 插值和多项式 IOPs

Interpolation in the usual sense means finding a polynomial that passes through a set of data points. In the context of the STARK compilation pipeline, _interpolation_ means finding a representation of the arithmetic constraint system in terms of polynomials. The resulting object is not an arithmetic constraint system but an abstract protocol called a _Polynomial IOP_.

The prover in a regular proof system sends messages to the verifier. But what happens when the verifier is not allowed to read them? Specifically, if the messages from the prover are replaced by oracles, abstract black-box functionalities that the verifier can query in points of his choosing, the protocol is an _interactive oracle proof (IOP)_. When the oracles correspond to polynomials of low degree, it is a _Polynomial IOP_. The intuition is that the honest prover obtains a polynomial constraint system whose equations hold, and that the cheating prover must use a constraint system where at least one equation is false. When polynomials are equal, they are equal everywhere, and in particular in random points of the verifier's choosing. But when polynomials are unequal, they are unequal _almost_ everywhere, and this inequality is exposed with high probability when the verifier probes the left and right hand sides in a random point.

The STARK proof system interpolates the algebraic execution trace literally -- that is to say, it finds $\mathsf{w}$ polynomials $t_i(X)$ such that the values $t_i(X)$ takes on a domain $D$ correspond to the algebraic execution trace of the $i$th register. These polynomials are sent as oracles to the verifier. At this point the AIR constraints give rise to operations on polynomials that send low-degree polynomials to low-degree polynomials only if the constraints are satisfied. The verifier simulates these operations and can thus derive new polynomials whose low degree certifies the satisfiability of the constraint system, and thus the integrity of the computation. In other words, the interpolation step reduces the satisfiability of an arithmetic constraint system to a claim about the low degree of certain polynomials.

### Cryptographic Compilation with FRI

In the real world, polynomial oracles do not exist. The protocol designer who wants to use a Polynomial IOP as an intermediate stage must find a way to commit to a polynomial and then open that polynomial in a point of the verifier's choosing. FRI is a key component of a STARK proof that achieves this task by using Merkle trees of Reed-Solomon Codewords to prove the boundedness of a polynomial's degree.

The Reed-Solomon codeword associated with a polynomial $f(X) \in \mathbb{F}[X]$ is the list of values it takes on a given domain $D \subset \mathbb{F}$. Consider without loss of generality domains $D$ whose cardinality is larger than the maximum allowable degree for polynomials. These values can be put into a Merkle tree, in which case the root represents a commitment to the polynomial. The _Fast Reed-Solomon IOP of Proximity (FRI)_ is a protocol whose prover sends a sequence of Merkle roots corresponding to codewords whose lengths halve in every iteration. The verifier inspects the Merkle trees (specifically: asks the prover to provide the indicated leafs with their authentication paths) of consecutive rounds to test a simple linear relation. For honest provers, the degree of the represented polynomials likewise halves in each round, and is thus much smaller than the length of the codeword. However for malicious provers this degree is one less than the length of the codeword. In the last step, the prover sends a non-trivial codeword corresponding to a constant polynomial.

There is a minor issue the above description does not capture: how does the verifier query a committed polynomial $f(X)$ in a point $z$ that does not belong to the domain? In principle, there is an obvious and straightforward solution: the verifier sends $z$ to the prover, and the prover responds by sending $y=f(z)$. The polynomial $f(X) - y$ has a zero in $X=z$ and so must be divisible by $X-z$. So both prover and verifier have access to a new low degree polynomial, $\frac{f(X) - y}{X-z}$. If the prover was lying about $f(z)=y$, then he is incapable of proving the low degree of $\frac{f(X) - y}{X-z}$, and so his fraud will be exposed in the course of the FRI protocol. This is in fact the exact mechanism that enforces the boundary constraints; a slightly more involved but similar construction enforces the transition constraints. The new polynomials are the result of dividing out known factors, so they will be called _quotients_ and denoted $q_i(X)$.

At this point the Polynomial IOP has been compiled into an interactive concrete proof system. In principle, the protocol could be executed. However, it pays to do one more step of cryptographic compilation: replace the verifier's random coins (AKA. _randomness_) by something pseudorandom -- but deterministic. This is exactly the Fiat-Shamir transform, and the result is the non-interactive proof known as the STARK.

![The STARK proof system revolves around the transformation of low-degree polynomials into new polynomials whose degree boundedness matches with the integrity of the computation.](./../../graphics/stark-overview.svg "Overview of the compilation pipeline for STARKs")

This description glosses over many details. The remainder of this tutorial will explain the construction in more concrete and tangible terms, and will insert more fine-grained components into the diagram.

[0](index) - **1** - [2](basic-tools) - [3](fri) - [4](stark) - [5](rescue-prime) - [6](faster)

[^1]: 另外，代数 _内部_ 表示。
[^2]: Note that FRI is defined in terms of abstract oracles which can be queried in arbitrary locations; a FRI protocol can thus be compiled into a concrete protocol by simulating the oracles with any cryptographic vector commitment scheme. Merkle trees provide this functionality but are not the only cryptographic primitive to do it.
[^3]: Formally, _knowledge_ is defines as follows: an extractor algorithm must exist which has oracle access to a possibly-malicious prover, pretends to be the matching verifier (and in particular reads the messages coming from the prover and sends its own via the same interface), has the power to rewind the possibly-malicious prover to any earlier point in time, runs in polynomial time, and outputs the witness. STARKs have been shown to satisfy this property, see section 5 of the [EthSTARK documentation](https://eprint.iacr.org/2021/582.pdf).
