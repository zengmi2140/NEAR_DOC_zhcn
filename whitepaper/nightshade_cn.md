

# <center>夜影：Near协议中的分片设计</center>

---
原版参见 [https://nearprotocol.com/downloads/Nightshade.pdf](https://nearprotocol.com/downloads/Nightshade.pdf)

---


**词汇表**

| **词汇**   | **释义**   | **备注**   | 
|:----|:----|:----|
| Nightshade   | 夜影   |    | 
| shard   | 分片   |    | 
| validator   | 验证人   |    | 
| partition   | 分区   | 来自oracle等主流数据库的概念   | 
| beacon chain   | 信标链   |    | 
| Ethereum Serenity   | 以太坊宁静   |    | 
| finalize   | 确定   |    | 
| fisherman   | 渔夫   |    | 
| erasure codes   | 纠删码   |    | 
| chunk   | 段   |    | 
| block producer   | 出块人   |    | 
| epoch   | 纪元   |    | 
| chunk producer   | 出段人   |    | 

---
目录

- [引言](#引言)
- [1 分片基础](#1-分片基础)
    - [1.1 验证人分区和信标链](#11-验证人分区和信标链)
    - [1.2 平方分片](#12-平方分片)
    - [1.3 状态分片](#13-状态分片)
    - [1.4 跨分片交易](#14-跨分片交易)
    - [1.5 恶意行为](#15-恶意行为)
        - [1.5.1 恶意分叉](#151-恶意分叉)
        - [1.5.2 批准无效块](#152-批准无效块)
- [2 状态有效性和数据可用性](#2-状态有效性和数据可用性)
    - [2.1 验证人轮转](#21-验证人轮转)
    - [2.2 状态有效性](#22-状态有效性)
    - [2.3 渔夫](#23-渔夫)
    - [2.4 简洁的非交互知识论证](#24-简洁的非交互知识论证)
    - [2.5 数据可用性](#25-数据可用性)
        - [2.5.1 监管权证明](#251-监管权证明)
- [参考文献](#参考文献)



---
# 引言

众所周知，以太坊作为迄今为止最广为使用的通用区块链，主网每秒只能处理不到20笔交易。这个局限性，加上该网络的流行度，导致了高额的gas价格（gas是网络上执行交易的开销）和漫长的确认时间。尽管在撰写该文的时候，以太坊平均每10到20秒产出一个新块，然而根据ETH Gas监测站的数据，一笔交易实际上要花1.2分钟才能加到链上。低产量，高价格和高延迟，这些都导致以太坊不适合运行那些要大规模使用的服务。

以太坊低产量的主要原因是，网络中的每个节点都要处理全网的每一笔交易。开发者们提出了许多解决方案，试图在协议层面解决这个问题。这些方案主要分为两类：一类是把所有的计算工作交给数量有限的强力节点；另一类是让网络中的每个节点只做所有计算工作的一部分。前一种方案的例子是Solana（[https://solana.com/](https://solana.com/)），系统中的每个节点通过细致的低层优化和GPU的使用，可以支撑每秒几十万笔简单的支付交易。Algorand（[https://algorand.com/](https://algorand.com/)），SpaceMesh（[https://spacemesh.io/](https://spacemesh.io/)），Thunder（[https://thundercore.com/](https://thundercore.com/)）都归入这一类，他们通过改进共识以及区块结构本身来超过以太坊的TPS，但是仍然受制于单台机器（尽管非常强大）的处理能力。

后一种方案，把工作分给所有的参与节点，叫做分片。这也是当前以太坊基金会打算扩展以太坊的方式。撰写本文时，以太坊的分片规范还未定稿，最新的规范可以通过该链接查看：[https://github.com/ethereum/eth2.0-specs](https://github.com/ethereum/eth2.0-specs.)。

Near协议也是基于分片的。Near团队的成员包括：若干前MemSQL工程师，负责构建分片、跨分片交易以及分布式的JOIN连接；5位前谷歌员工，他们在构建分布式系统方面有着丰富的行业经验。

本文重点阐述了区块链分片的通用方法，以及需要解决的主要问题，包括状态有效性和数据可用性问题。本文还提出了夜影方案，Near协议基于该方案进行打造，解决了上述问题。

# 1 分片基础

让我们从最简单的分片方法开始。在这个方法中，不同于运行一个链，我们运行多个链，每个链称为一个“分片”。每个分片有自己的验证人集合。这里以及下文，我们都使用这个原生词汇“验证人”，指代那些进行交易验证和生产区块的参与者，不管是POW中挖矿的方式，还是投票机制。现在，让我们先假设分片之间互相不通信。

这种设计，尽管简单，却足够显现分片中初始面临的一些主要挑战了。

## 1.1 验证人分区和信标链

假设系统包含10个分片。第一个挑战是，由于每个分片有自己独立的验证人，现在每个分片的安全性都只有原先整条链的十分之一了。如果一个包含X位的非分片链决定硬分叉成一个分片的链，X位验证人就会分布到10个分片上，每个分片现在仅有 X/10 位验证人。攻击一个分片只需要买通5.1%（51% / 10）的验证人（见图1），

[//]: ![图片](https://uploader.shimo.im/f/nPDuo7aobUsNVIps.png!thumbnail)

![图片](https://github.com/marco-sundsk/NEAR_DOC_zhcn/blob/master/static/images/Nightshade/shards1.png?raw=false)

**<center>图 1： 分片之间分割验证人</center>**

</br>
这就带来第二个问题：谁负责为每个分片选择验证人？5.1%的验证人被攻破，也仅仅在这些验证人正好处于同一个分片时才有威胁。如果验证人不能自主选择去验证哪个分片，那么一个控制了5.1%验证人的参与者是很难让他控制的所有验证人都在同一个分片的。这就大大降低了他破坏系统的能力。

当今几乎所有的分片系统都依赖某种随机源来给分片分配验证人。区块链上的随机性本身就是一个极富挑战性的话题，它超出了本文讨论的范畴。现在，让我们假设有这样一个可用的随机源。我们会在2.1节介绍验证人分配的更多细节。


随机性和验证人分配都需要独立于任何特定分片的计算量。对于这种计算，几乎所有现在的设计都使用一个独立的区块链，执行维护整个网络所需要的操作。除了随机数生成和给分片分配验证人外，这些操作通常还包括接收分片的更新以及执行分片的快照，处理POS系统中的stake和slashing，以及在支持平衡功能的分片系统上进行分片的再平衡。这种独立链的名称，以太坊上叫信标链，波卡上叫中继链，Cosmos上叫Cosmos Hub。

本文统一将其称为信标链。信标链的存在引出了下面一个有趣的主题，平方分片。

## 1.2 平方分片

分片经常被视为一种可以随着网络中节点数的增加而无限扩展的方案。实际上只在理论上有可能设计出这种方案，任何使用信标链概念的方案都不可能无限扩展。为了理解其原因，要意识到，信标链要做一些记账的计算工作，例如给分片分配验证人，或快照分片链区块，这些都与系统中的分片数量成比例关系。既然信标链本身只是一条单独的链，计算量受限于运行节点的算力，因此分片数量自然就受到了限制。

然而，分片网络的结构确实对任意节点的改进产生了倍增的效果。想一下这样一个场景，网络中节点效率作出任意的改进，使得节点处理交易的时间缩短。

如果这些改进了的节点运作着网络，包括信标链中的节点，都提高了4倍的速度。那么，每个分片都可以处理原先交易量的四倍多，而信标链可以维持原先分片数量的四倍多。整个系统产量的提升结果就是：4 × 4 = 16 倍，这就是平方分片名称的由来。

当前，对于究竟能支持多少分片，很难提供精确的度量。但是，在可以预见的未来，区块链用户的交易量需求不太可能超过平方分片的能力。在这么大量的分片中要安全运行如此多的节点数目，可能比当今所有区块链节点数的和还要高出一个数量级。

## 1.3 状态分片

直到这里，我们还未准确定义当网络切割成分片后，究竟什么被分割开，什么没有被分割出去。特别的，区块链中的节点会执行三个重要任务：他们不仅要1）执行交易，还2）把经过验证的交易和构建完成的区块中继给其他节点，并且3）存储整个网络账本的状态和历史。三个任务中的每一个都对运行网络的节点提出了不断增长的需求：

1. 随着需要处理的交易量的增加，执行交易的必要性导致更高的算力需求；
2. 随着需要中继的交易量的增加，中继交易和区块的必要性导致更高的网络带宽需求；
3. 随着状态数据的增长，存储数据的必要性导致更大的存储需求。重要的是，不像算力和网络需求，存储要求是持续增长的，即使交易速率（TPS）保持恒定。

上面的列表可能让人觉得，存储需求是最急迫的，因为它是唯一一个随着时间持续增长的因素（即使每秒交易量不变）。但实际上，当前最急迫的需求是算力。撰写本文时，以太坊的全网状态大小是100GB，大部分的节点都可以轻松应对。而以太坊的每秒可处理交易量在20笔左右，比许多实际应用场景的需求低了好几个数量级。

Zilliqa（[https://zilliqa.com/](https://zilliqa.com/)）是最广为人知的对处理而不是存储进行分片的项目。对处理的分片相对容易：因为每个节点都有完整的状态，意味着合约可以自由调用其他合约，从链上读取任何数据。为了确保多个分片对状态同一部分的更新不会产生冲突，还需要一些细致的技术工作。从这方面来说，Zilliqa采用了相对简略的实现。

尽管之前有提出过分片存储而非分片处理的方案，但它是极不寻常的情况。在实践中，对存储的分片，也就是状态分片，几乎总是隐含了对处理的分片和网络的分片。

实际上，状态分片中，每个分片中的节点都在构建他们自己分片的区块链，这条链仅包含影响到属于这个分片的状态的那些交易。因此，分片的验证人只需要存储全局状态的本地（分片）部分，也仅仅执行和中继那些影响到这些本地状态的交易。这种分区线性减少了所有包括算力、存储、带宽的需求。但是这也带来了新的问题，例如数据可用性和跨分片交易的问题。这两个问题我们会在下文阐述。

## 1.4 跨分片交易

迄今为止，我们描述的分片模型还不是很有用。因为，如果单个分片不能跟其他分片互相通信，整个系统不过就是多个独立的区块链而已。甚至分片技术还不可用的今天，不同区块链之间的互通依然是一个巨大的需求。

现在，让我们只考虑简单支付交易，每个参与者仅在一个分片上拥有账户。如果有用户希望在同一分片内从一个账户转账到另一账户，这笔交易可以由本分片的验证人全权处理。然而，如果分片1上的爱丽丝想转账给分片2上的鲍勃，无论是分片1上的验证人（他们无法把账存入鲍勃的账户），还是分片2上的验证人（他们无法把账从爱丽丝的账户取出），都不能处理整笔交易。

有两种跨分片交易的实现方案：

* **同步：** 无论何时，一笔跨分片交易需要执行时，所涉及分片上含有相关交易状态转移信息的区块都同时生成。这些分片的验证人合作执行此类交易。

* **异步：** 影响多个分片的跨分片交易在其所涉及的分片上异步执行。贷记方的分片一旦有了充足的证据，表明借记方分片已经执行了借记那边的工作，就可以执行自己这边的贷记工作。这种方案更趋于流行，因为它比较简单，并且易于协同。这种系统应用于当今的Cosmos、以太坊宁静、Near、Kadena等其他项目中。这种方案有一个问题，如果区块是独立产生的，相关区块中的某个成为孤儿块的可能性总归存在，导致整个交易仅被部分执行。想一下图2中描述的情况：两个分片都遭遇分叉。同时，一笔跨分片交易相应记录于块 **A** 和块 **X'** 中。如果链 **A-B** 和 链 **V'-X'-Y'-Z'** 成为了各自分片的正统链，则跨分片交易完全确认。若 **A'-B'-C'-D'** 和 **V-X** 成为正统链，则跨分片交易完全被废弃，这也是可以接受的。但如果 **A-B** 和 **V-X** 成为正统链，则跨分片交易的一部分确认，另一部分被废弃，这会导致一个原子性错误。我们在第二部分中提出的协议里，会介绍该问题的解决办法。这部分会介绍我们就分片协议提出的分叉选择规则以及共识算法的变动。

![图片](https://uploader.shimo.im/f/zmfZjjqQgBEK9sJw.png!thumbnail)

**<center>图&nbsp;2：异步跨分片交易</center>**

值得注意的是，跨链通信不止在分片区块链有用。链间的互操作性是个复杂的问题，许多项目想要解决这个问题。在分片区块链中，该问题相对简单些，因为区块结构和共识在分片之间都是相似的，还有一个信标链可以用来做协调。不同于分片区块链中的所有分片都是相似的，全局的区块链生态体系中有许多不同的链，他们有不同的目标使用场景、不同的去中心化程度和隐私保证程度。

构建这样一个系统，包含一个公共的信标链，和一系列虽然拥有不同属性、但有一系列足够类似的共识和区块结构的链，由此打造一个有可用互操作子系统的异构区块链生态系统。这样的系统不太可能有验证人轮转的功能，所以需要采取一些额外措施确保安全性。Cosmos（[https://cosmos.network/](https://cosmos.network/)）和波卡（[https://polkadot.network/](https://polkadot.network/)）实际上都属于这种系统。

## 1.5 恶意行为

这一节，我们将回顾，如果恶意验证人想破坏一个分片，会采用什么样的敌对行为。我们将在2.1节回顾那些防止分片破坏的经典实现方法。

### 1.5.1 恶意分叉

一部分恶意验证人也许试图创建一个分叉。需要注意的是，无论底层共识是否是BFT，买通足够数量的验证人就都可能创建一个分叉。

相对来说，最可能发生的是买通一个分片中的50%以上（的验证人），而不是买通整个链的50%以上（的验证人）（我们将在2.1节进一步阐述这些可能性）。正如1.4节中所讨论的，跨分片交易牵涉到多个分片中的状态转换，那些分片中相应应用这些状态变化的区块要么全部确定（也就是说，出现在对应分片被选中的链分支上），要么全部成为孤儿块（也就是说，不出现在对应分片被选中的链分支上）。因为一般意义上，分片被破坏的可能性是不可忽视的，即使分片验证人之间已经达成了某种拜占庭共识或是包含状态变化的块之上已经产出许多区块，我们也不能假设不会发生分叉。

这个问题有多种解决方案，最常用的一种是，将分片链上最新的区块间或交叉连接到信标链上。分片链上的分叉选择规则相应变化为总是优选那些交叉连接的链，并且只对最后一个交叉连接之后产生的区块使用分片特定的分叉选择规则。

### 1.5.2 批准无效块

一些验证人也许会尝试创建一个区块，里面应用了不正确的状态转换函数。例如，从爱丽丝有10枚代币、鲍勃有0枚代币的状态开始，区块可能包含包含一笔爱丽丝转给鲍勃10枚代币的交易，但是最终状态如图3所示，爱丽丝有0枚代币，鲍勃却获得了1000枚。

![图片](https://uploader.shimo.im/f/SpzPbl5rP3oe0dYx.png!thumbnail)

**<center>图 3： 无效区块的一个例子</center>**


在一个经典的非分片区块链中，这种攻击是不可能成功的，因为网络中所有参与者都会验证所有的区块。一个带有无效状态转换的区块，其他出块人和不出块的网络参与者都会拒绝。即使恶意验证人继续在这个无效区块之上以相对诚实验证人更快的速度堆积新的区块，使得包含无效区块的链更长，也没有关系。因为，不管出于何种目的使用这个区块链的参与者都会验证所有区块，并丢弃那些构建在无效区块之上的所有区块。

图4中有5个验证人，其中三个是作恶者。他们创建了一个无效区块**A'**，并继续在其上创建新的区块。两个诚实验证人丢弃无效的**A'**，并在最后一个在他们看来有效的区块上构建区块，导致了一个分叉。因为诚实验证人的分叉中验证人更少，所以他们的链更短一些。然而，在经典的非分片区块链中，每个出于各种目的使用区块链的参与者都负责验证所收到的所有区块，并重新计算状态。因此，任何关注该区块链的人都能发现**A'**是无效的，并立即丢弃**B'**、**C'** 和**D'**。这样一来，链**A-B**就是当前最长的有效链了。

![图片](https://uploader.shimo.im/f/KUEOaJxwMjQyFP0Y.png!thumbnail)

**<center>图 4： 试图在非分片区块链中构建无效区块</center>**


但是，分片区块链中， 任何参与者都无法验证所有分片上的全部交易。所以它们需要某种方法确认，链上任何分片的任何历史状态中，都不存在无效的区块。

需要注意的是，与分叉不同，交叉连接到信标链并不是该问题的一个充分的解决方案，因为信标链无法验证所有区块。它只能验证该区块获得了分片中足够数量的验证人签名（并以此作为其正确性的证据）。

我们将在下面的2.2节讨论该问题的解决方案。

# 2 状态有效性和数据可用性

分片区块链的核心思想是大部分操作和使用网络的参与者无法验证所有分片的区块。因此，任何时候，参与者需要与某个特定的分片互动时，他们一般都无法下载和验证该分片的完整历史状态。

然而，分片的分区属性，引起了一个显著的潜在问题：如果不下载和验证特定分片的完整历史状态，参与者无法确信他们操作的状态是某个有效区块序列的结果，而且也不能确信这个序列就是该分片的正统链。这个问题在非分片区块链上是不存在的。

首先，我们会呈现解决该问题的一个简单方案，许多协议中也提出过这个方案。然后分析如何破解该方案，以及为了防止破解进行了哪些尝试。

## 2.1 验证人轮转

解决状态有效性一个比较直接的方案如图5所示：假设整个系统有数千位验证人，其中不超过20%的是作恶者或会失效（比如掉线无法出块）。然后，如果我们取出200个验证人作为样本，那么样本中超过1/3的验证人会失效的概率在实际中可以视为0。

![图片](https://uploader.shimo.im/f/D4IQwvPks24L2Cxc.png!thumbnail)

**<center>图 5： 验证人取样</center>**


1/3是个很重要的阈值。有一个共识协议族，叫做BFT共识协议，可以保证只要少于1/3的参与者失效，不管是宕机或以某种方式违反了协议，整个系统还是能达成共识。


在这种诚实验证人百分比的假设下，若某分片的当前验证人集合提供了某个块，直接的方案就认为该块是有效的，且构建在这些验证人开始验证工作时就认定的该分片的正统链上。这些验证人从之前的验证人集合中识别了正统链，而根据同样的假设，这些验证人构建的区块就在正统链的前面。这样推导下去，整个链都是有效的。而且，既然任何时间节点的验证人集合都没有产生分叉，这种直接的方案也确信当前链就是该分片的唯一链。观察图6可获得一个直观感受。

![图片](https://uploader.shimo.im/f/KNykZI7OgqQFg04k.png!thumbnail)

**<center>图 6：每个块都通过BFT共识确定的区块链</center>**


如果我们假设验证人受到自适应攻陷，这个简单的方案就失效了。这个假设并非不合情理。自适应攻陷拥有1000个分片的系统中的一个分片，其成本远低于去收买整个系统。因此，协议的安全性随着分片数量的增加而线性降低。为了确信一个块的有效性，我们必须知道系统的所有分片在任何历史时间点上，其大多数验证人都没有勾结在一起。然而这种自适应破坏，我们不再可以确信块的有效性。正如在1.5节中讨论的，被收买的验证人可以实行两种基础的破坏行为：创建分叉，以及创建无效区块。

恶意分叉可以通过向信标链交叉连接区块的方式解决。信标链通常在设计上就比分片链有明显更高的安全性。然而，创建无效区块显然是个解决起来更有挑战性的问题。

## 2.2 状态有效性

看一下图7的情况：分片1被攻破了，恶意行为人产生了无效的区块**B**。假设在这个区块**B**里，爱丽丝的账户上凭空铸造了1000枚代币。恶意行为人接着在B后面创建了有效区块**C**（意思是**C**中的交易正确执行），掩盖了无效的区块**B**，并发起了一个到分片2的跨分片交易，内容是转账那1000枚代币给鲍勃的账户。此时，非法生成的代币反而驻留在分片2中一个完全有效的区块里。

![图片](https://uploader.shimo.im/f/fCPT6bIum9QN31Bm.png!thumbnail)

**<center>图 7： 来自一个无效区块的跨分片交易</center>**

以下是应对该问题的一些简单方法：

1. 分片2的验证人（跨分片）验证产生交易的那个区块。这个方法在上图的场景中都不起作用，因为区块**C**看起来是完全有效的。
2. 分片2的验证人（跨分片）验证交易所在区块之前的一大批前置区块。自然的，对于接收分片验证的任何区块个数**N**，恶意验证人都可以在产出的无效区块上创建**N+1**个有效区块来应对。

解决该问题的一个可行的方法是：把分片排列成无向图，每个分片都与若干其他分片相连。并且只允许这些相邻分片之间的跨分片交易（例子有，这就是Vlad Zamfir的[分片精要](https://medium.com/nearprotocol/37e538177ed9)的做法，而且在Kadena的Chainweb中也用了相似的思想[1]）。如果非相邻的分片需要一个跨分片交易，多个（相邻）分片路由的方式可以解决。在这个设计中，每个分片中的验证人都要验证自己分片和相邻分片的所有区块。下图有10个分片，每个都有4个相邻分片，对于跨分片交易的两个分片之间最多仅需要两跳（见图8）。

![图片](https://uploader.shimo.im/f/Q5jvS3K1NzcaOp6s.png!thumbnail)

**<center>图 8：一个跨分片的无效交易会在类chainweb系统里被检测到</center>**


分片2不仅验证自己的区块，还验证所有相邻分片的，包括分片1。如果一个分片1上的恶意行为人试图创建一个无效区块**B**，在其上创建区块**C**并发起跨分片交易，则这样的交易无法成功。因为分片2会验证分片1的完整历史，由此识别出无效的区块**B**。

尽管只攻破单个分片不再是可行的攻击方法，攻破若干分片仍然存在问题。在图9中，一个对手同时攻破了分片1和2，使用来自无效区块B的资金，执行了到分片3的跨分片交易：

![图片](https://uploader.shimo.im/f/S8UGFTXXpNQHcmFB.png!thumbnail)

**<center>图 9：一个在类chainweb系统中检测不到的无效跨分片交易</center>**


分片3验证了分片2的所有区块，但是并不验证分片1的区块，因此无法检测到那个有害区块。

正确解决状态有效性问题有两个主要方向：渔夫与密码学计算证明。

## 2.3 渔夫

第一种实现方法背后的思想是：一旦一个区块头因为某种目的（比如交叉连接到信标链或跨分片交易）在链间传递，就有一个时间段让诚实的验证人提交那个区块无效的证明。存在各种各样的设计，允许通过简洁的证明指认无效区块。对于接收节点来说，区块头的通信远比要接收整个区块小多了。

通过这种方法，只要分片中至少有一个诚实验证人，系统就是安全的。

![图片](https://uploader.shimo.im/f/OdSA4jHDfrYn9RJq.png!thumbnail)

**<center>图 10： 渔夫</center>**

这是当前提出的协议中主流的实现方法（除了那些装作不存在这种问题的协议）。然而，这种实现有两大主要缺陷：

1. 挑战时间段要足够长，让诚实验证人可以识别产生的区块、下载并完全校验区块，以及当区块无效时，做好挑战的准备工作。引入这么一种时间段会明显拖慢跨分片交易。
2. 挑战机制的存在导致了一个新的攻击维度，作恶者用无效的挑战冲击系统。该问题的一个直接解决方案是让挑战者抵押一定量的代币，挑战成功才归还。但这也只是部分解决了问题，因为有可能对手（在抵押被没收的情况下）仍然能通过无效挑战获利。例如，以此阻止诚实验证人的挑战获得通过。这种攻击被称为**悲痛攻击**。

处理后面这个问题的方法，参见3.7.2节。

## 2.4 简洁的非交互知识论证

第二种应对多分片贿赂的方案是用某种密码学的措施，允许一个人证明某个计算（比如从一系列交易中算出区块）被正确执行了。这种措施确实存在，比如 zk-SNARKs、zk-STARKs 等技术， 一些技术经常用在当今私人支付领域的区块链协议中，其中最知名的是ZCash（[https://z.cash/](https://z.cash/)）。那些原语的主要问题是计算起来非常慢。例如，Coda协议（[https://codaprotocol.com/](https://codaprotocol.com/)），专门使用zk-SNARKs证明链上的所有区块都是有效的，在一次访谈中表示，为了创建一个证明，每笔交易要花费30秒时间（目前这个数字可能要小些）。

有趣的是，证明未必由一个可信实体进行计算。因为证明不仅能证实它的目标计算的有效性，还能证实其自身的有效性。因此，这种证明的计算工作可以分割给一系列参与者，与必须执行某些无需信任的计算相比，显著降低了冗余。它也允许计算zk-SNARKs的参与者运行在特殊的硬件上，同时不会减低系统的去中心化程度。

除了性能外，zk-SNARKs面临的挑战还有：

1. 依赖于鲜有研究和缺少时间验证的密码学原语；
2. “有害废弃物”-- zk-SNARKs依赖于一个可信的设置过程，其中一组人执行某种计算，并丢弃中间的计算结果。如果该过程的所有参与者都被买通，保留了中间结果，则就能创建虚假证明；
3. 系统设计引入额外的复杂性；
4. zk-SNARKs只适用于所有可能计算的一个子集。所以，支持图灵完备的智能合约语言的协议将不能使用SNARKs证明链的有效性。
## 2.5 数据可用性

我们遇到的第二个问题是数据可用性。一般来说，运行特定区块链的节点分为两种：全节点，指那些下载每一个完整区块并验证每一笔交易的节点；和轻节点，指那些仅下载区块头、并对其关注的部分状态和交易应用默克尔证明的节点。参见图11。

![图片](https://uploader.shimo.im/f/4vS7SQIbxJgZciv2.png!thumbnail)

**<center>图 11：默克尔树</center>**


如果大部分全节点被买通，他们可以创建有效或无效的区块，并发送其hash给轻节点，但从不暴露该区块的全部内容。有多种方式可以让他们从中受益。例如，图12的情况：

![图片](https://uploader.shimo.im/f/VPNsO3cICEAa22ln.png!thumbnail)

**<center>图 12：数据可用性问题</center>**


这里有3个区块：前置区块，**A**，由诚实验证人创建；当前区块，**B**，由受贿的验证人签发；以及接下来的，**C**，也将由诚实验证人创建（区块链描绘在图的右下角）。

你是个商人。当前区块**B**的验证人从前代验证人手里接收到区块**A**，接着计算出你收到钱的区块，并发送给你一个包括默克尔证明的该块的头部，显示你拥有这笔钱的状态（或者是发送给你钱的有效交易的默克尔证明）。你确信这笔交易已确定，所以提供了相应服务。

然而，验证人从未把区块B的全部内容分发给任何人。因此，构建区块C的诚实验证人无法获取该区块，只能强制终止整个系统，或者在区块A上构建新块，同时剥夺你对那笔钱的所有权。

当我们把相似的场景放到分片上，全节点和轻节点的定义一般可以对应到每个分片上：每个分片的验证人下载该分片的每个区块并验证其中的每笔交易，但是系统中的其他节点，包括那些快照分片链状态到信标链的节点，都只下载头部。因此，分片验证人对该分片来说，实际上就是全节点，而系统的其他参与者，包括信标链，就像一个轻节点那样运作。 

为了让我们前面讨论的渔夫方案有效，诚实验证人需要能下载交叉连接到信标链的区块。然而，如果恶意验证人交叉连接一个无效区块的头部（或使用它发起跨分片交易），但从不分发该区块，那么诚实验证人就无法发起挑战。

我们将介绍应对该问题的三种互补的解决方案。

### 2.5.1 监管权证明
未完待续



# 参考文献
[1] Monica Quaintance Will Martino and Stuart Popejoy. Chainweb: A proofof-work parallel-chain architecture for massive throughput. 2018. 

[2] Mustafa Al-Bassam, Alberto Sonnino, and Vitalik Buterin. Fraud proofs: Maximising light client security and scaling blockchains with dishonest majorities. CoRR, abs/1809.09044, 2018. 

[3] Songze Li, Mingchao Yu, Salman Avestimehr, Sreeram Kannan, and Pramod Viswanath. Polyshard: Coded sharding achieves linearly scaling efficiency and security simultaneously. CoRR, abs/1809.10361, 2018. 

[4] Ittai Abraham, Guy Gueta, and Dahlia Malkhi. Hot-stuff the linear, optimalresilience, one-message BFT devil. CoRR, abs/1803.05069, 2018. 

[5] Yossi Gilad, Rotem Hemo, Silvio Micali, Georgios Vlachos, and Nickolai Zeldovich. Algorand: Scaling byzantine agreements for cryptocurrencies. In Proceedings of the 26th Symposium on Operating Systems Principles, SOSP ’17, pages 51–68, New York, NY, USA, 2017. ACM. 

[6] Vitalik Buterin and Virgil Griffith. Casper the friendly finality gadget. CoRR, abs/1710.09437, 2017.
