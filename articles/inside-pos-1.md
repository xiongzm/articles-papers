
内置权益证明加密货币  第1部分： 基本结构
========================================

前言
----

纯权益证明(PoS)机制和混合权益证明机制加密货币目前正处于上升的趋势之中。以最著名的纯POS机制的加密货币Nxt为例，目前Nxt已经有几十个分叉版本（其中一些非常有前景）；Ethereum团队也在探索混合PoS/PoW机制挖矿原理，还有很多PoS型加密货币。与此同时，在概念方面也存在很多忧虑，没有足够好的正式形式说明。比如，有名的最为不幸的 “无风险”(Nothing at Stake)攻击至今仍然没有很好的描述和说明。可怕的是不论权益证明好的方面还是不好的方面，目前其背后都没有形成良好的模型和形式化的描述。这一系列文章的挑战就是在这方面取得了一些进展。


前提条件
--------

我假设您已经了解比特币和工作证明挖矿的一些简单常识。最好还有基本的编码技术以理解一些代码片段。Haskell编程语言（Haskell知识不是必须的）在文中被用来进行代码片段描述。我们将从尽可能简单的模型开始。我偏向于从最简单的事情逐步深入到更难一点的问题，而不是从一开始就进入最复杂的情况。

时间
----

维基百科将时间定义为“一种量度，事件可以通过过去，经由现在到达未来而可以排序，也是事件持续过程以及事件之间的间隔的一种测量”。在分布式环境中，因为时钟的同步以及事件的排序等等问题，时间的处理是一件很头痛的事情。幸运的是基于区块链的网络使得一些问题更加容易。因此，我们的模型可以更加简易：我们假设系统中所有时钟全部完美同步！即便如此，因为区块链网络的特性，在模型质量方面我们也不会损失太多。稍后我们将引入局部时间。

在我们的方法中，时间定义为以秒为单位，自创世块后经历的秒数，因此我们第一个对象定义相当简单：

`type Timestamp = Integer`


