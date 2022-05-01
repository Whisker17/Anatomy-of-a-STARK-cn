# STARK 详解 第 0 部分：导言

该系列将通过 6 篇文章来剖析 STARK 证明系统。 其受众面是那些掌握基本数学和编程知识的技术向读者。

- 第 0 部分：导言
- [第 1 部分：STARK 概述](概述)
- [第 2 部分：基本工具](基础工具)
- [第 3 部分：FRI](fri)
- [第 4 部分：STARK 多项式 IOP](stark)
- [第 5 部分：一个 Rescue-Prime STARK 系统](rescue-prime)
- [第 6 部分：性能优化](faster)

## 什么是 STARK ？

密码学证明系统领域近来最令人振奋的进展之一便是 STARKs 系统的开发。 它随着区块链行业的迸发逐渐步入大众视野， 一般的证明系统都有一定前提要求：区块链网络通常由 _互不信任的当事方_ 组成，希望进行 _交易_或一般 _更新群体状态_ 依据 _状态进化规则_, 使用 _秘密信息_ 由于参与者相互不信任，他们需要想办法核实各节点提交的交易（或状态更新）的有效性。 _Zk-SNARKs_ 由于其有以下特性，能够保证在此环境中的计算完整性：

- zk-SNARKs（通常是）具有普遍性，即它们能够证明任意计算的完整性。
- zk-SNARKs 是非交互式证明系统，意味着整个完整性证明包含一个单一信息；
- zk-SNARKs 是可有效验证的，这意味着验证人的工作量比单纯重新计算一遍的工作量少一个数量级；
- zk-SNARKs 是零知识的，这意味着它们不泄露任何关于为计算的秘密输入信息。

![Vitalik Buterin 喜欢 SNARKs](./../../graphics/twitter-vitalik.png "预计 zk-SNARKs 将是一场重大革命。")

zk-SNARKs 距离开发出来已经过去了一段时间，但 STARK 系统还处在一个比较早期的阶段。 其脱颖而出主要有以下几个原因：

- 传统的 zk-SNARK 依靠尖端密码学难题和假设，而 STARK 证明系统中唯一的密码学集成是防碰撞散列函数。 因此该证明系统是在一个理想化的散列函数 <sup id="fnref:1"><a href="#fn:1" class="footnote-ref"></a></sup> 的情况中是抗量子计算的。 这与第一代 SNARKs 的情况形成鲜明对比，后者使用双线映射，只能在不可伪造的假设下证明是安全的。
- STARKs 的算法领域独立于密码学难题，因此可以具体选择这个领域以优化性能。 因此 STARKs 保证了更快的证明。
- 传统的 zk-SNARKs 依靠可信初始化来产生公共参数。 可信初始化结束后，其使用的随机数都必须销毁。 我们必须这样做是因为如果参与者拒绝或忽略删除这种破坏密码学的行为，他们将保留伪造证明的能力。 相比之下，STARKs 不需要任何可信初始化，因此没有这样的问题。

![Eli Ben-Sasson 更喜欢 STARKs。](./../../graphics/twitter-eli.png "STARKs 将击败 SNARKs。")

