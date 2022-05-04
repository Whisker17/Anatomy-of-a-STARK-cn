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
  - When the verifier has no access to secret information that is available to the prover, and when the proof system protects the confidentiality of this secret, the proof system satisfies _zero-knowledge_. The verifier is convinced of the truth of a computational claim while learning no information about some or all of the inputs to that computation.
- Especially in the context of zero-knowledge proof systems, the computational integrity claim may need a subtle amendment. In some contexts it is not enough to prove the correctness of a claim, but the prover must additionally prove that he _knows_ the secret additional input, and could as well have outputted the secret directly instead of producing the proof.[^3] Proof systems that achieve this stronger notion of soundness called knowledge-soundness are called _proofs (or arguments) of knowledge_.

A SNARK is a _Succinct Non-interactive ARgument of Knowledge_. The [paper](https://eprint.iacr.org/2011/443.pdf) that coined the term SNARK used _succinct_ to denote proof system with efficient verifiers. However, in recent years the meaning of the term has been diluted to include any system whose proofs are compact. This tutorial takes the side of the original definition.

## STARK Overview

The acronym STARK stands for Scalable Transparent ARgument of Knowledge. _Scalable_ refers to the fact that that two things occur simultaneously: (1) the prover has a running time that is at most quasilinear in the size of the computation, in contrast to SNARKs where the prover is allowed to have a prohibitively expensive complexity, and (2) verification time is poly-logarithmic in the size of the computation. _Transparent_ refers to the fact that all verifier messages are just publicly sampled random coins. In particular, no trusted setup procedure is needed to instantiate the proof system, and hence there is no cryptographic toxic waste. The acronym's denotation suggests that non-interactive STARKs are a subclass of SNARKs, and indeed they are, but the term is generally used to refer to a _specific_ construction for scalable transparent SNARKs.

The particular qualities of this construction are best illustrated in the context of the compilation pipeline. Depending on the level of granularity, one might opt to subdivide this process into more or fewer steps. For the purpose of introducing STARKs, the compilation pipeline is divided into four stages and three transformations. Later on in this tutorial there will be a much more fine-grained pipeline and diagram.

![The compilation pipeline is divisible into three transformation and four stages.](./../../graphics/pipeline.svg "Overview of the compilation pipeline for SNARKs")

### Computation

The input to the entire pipeline is a _computation_, which you can think of as a program, an input, and an output. All three are provided in a machine-friendly format, such as a list of bytes. In general, the program consists of instructions that determine how a machine manipulates its resources. If the right list of instructions can simulate an arbitrary Turing machine, then the machine architecture is Turing-complete.

In this tutorial the program is hardcoded into the machine architecture. As a result, the space of allowable computations is rather limited. Nevertheless, the inputs and outputs remain variable.

The _resources_ that a computation requires could be _time_, _memory_, _randomness_, _secret information_, _parallelism_. The goal is to transform the computation into a format that enables resource-constrained verifier to verify its integrity. It is possible to study more types of resources still, such as entangled qubits, non-determinism, or oracles that compute a given black box function, but the resulting questions are typically the subject of computational complexity theory rather than cryptographical practice.

### Arithmetization and Arithmetic Constraint System

The first transformation in the pipeline is known as _arithmetization_. In this procedure, the sequence of elementary logical and arithmetical operations on strings of bits is transformed into a sequence of native finite field operations on finite field elements, such that the two represent the same computation. The output is an arithmetic constraint system, essentially a bunch of equations with coefficients and variables taking values from the finite field. The computation is integral _if and only if_ the constraint system has a satisfying solution -- meaning, a single assignment to the variables such that all the equations hold.

The STARK proof system arithmetizes a computation as follows. At any point in time, the state of the computation is contained in a tuple of $\mathsf{w}$ registers that take values from the finite field $\mathbb{F}$. The machine defines a _state transition function_ $f : \mathbb{F}^\mathsf{w} \rightarrow \mathbb{F}^\mathsf{w}$ that updates the state every cycle. The _algebraic execution trace (AET)_ is the list of all state tuples in chronological order.

The arithmetic constraint system defines at least two types of constraints on the algebraic execution trace:

- _Boundary constraints_: at the start or at the end of the computation an indicated register has a given value.
- _Transition constraints_: any two consecutive state tuples evolved in accordance with the state transition function.

Collectively, these constraints are known as the _algebraic intermediate representation_, or _AIR_. Advanced STARKs may define more constraint types in order to deal with memory or with consistency of registers within one cycle.

### Interpolation and Polynomial IOPs

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