交易模型
--------
[比特币类模型](https://en.bitcoin.it/wiki/Transaction)中交易具有与其他交易输出相关联的交易输入部分，和它自己的输出（可能没有花费掉），与此不同，我们将从更简单的情况开始。首先，我们引入账户对象：

    data Account =
        Account {
            balance :: Integer,
            publicKey :: ByteString,
            isForging :: Bool
        }

账户中具有明确定义的账户余额。相比于真实的加密货币，这里有很多简化的东西，尤其是：

* 我们这里没有地址。在真实货币中，地址是hash(公钥)， 然而我们使用公钥本身作为地址。
* 我们甚至不使用私钥！在交易签名中需要使用私钥，这不是我们现在研究考虑的范围。

描述完时间和账户对象后，我们可以对交易进行如下的定义：

    data Transaction =
        Transaction {
            sender :: Account,
            recipient :: Account,
            amount :: Integer,
            fee :: Integer,
            txTimestamp :: Timestamp
        }

当然，现在离实际应用还差得很远。上面定义的交易中没有id, 甚至没有签名，因此任何人都可以很容易的印钱。


铸币
----

在权益证明型加密货币中没有第三方矿工，然而，股东可以是区块生成者。与工作证明型货币中的挖矿相对应的，我们则使用铸币。两者意义相当，但是：铸币是区块生成的一个过程。还记得上述的账户定义吗？isForging数据域表示账户是否正在参与铸币。


区块链
------

我们知道在每种加密货币中交易被打包进入区块中。从下面的区块定义开始，权益证明的本质开始在我们模型中显现：

    data Block =
        Block {
            transactions :: [Transaction],
            generator :: Account,
            blockTimestamp :: Timestamp
            baseTarget :: Integer,
            generationSignature :: ByteString
        }

其中`transactions`和`blockTimestamp`数据域显而易见。`generator`是生成这个区块的账户。`baseTarget`和`generationSignature`数据域与铸币算法相关，我们将在本系列文章的下一章中进行解释。

区块链则是由区块组成的序列：

`data BlockChain = BlockChain { blocks :: [Block] }`


有效余额
--------

在权益证明中生成区块的概率依赖于账户的余额。在Nxt这一类加密货币中，其概率则是有效余额的函数。比如，在Nxt中，账户的有效余额是自上一个区块之前1440个区块时的余额减去这1440个区块区间的所有花费，再加上其他账户授权给该账户铸币的额度。我们可以有如下简单的定义：

    effectiveBalance :: Account -> Integer
    effectiveBalance acc = balance acc

因此我们的模型中有效余额就仅仅只是当前的余额。


网络
----

到目前为止我们还没有谈到点对点p2p网络。网络中第一个需要定义的对象是节点：

    data Node =
        Node {
            nodeChain :: BlockChain,
            unconfirmedTxs :: [Transaction],
            account :: Account
        }


同样，我们进行了一些简化：
* 每个节点仅有一个区块链(这一点对于目前Nxt的参考软件实现的确是事实)
* 每个节点只有一个账户

接下来，定义连接(Connection)对象为节点的一个二元组：

    type Connection = (Node, Node)

我们现在可以定义我们模型中最大的类型，系统：

    data System =
        System {
            nodes :: [Node],
            connections :: [Connection],
            accounts :: [Account]
        }

在权益证明型货币中通常所有的货币都是在创世块中被“印”出来的。Nxt系统中在创世块中设置的余额为10亿：

    systemBalance :: Integer
    systemBalance = 1000000000


分叉
----

和工作证明型加密货币一样，在权益证明型加密货币中也可能发生分叉。如何定义分叉呢？区块链存储在节点中，而系统有一系列的节点([Node])，因此从系统中可以提取出带有签名的区块树（具体实现在此忽略）。

     blockTree :: System -> BlockTree

其中BlockTree仅仅是一系列区块链的别名：`type BlockTree = [BlockChain]`

为什么将一个列表称之为树呢？所有的区块链都有一些共同的前缀（至少存在共同的创世块），因此事实上我们正在处理的是树结构。然而，在系统的类型方面没有强制使用树结构，这肯定不是一个最好的设计，但是为简单起见，我们在此就这样。


性能分析
--------

如何采用Haskell代码来帮助我们进行形式模型分析？让我们从[Gavin Wood's Ethereum YellowPaper](http://gavwood.com/Paper.pdf) 中给出的一个属性开始：

`正则区块链是从根通过区块树到叶子的一个路径。为了确定路径而达成共识，概念上我们认定为拥有最多计算量的路径，或者说是最重的路径。`

正则区块链应该通过一个具有签名的函数来定义：

 `canonicalBlockchain :: System -> Maybe BlockChain`

(同样，目前没有具体的实现)。也许区块链BlockChain可以仅仅是blockchain(因此已定义了)，或者是没有任何东西(意味着没有定义) 因此，如果函数结果是定义了一个具有正则区块链的系统，那么我们可以强制更严格的条件，比如之前一段时间的正则区块链应该是当前正则区块链的前缀。

有如下两种方式来给出关于性能的结论：
1.	我们可以做出可执行的铸币算法模型，然后收集函数结果的统计信息。
2.	我们可以将我们的Haskell模型翻译成具有支持Haskell类似的语言(比如Coq)，然后根据[Curry–Howard correspondence](https://en.wikipedia.org/wiki/Curry%E2%80%93Howard_correspondence)，就可以形成定理并且证明模型属性。

以上两种方法在接下来的一章中都将展示出来。


结论及进一步的工作
-----------------

通过最简单的函数给出了我们模型中的数据结构（Timestamp, Account, Transaction, Block, BlockChain, Node, Connection, System, BlockTree）。下一章中将定义铸币算法函数，然后将提供可执行的程序模拟和形式分析。



Inside a Proof-of-Stake Cryptocurrency part 1: Basic Structures
===============================================================

Introduction
------------

Pure and hybrid Proof-of-Stake cryptocurrencies are on the rise today. The most known pure PoS example, Nxt, has tens of forks
at the moment (some of them are quite promising), the Ethereum team thinks about hybrid (PoS/PoW) mining principle, and so on.
At the same time, there are many concerns about the concept, not formalized well enough, unfortunately, for example for example even
the famous "nothing-at-stake" attack has not a good description. The sad thing is both good and bad things about proof-of-stake
have no good formal description and model behind. The challenge of this series is to make some progress in this area.


Prerequisites
-------------

I assume you know some common things about Bitcoin and proof-of-work mining. It's better to have basic coding skills
 to understand code snippets. Haskell programming language is being using for them (though Haskell knowledge isn't required).
We will start with model as simple as possible. I prefer to make a simplest thing harder than to work with complicated case from day one.

Time
----

Wikipedia defines time as "a measure in which events can be ordered from the past through the present into the future,
and also the measure of durations of events and the intervals between them".
For distributed environments working with time means headache because of clock synchronization, events ordering etc.
Fortunately blockchain-based networks make some problems easier.  Due to this our model could be even easier: we assume all clocks
in the system are synchronized! Fortunately, we will lose not too much in modelling quality because of that. Later we can introduce
local clocks.

Time in our approach is just a number of seconds from genesis block, so our first entity definition is pretty simple:

`type Timestamp = Integer`


Transaction Model
-----------------

Unlike the [Bitcoin-like model](https://en.bitcoin.it/wiki/Transaction) where transaction has inputs connected with
other transaction's outputs as well as own outputs(possibly unspent), we will work with simpler approach. First, we
introduce the Account entity:

    data Account =
        Account {
            balance :: Integer,
            publicKey :: ByteString,
            isForging :: Bool
        }

It has an explicitly defined balance. Many simplifications made in comparison with real currencies, in particular:

* We use no addresses. In real currencies the address is hash(publicKey), while we use publicKey itself as an address.

* We even don't use the private key! The private key is needed for transaction signing, and that is out of scope of our study.


Having Timestamp and Account entities described, we can define transaction as:

    data Transaction =
        Transaction {
            sender :: Account,
            recipient :: Account,
            amount :: Integer,
            fee :: Integer,
            txTimestamp :: Timestamp
        }

Again, it's far away from practical use. There's no id, no even signature, so anyone could print money easily.


Forging
-------

In proof-of-stake currencies there are no third-party "workers", instead, stake holders are block generators.
Instead of term *mining* we'll use *forging*. The meaning is the same though: forging is a process of blocks generation.
Remember the Account definition? The isForging field there shows whether account is participating in forging or not.

Blockchain
----------

In every cryptocurrency I know transactions are grouped into blocks. With the following block definition Proof-of-Stake
 nature of our model starts to appear:

    data Block =
        Block {
            transactions :: [Transaction],
            generator :: Account,
            blockTimestamp :: Timestamp
            baseTarget :: Integer,
            generationSignature :: ByteString
        }

`transactions` and `blockTimestamp` fields are self-describing.
`generator` is account generated this block.
`baseTarget` and `generationSignature` fields are related to forging algorithm and will be described in a next chapter of the tutorial.


The Blockchain is just a sequence of blocks:

`data BlockChain = BlockChain { blocks :: [Block] }`


Effective Balance
-----------------

In proof-of-stake the probability of block generation  depends on the account’s balance.
In Nxt-like currencies, it is a function of the effective balance .
E.g. in Nxt itself the effective balance is the balance 1440 blocks ago from last one minus all spendings
for that period plus leasing forging balance given by other accounts etc. We can simplify again:

    effectiveBalance :: Account -> Integer
    effectiveBalance acc = balance acc

So the effective balance in our model is just the current balance.


The Network
-----------

Nothing has been said about whole p2p network yet. The first entity to be defined here is a node in the network:

    data Node =
        Node {
            nodeChain :: BlockChain,
            unconfirmedTxs :: [Transaction],
            account :: Account
        }

Again, we're  dealing with simplifications:

* Only one blockchain per node(this is true though e.g. for current Nxt Reference Software implementation)

* Only one account per node

Having the Connection entity defined as just a tuple of Nodes:

    type Connection = (Node, Node)

we can define biggest type in our model, System:

    data System =
        System {
            nodes :: [Node],
            connections :: [Connection],
            accounts :: [Account]
        }

In proof-of-stake currencies all the money is usually being “printed” in a genesis block.
The Nxt system has ~ 1 Billion balance from the genesis block:

    systemBalance :: Integer
    systemBalance = 1000000000

Forks
-----

Forks are possible as well as with proof-of-work mining. How could we define fork? Well, a blockchain is stored in a node,
and System has a list of nodes([Node]), so tree of blocks could be exctracted from System by function with following
signature(implementation is missed for now).

     blockTree :: System -> BlockTree

Where Blocktree is just the alias for a list of Blockchain: `type BlockTree = [BlockChain]`

Why is a list called a tree? All blockchains have some common prefix(genesis block at least), so in fact
  we're dealing with a tree structure. However, not enforcing tree structure by type system is not the best design
definitely, but for simplicity's sake we leave it as is for now.


Properties Analysis
--------------------

How can Haskell code helps us with formal model analysis? Let's start with a property given in
[Gavin Wood's Ethereum YellowPaper](http://gavwood.com/Paper.pdf) :

`The canonical blockchain is a path from root to leaf
 through the entire block tree. In order to have consensus
 over which path it is, conceptually we identify the path
 that has had the most computation done upon it, or, the
 heaviest
 path.`

Canonical blockchain could be defined with a function having signature:

 `canonicalBlockchain :: System -> Maybe BlockChain`

(again, no implementation for now). Maybe BlockChain could be Just blockchain(so defined), or Nothing(means undefined).
So if function result is defined a system has some canonical blockchain. We can then enforce more strict conditions, e.g
canonical blockchain for some time ago should be the prefix of current canonical blockchain.

There are two ways to make conclusions about the property:

1. We can make executable forging algo model then gather statistics about function result.

2. We can translate our Haskell model some Haskell-like language with dependent types(e.g. Coq) then, thanks to [Curry–Howard correspondence](https://en.wikipedia.org/wiki/Curry%E2%80%93Howard_correspondence),
theorems could be formulated and proven about model properties.

Both ways will be shown in action in next chapters!


Conclusion & Further Work
-------------------------

Data structures needed for model (Timestamp, Account, Transaction, Block, BlockChain, Node, Connection, System, BlockTree) are described
along with simplest functions over them. In next chapter forging algo functions will be defined, then executable imitation and
formal analysis will be provided.