在这个教程中，我试图解释 STARKs 证明系统的各部分工作原理。 该教程还支持一个 python 实现的支持，用于证明和验证基于 [Rescue-Prime](https://eprint.iacr.org/2020/1143.pdf) 散列函数的简单计算。 阅读或研究本教程后，您应该能够为自己的计算问题编写零知识 STARKs 证明人和验证人。

## 为什么要写这篇教程？

提前声明，目前有各种各样的资料来源可以了解 STARKs。 这里是一个不完整的列表。

- 关于 [FRI](https://eccc.weizmann.ac.il/report/2017/134/revision/1/download/), [STARK](https://eprint.iacr.org/2018/046.pdf), [DEEP-FRI](https://eprint.iacr.org/2019/336.pdf), 以及最新 [FRI 可靠性分析](https://eccc.weizmann.ac.il/report/2020/083/)
- Vitalik Buterin 的三篇教程 (第 [I](https://vitalik.ca/general/2017/11/09/starks_part_1.html)/[II](https://vitalik.ca/general/2017/11/22/starks_part_2.html)/[](https://vitalik.ca/general/2018/07/21/starks_part_3.html) 部分)
- StarkWare的一系列博客文章 (parts [1](https://medium.com/starkware/stark-math-the-journey-begins-51bd2b063c71), [2](https://medium.com/starkware/arithmetization-i-15c046390862), [3](https://medium.com/starkware/arithmetization-ii-403c3b3f4355), [](https://medium.com/starkware/low-degree-testing-f7614f5172db), [5](https://medium.com/starkware/a-framework-for-efficient-starks-19608ba06fbe)
- StarkWare 推出的 [STARK @ Home](https://www.youtube.com/playlist?list=PLcIyXLwiPilUFGw7r2uyWerOkbx4GFMXq) 视频
- StarkWare 推出的 [STARK 101](https://starkware.co/developers-community/stark101-onlinecourse/) 在线课程
- StarkWare 推出的 [EthStark 文档](https://eprint.iacr.org/2021/582.pdf)
- 总的来说 [StarkWare](https://starkware.co) 所做的任何工作都值得一看

有了这些资源，那我为什么要写另一个教程？

_因为这些教程都比较浅显。_ 它们很好地从高层次解释了这些技术如何工作，并告诉大家它能够发挥作用。 然而，它们并没有描述一个可供开发的完整系统。 例如，没有哪个教程能够说明如何实现零知识， 如何批处理各种底层证明，或如何确定由此产生的安全等级。 EthSTARK 这个教程确实提供了一个完整的参考资料，以回答其中的大部分问题。 但它是根据一种特定的计算而设计的，不包括零知识，也不强调直观的解释。

_而相关论文又显得晦涩难懂。_ 令人遗憾的是，科学出版的激励措施使学术论文难以被外行人读懂。 因此我们需要这样一个教程，使更多的读者能够读懂这些论文。

_另外部分文档内容有些过时了_ 各种教程中描述的许多技术已经得到改 例如， EthSTARK 文档(上面提到的里面最新的文档)描述了 _DEEP 插入技术_ ，以便将多项式的正确求值声明减少为有界多项式的正确求值声明。 而教程中没有提到这种方法，因为当时还没有。

_我更加倾向于自己的风格。_ 我不喜欢许多符号和名字，我希望人们会使用正确的符号和名字。 特别是，我喜欢把多项式作为证明系统中最基本的对象。 但恰恰相反， 所有其他教程都根据在 Reed-Solomon 编码 [^2] 操作描述了证明系统的机制。

_这有助于我理解事物。_ 写此教程帮助我系统地掌握我自己的知识，自检我所掌握的程度。

## 所需的背景知识

本教程在需要时也会将背景材料再讲述一遍。 然而，读者也可以选择研究以下主题，因为如果他们不熟悉它们，该教程的内容对他们来说可能太信息量太大了。

- 有限域及其扩展域
- 有限域上的多项式，包括一元域和多元域
- 快速傅里叶变换
- 散列函数

## 路线图

- [第 1 部分：STARK 概述](overview) 从上层了解其概念和工作流程。
- [第 2 部分：基本工具](basic-tools) 介绍了基本的数学和加密工具，作为证明系统的基础。
- [第 3 部分：FRI](fri) 涵盖底层测试，这是证明系统的密码学核心。
- [第 4 部分：STARK 多项式 IOP](stark) 解释了从任意计算声明到生成抽象证明系统的信息理论。
- [第 5 部分：Rescue-Prime STARK](rescue-prime) 使用上述工具，并为简单的计算建立一个透明的零知识证明系统。
- [第 6 部分：性能优化](faster) 介绍了更快的算法和技术进行优化，有效地将“S”放入 STARK。

## Python 代码支持

除了文本中包含的代码片段外，还有完整可行的 python 实现。 在 [这里](https://github.com/aszepieniec/stark-anatomy) 下载资源库。 顺便说一句，如果你发现了任何错误， 或者如果你想建议一个改进，请随时提出 PR 请求。

## 问题和讨论

问题和讨论的最佳场所是在 [社区论坛上的零知识播客](https://community.zeroknowledge.fm)。

## 致谢

感谢 Bobbin Threadbare、Thorkil Værge 和 Eli Ben-Sasson 的有益反馈和评论。 以及 [Nervos](https://nervos.org) 基金会财政资助。 发送电子邮件到 `alan@nervos.org` 或在 twitter 或 Github 上关注 `aszepieniec` 。 捐赠可以通过 [btc](bitcoin:bc1qg32wme6sqltus5e9yzuq4y56xxc0rutly8ak7y), [ckb](nervos:ckb1qyq9s4rvld206a3rl6jmzxav4ffx58uj5prsv867ml) 或 [eth](ethereum:0x934B24cE32ceEDB38ce088Da1D9366Fa23F7B3f4)。

## 镜像

本教程托管在几个地方。 如果你也是一个相同的副本或翻译，请联系我。

- [GitHub Pages](https://aszepieniec.github.io/stark-anatomy/)
- [Neptune 项目网站](https://neptune.cash/learn/stark-anatomy/)

**0** - [1](overview) - [2](basic-tools) - [3](fri) - [4](stark) - [5](rescue-prime) - [6](faster)

[^1]: 在文献中，这种理想化被称为量子随机预言机模型。
[^2]: Reed-Solomon 编码是一个低次多项式在给定的点域上求值的向量。 当定义多项式不同而求值域相同时，不同编码属于同一码
