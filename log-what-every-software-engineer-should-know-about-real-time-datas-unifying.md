# Log：程序员应当知道的实时数据统一抽象

origin:  [The Log: What every software engineer should know about real-time data's unifying abstraction](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)

我在 LinkedIn 度过了很愉快的六年时光。那时我们开始遇到单体中心数据库的局限性，有了向分布式系统演进的需要。这是一段精彩的经历：我们构建，部署了多个分布式系统并一直运行到了今天，包括一个分布式图像数据库，一个分布式搜索后端，一个 Hodoop 安装，一个 一代二代 KV 存储。

从这段经历中，我学到的最有用的东西之一是，我们所构建的很多东西的核心都是基于一个非常简单的概念：log。有时我们称之为 write-ahead log 或者 transaction log。 log 几乎伴随这计算机的诞生而诞生，许多分布式数据系统和实时应用架构的核心都是log。

如果你不理解 log，你就不可能完全理解 数据库，NoSQL, KV 存储，复制，PAXOS, hadoop，版本控制，甚至是任何的软件系统；然而，大多数对 log 都不是很熟悉。我希望改变这个现状，在本文中，我将带你逐步了解log相关的所有知识，包括：什么是log；如何使用log进行数据集成，实时处理和系统构建。



## Part 1：什么是Log?

日志或许是最简单的存储抽象。它是 append-only 的，按时间排序的record序列。看起来像这样：

![img](https://content.linkedin.com/content/dam/engineering/en-us/blog/migrated/log.png)

record被逐一append到日志的结尾，读操作从左到右依次处理。每个 entry 都会分配一个 唯一序列编号 log entry number。

record的顺序定义了时间的概念，左边的entry比右边的entry更早发生。log entry number 可以理解为这个entry的时间戳。将这个顺序描述成时间的概念一开始会有点奇怪，但是它很方便，与任何特定的物理时钟没有之间关联，在分布式系统中，这个方便的特性将变得至关重要。

record 的内容和格式对于此讨论而言并不重要。另外，我们不可能一直向日志中添加记录，因为空间总是会耗尽的。这个后续我会再深入。

所以，日志与文件和表格并没有本质的区别，文件是保存bytes的序列，表格是保存记录的序列，日志其实就是一个文件或者表格，只是日志中的记录必须以时序排列。

这里我们可能会疑惑，为什么要讨论一个这么简单的东西。append-only 记录序列是与数据系统有什么关联呢？答案是，日志有一个特定的目的：记录 在什么时候发生了什么（ what happened and when）。对于分布式数据系统，很多时候这都是问题的关键。

在展开讨论之前，我要先澄清一个疑惑。程序员都对另一种日志很熟悉——无结构的错误信息或者应用的追踪信息，使用syslog/log4j 写入到本地文件。为了区分，我将这种日志成为应用程序日志。应用程序日志是我正在描述的日志概念的退化形式。最大的区别在与，文本日志主要是供人类阅读的，而本文讨论的 "journal" 或者 ”data logs“ 是供程序访问的。

（实际上，思考一下你就会发现，人类在单台计算机上读取日志的想法已经不合时宜了。当涉及到许多服务和服务器时，这种方法很快就变得难以管理，日志的目的迅速转变成了查询和图形的输入，以了解许多机器上的行为——对于这种目的，文件中的文本信息就没有此处描述的那种结构化日志来的合适了）

### Logs in databases

我不知道log概念的起源——或许它就像二分查找那样，看起来过于简单，以至于发明者并没有留意到这是一个发明。它最早出现在IBM的  [System R](http://www.cs.berkeley.edu/~brewer/cs262/SystemR.pdf) 。日志在数据库中的使用与出现崩溃时保持各种数据结构和索引的同步有关。为了使其具有原子性和持久性，数据库**在将变更的作用到它数据结构之前，使用日志写出它们将要修改的记录的信息**。日志是对一系列发生事件的记录，每个表或索引都是记录的投影。由于日志会立即保留，因此在发生崩溃时，该日志将用作还原所有其他持久性结构的权威来源。

随着时间的流逝，日志的使用从ACID的实现细节发展为**在数据库之间复制数据**的方法。事实证明，**数据库上发生的变更顺序正是保持远程副本数据库同步所需的**。 Oracle，MySQL和PostgreSQL包含日志传送协议，以将日志的一部分传输到充当从属的副本数据库。  Oracle已将日志产品化为常规数据订阅机制，提供给非Oracle数据订阅者的 [XStreams](http://docs.oracle.com/cd/E11882_01/server.112/e16545/xstrm_intro.htm)和 [GoldenGate](http://www.oracle.com/technetwork/middleware/goldengate/overview/index.html) 使用，MySQL和PostgreSQL中的类似功能是许多数据体系结构的关键组件。

由于这样的起源，机器可读日志的概念在很长一段时间里都局限于数据库内部。使用日志作为数据订阅机制几乎是偶然出现的。但是，**这种非常抽象的方法非常适合支持各种消息传递，数据流和实时数据处理**。

### Logs in distributed systems

日志解决的两个问题，**序列化变更和分发数据**，相比之下，这二者在分布式数据系统就中显得更为重要了。这些分布式系统的核心设计问题之一是**对更新顺序达成共识**（或者对不一致产生共识并应对这些副作用）。（Agreeing upon an ordering for updates (or agreeing to disagree and coping with the side-effects) are among the core design problems for these systems.）

以日志为中心的分布式系统方法源自一个简单的观察，我将其称为状态机复制原理：

**如果2个相同的，确定性的进程，开始于一个相同状态，并且以相同的顺序接收相同的输入，它们将产生相同的输出，并结束于相同的状态**。（If two identical, deterministic processes begin in the same state and get the same inputs in the same order, they will produce the same output and end in the same state.）

确定性（[Deterministic](http://en.wikipedia.org/wiki/Deterministic_algorithm) ) 表示这个进程是 与时间无关的，不会有任何其他的外部输入影响其结果。例如，如果一个程序，其输出受线程执行的特定顺序影响，或调用gettimeofday，或某些其他不可重复的东西影响，通常认为是 不确定性的 （non-deterministic）。

进程的状态指的是在处理结束时计算机上保留在内存和磁盘中的任何数据。

**以相同的顺序获得相同的输入—— 这就是日志的来源**。这是一个非常直观的概念：如果对两个确定性代码段输入相同的输入日志，则它们将产生相同的输出。

分布式计算的应用非常明显。您可以将**让多台计算机执行相同操作**，转化成 **实现分布式一致日志以供这些进程输入**。此处日志的目的是从输入流中挤出所有不确定性，以确保处理此输入的每个副本保持同步。

当您理解它时，就会发现这个原理并没有什么复杂或深奥的：它大体的意思就是“**确定性的处理是确定性的**”（**"deterministic processing is deterministic"**）。尽管很简单，我认为它是用于分布式系统设计的通用的工具之一。

这种方法的优点之一是：**索引日志的时间戳现在充当了副本状态的时钟——你可以用一个数字来描述每个副本，即已处理日志项的最大时间戳**。该时间戳与日志相结合，可以独一无二地捕捉这个副本的完整状态。

有多种方法可以根据日志中的内容在系统中应用此原理。例如，我们可以在日志中记录服务请求，或者服务响应于请求而经历的状态更改，或者它执行的转换命令。从理论上讲，我们甚至可以记录一系列机器指令，以便每个副本执行或在每个副本上调用方法名称和参数。只要两个进程以相同的方式处理这些输入，这些进程将在副本之间保持一致。

分布式系统文献通常将 处理与复制方法 大体分两类。

- “**状态机模型**”通常是指主动-主动模型（***active-active model**），其中我们记录传入请求的日志，每个副本处理每个请求。
- 对此的略微修改版本，（称为“**主备份模型**” ，**primary-backup model**），是选择一个副本作为领导者，并允许该领导者按请求到达的顺序处理请求，并从处理请求中注销对其状态的更改。领导者进行的状态更改按顺序到应用其他副本，，以便它们同步并准备在领导者失败时接任领导者。

![img](https://content.linkedin.com/content/dam/engineering/en-us/blog/migrated/active_and_passive_arch.png)

要了解这两种方法之间的区别，让我们看一个示例问题。考虑一个复制的“算术服务”，该服务将单个数字作为其状态（初始化为零）并将该值进行加法和乘法。

- 主动-主动方法可能会注销要应用的转换，例如“ +1”，“ * 2”等。每个副本都将应用这些转换，因此要经过相同的一组值。 
- “主动-被动”方法将使单个主机执行转换并注销结果，例如“ 1”，“ 3”，“ 6”等。**此示例还清楚说明了为什么排序对于确保副本之间的一致性至关重要：对加法和乘法进行重新排序将产生不同的结果。**

![img](https://content.linkedin.com/content/dam/engineering/en-us/blog/migrated/paxos_postcard.jpg)

**分布式日志可以看作是建模共识问题的数据结构**。毕竟，日志代表对要附加的“下一个”值的一系列决策。你必须绕一下弯才能看到Paxos系列算法中的日志，尽管日志构建是它们最常见的实际应用。**对于Paxos，通常使用协议的扩展名“ multi-paxos”来完成此操作，该协议将日志建模为一系列共识问题，每个问题对应一个日志**。日志在[ZAB](http://www.stanford.edu/class/cs347/reading/zab.pdf), [RAFT](https://ramcloud.stanford.edu/wiki/download/attachments/11370504/raft.pdf), and [Viewstamped Replication](http://pmg.csail.mit.edu/papers/vr-revisited.pdf) 等其他协议中更为突出，这些协议**直接对维护分布式一致日志的问题进行建模**。

我的怀疑是，我们对此的看法在一定程度上受到了历史道路的误导，这也许是由于几十年来分布式计算的理论超过了其实际应用。实际上，共识问题比想象中的要简单得多。**计算机系统很少需要确定单个值，它们几乎总是处理一系列请求。因此，使用日志而不是简单的单值寄存器是更自然的抽象**。

此外，**对算法的关注掩盖了底层日志抽象系统的需求**。**我有理由相信，我们最终将把日志作为商品化的构建模块而不用关心其内部实现，就像我们经常谈论哈希表一样，而不必费心弄清楚是线性探测还是杂项哈希的细节其他变体。日志将变成商品化的接口，许多算法和实现会相互竞争以提供最佳的保证和最佳的性能。**



### Changelog 101: Tables 和 Events 是阴阳两极

让我们回到数据库。变更日志和表之间有一个令人着迷的对偶。日志类似于所有贷方和借方以及银行流程的列表。表保存所有帐户的当前余额。**如果有更改日志，则可以应用这些更改以创建捕获当前状态的表。该表将记录每个键的最新状态（以特定的日志时间为准）**。从某种意义上说，日志是更基本的数据结构：除了创建原始表之外，您还可以对其进行转换以创建各种派生表。 （是的，表可以表示非关系型的键控数据存储。）

![img](https://content.linkedin.com/content/dam/engineering/en-us/blog/migrated/yin-yang.jpg)

反过来，如果您有一个表进行更新，则可以记录这些更改并发布所有更新到该表状态的“更改日志”。此变更日志正是支持近实时副本所需要的。因此，从这个意义上讲，您可以将表和事件看作是互补的：**表支持静态数据，日志捕获更改**。**日志的神奇之处在于，如果它是更改的完整日志，则它不仅保存表的最终版本的内容，而且还允许重新创建可能已经存在的所有其他版本。实际上，它是对表的每个先前状态的一种备份**。

这可能使您想起源代码版本控制。源代码管理与数据库之间有着密切的关系。**版本控制解决了与分布式数据系统必须解决的问题非常相似的问题——管理状态的分布式并发更改**。版本控制系统通常对补丁序列进行建模，实际上就是日志。您可以直接与类似于表的当前代码的已签出“快照”相互作用。您会注意到，在版本控制系统中，就像在其他分布式有状态系统中一样，复制是通过日志进行的：更新时，您仅下拉补丁并将其应用到当前快照。

最近有人从 [Datomic](http://www.datomic.com/)（一家以日志为中心的数据库销售公司）看到了其中一些想法。此演示文稿很好地概述了他们如何在系统中应用该想法。当然，这些想法并不是该系统独有的，因为它们成为分布式系统和数据库文献的一部分已有十多年了。

这节看起来有点偏理论。不要担心，我们将很快介绍实用的内容。

### What's next

在本文的其余部分中，我将 不局限于 分布式计算或抽象分布式计算模型 领域，尝试 介绍 log 所擅长的各种领域。这包括： 

- **数据集成**——使组织的所有数据可以轻松地在其所有存储和处理系统中使用。 
- **实时数据处理**——计算派生数据流。 
- **分布式系统设计**——以日志为中心的设计如何简化实际系统。 

这些都围绕着**将日志作为独立服务的思想**来解决。 在每种情况下，**日志的实用性都来自日志提供的简单功能：生成持久的，可重播的历史记录**。神奇的是，这些问题的核心 都是 能够 **以确定的方式**，**以自己合适的效率** 让 **多台计算机** **回放历史记录**。



## Part Two: 数据集成

首先让我说一下“数据集成”的含义以及为什么我认为它很重要，然后我们将了解它与日志之间的关系。

`数据集成使一个组织可以在其所有服务和系统中使用所有数据。`

“数据集成”这个词并不常见，但我不知道有什么更好的。术语ETL通常仅涵盖数据集成的一小部分——填充关系数据仓库。而我所描述的许多内容可以看作是ETL的泛化，涵盖了实时系统和处理流程。

![img](https://content.linkedin.com/content/dam/engineering/en-us/blog/migrated/cabling.jpg)

或许你之前听到的更多的是关于大数据概念的炒作，而几乎没有听说过有关数据集成的内容，但是，我认为，“**使数据可用**”这个平凡的问题是大家可以关注的更有价值的事情之一。

有效使用数据遵循Maslow的[需求层次结构](http://en.wikipedia.org/wiki/Maslow's_hierarchy_of_needs)。**金字塔的底部，涉及的是捕获所有相关数据，并能够将其放到适用的处理环境中（例如精美的实时查询系统或仅文本文件和python脚本）。这些数据需要以统一的方式建模，以使其易于读取和处理。一旦满足了以统一方式捕获数据的这些基本需求，就可以在基础架构上以各种方式处理该数据（MapReduce，实时查询系统等）进行合理处理**。

值得注意的是：当时并没有可靠且完整的数据流，Hadoop集群仅是非常昂贵且难以组装的小型供暖炉。一旦数据和处理可用，人们就可以将注意力转移到更好的数据模型和一致的语义理解上。最后，注意力可以转移到更复杂的处理上-更好的可视化，报告以及算法处理和预测。

以我的经验，大多数组织在金字塔的底部都有巨大的漏洞——他们缺乏可靠的完整数据流，而是考虑直接跳到高级数据建模技术。这是完全错误的方向。

问题的关键在于，**我们应该如何在所有的数据系统中建立可靠的数据流？**

### 数据集成的两种复杂性

有两种趋势正在让数据集成变得愈加困难

**事件数据流水线**

第一个趋势是事件数据的增长。事件数据记录事情发生而不是事情是什么(. Event data records things that happen rather than things that are. )。在Web系统中，事件是用户活动日志记录，在监控和保障数据中心时，  事件是机器级别事件和统计信息。人们通常称其为“日志数据”，因为通常将其写入应用程序日志中，但这会造成 格式 和 功能 的混淆。这些数据是现代网络的核心：毕竟，Google的财富是，是建立 在点击和印象之间的关联管道 上的。

这些东西不仅限于网络公司，只是因为网络公司已经完全数字化，因此更易于使用。财务数据长期以来都是以事件为中心的。  [RFID](http://en.wikipedia.org/wiki/RFID) 将这种跟踪添加到物理对象。我认为这种趋势将随着传统业务和活动的[数字化](http://online.wsj.com/article/SB10001424053111903480904576512250915629460.html)而继续。

这种类型的事件数据记录了发生的事情，并且往往比传统数据库使用的数据大几个数量级。这对加工提出了重大挑战。

**专用数据系统的爆炸式增长**

第二个趋势来自专用数据系统的[爆炸式增长](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.68.9136)，这些数据系统在过去五年中变得很流行，并且通常免费提供。存在用于[OLAP](https://github.com/metamx/druid/wiki)，[搜索](http://www.elasticsearch.org/)，[简单的在线存储](http://cassandra.apache.org/)，[批处理](http://hadoop.apache.org/)，[图分析](http://graphlab.org/) , [KV存储](http://spark.incubator.apache.org/) 等领域的专用系统。

更多种类的更多数据的组合，以及将这些数据导入更多系统的愿望 导致了巨大的数据集成问题。

### 日志结构的数据流

日志是用于处理系统之间数据流的自然数据结构。配方很简单： **获取组织的所有数据，并将其放入中央日志以进行实时订阅**。

每个逻辑数据源都可以建模为自己的日志。数据源可以是打印 事件日志（例如单击或页面浏览）的应用程序，也可以是接受修改的数据库表。每个订阅系统都将尽快从该日志中读取日志，将每个新记录应用于其自己的存储，并更新在日志中的消费位置。订阅者可以是任何类型的数据系统—— 缓存，Hadoop，另一个站点中的另一个数据库，搜索系统等。

![img](https://content.linkedin.com/content/dam/engineering/en-us/blog/migrated/log_subscription.png)

例如，日志概念为每次更改提供一个逻辑时钟，以此可以度量所有订阅者的读取进度。这使得关于 **不同订阅者系统相对于彼此的状态** 的推论要简单得多，因为每个订阅者系统都有其已读取的“时间点”。

为了更加具体，请考虑一个简单的情况，假设有一个数据库和一组缓存服务器。日志提供了一种将更新同步到所有这些系统的方法，并提供了有关每个系统各个时间点状态变更的原因。假设我们用日志条目X编写了一条记录，然后需要从缓存中进行读取。如果我们想保证我们不会看到过时的数据，我们只需要确保我们不会从任何尚未复制到X的缓存中读取数据即可。

日志还充当缓冲区，使数据生产与数据消耗之间可以异步进行。由于许多原因，这一点很重要，但特别是当有多个订阅者可能以不同的速率消费时。这意味着，订阅系统可能会崩溃或停机进行维护，并在恢复时赶上来：订阅者以其控制的速度消费。诸如Hadoop或数据仓库之类的批处理系统可能仅每小时或每天消耗一次，而实时查询系统则可能需要实时更新。**原始数据源和日志都不了解各种数据目标系统，因此可以在不更改管道的情况下添加和删除使用者系统**。



![img](https://content.linkedin.com/content/dam/engineering/en-us/blog/migrated/tolstoy.jpg)

[“每个工作数据管道的设计都像一个日志；每个断开的数据管道都以自己的方式断开。”-托尔斯泰伯爵（作者翻译）](http://en.wikipedia.org/wiki/Anna_Karenina_principle)

["Each working data pipeline is designed like a log; each broken data pipeline is broken in its own way."—Count Leo Tolstoy (translation by the author)](http://en.wikipedia.org/wiki/Anna_Karenina_principle)

尤其重要的是：目标系统仅知道 日志 本身，而不知道源系统的任何详细信息。消费者系统不必担心数据是来自RDBMS，新型的键值存储，还是在没有任何类型的实时查询系统的情况下生成的。这看起来只是一个小问题，但实际上很关键。

我在这里使用术语“ log”代替“ messaging system”或“ pub sub”，因为它在语义上更加具体，并且在实际实现中为支持数据复制所需要的内容更加详尽地描述。我发现“发布订阅”并不意味着间接寻址消息——如果您比较两个 发布-订阅的 消息系统，您会发现它们可能 提供完全不同的事物，并且大多数模型在该领域都没有用。你可以**将日志视为一种具有持久性保证和强大排序语义的消息传递系统**。在分布式系统中，这种通信模型有时会使用[原子广播](http://en.wikipedia.org/wiki/Atomic_broadcast)的名称（名字有点糟糕）。

值得强调的是，日志仍然只是基础架构。这还不是掌握数据流的故事的结局：故事的其余部分围绕**元数据**，**模式**，**兼容性**以及**处理数据结构和演化的所有细节**。但是，除非有可靠，通用的方法来处理数据流的机制，否则谈 语义细节 就为时尚早。

### At LinkedLn

`随着LinkedIn从集中式关系数据库转移到分布式系统集合中，我看到了这种数据集成问题迅速出现。`

这些天，我们的主要数据系统包括：

- [Search](http://data.linkedin.com/projects/search)
- [Social Graph](http://engineering.linkedin.com/real-time-distributed-graph/using-set-cover-algorithm-optimize-query-latency-large-scale-distributed)
- [Voldemort](http://project-voldemort.com/) (key-value store)
- [Espresso](http://data.linkedin.com/projects/espresso) (document store)
- [Recommendation engine](http://www.quora.com/LinkedIn-Recommendations/How-does-LinkedIns-recommendation-system-work)
- OLAP query engine
- [Hadoop](http://hadoop.apache.org/)
- [Terradata](http://www.teradata.com/)
- [Ingraphs](http://engineering.linkedin.com/52/autometrics-self-service-metrics-collection) (monitoring graphs and metrics services)

这些都是专业的分布式系统，在其专业领域提供高级功能。

![img](https://content.linkedin.com/content/dam/engineering/en-us/blog/migrated/linkedin.png)

在我来这里之前，在LinkedIn就已经流传着使用日志作为数据流的想法。我们开发的最早基础架构之一是 [databus](https://github.com/linkedin/databus) 服务，该服务在我们早期的Oracle表之上提供了日志缓存抽象，以扩展对数据库更改的订阅，基于此我们可以提供社交图和搜索索引。

我将提供一些相关史料来提供相关的背景。在我们交付键值存储之后，我于2008年左右开始参与该项目。我的下一个项目是尝试对 运行中的Hadoop 进行配置变更，并在其中移动一些推荐处理。由于在这方面经验很少，我们自然会预算几个星期来获取和获取数据，而其余时间则用于实现花哨的预测算法。于是开始了一段漫长的艰苦奋斗。

我们最初计划仅从现有的Oracle数据仓库中抓取数据。第一个发现的问题是，迅速地从Oracle中抓取数据是一个黑科技。更糟糕的是，数据仓库处理不适合我们后续的生产批处理(计划使用hadoop做批处理) ——许多处理是不可逆的，并且特定于正在执行的报告。我们最终了绕开数据仓库，直接进入源数据库和日志文件。最后，我们实现了另一个管道，以将数据加载到键值存储中（[load data into our key-value store](http://data.linkedin.com/blog/2009/06/building-a-terabyte-scale-data-cycle-at-linkedin-with-hadoop-and-project-voldemort)）以提供结果。

这种日常的数据复制最终成为开发前期的主要事项。更糟糕的是，每当任何管道出现问题时，Hadoop系统就几乎没法用，对不良数据执行花哨的算法只会产生更多不良数据。

**尽管我们以相当通用的方式构建事物，但是每个新数据源都需要自定义配置才能进行设置**。它也被证明是大量错误和失败的根源。我们在Hadoop上实现的站点功能变得非常流行，并且我们发掘了一大堆对此感兴趣的工程师。**每个用户都有一个他们想要与之集成的系统列表以及一堆他们想要的新数据源**。

![img](https://content.linkedin.com/content/dam/engineering/en-us/blog/migrated/sisyphus.jpg)

我逐渐明白了几件事。

- 第一，尽管有些混乱，但我们建立的管道确实非常有价值。仅仅在新处理系统（Hadoop）中使数据可用的过程就释放了很多可能性。对以前很难做到的数据进行新的计算是可能的。许多新产品和分析只是来自将以前锁定在专用系统中的多个数据组合在一起。 

- 第二，很明显，可靠的数据加载将需要数据管道的深入支持。如果我们捕获了所需的所有结构，就可以使Hadoop数据加载完全自动化，从而无需进行手动操作即可添加新的数据源或处理架构更改——数据会神奇地出现在HDFS中，并且Hive表会自动生成具有相应列的数据源。 

- 第三，我们的数据覆盖率仍然很低。也就是说，如果您查看LinkedIn在Hadoop中可用的数据的总体百分比，那么它仍然是非常不完整的。考虑到操作每个新数据源所需的工作量，要完成工作并非易事。

我们一直在进行的**为每个数据源和目标建立自定义数据负载的方法显然是不可行的**。我们有数十个数据系统和数据存储库。连接所有这些将导致在每对系统之间建立自定义管道，如下所示：

![img](https://content.linkedin.com/content/dam/engineering/en-us/blog/migrated/datapipeline_complex.png)

请注意，数据经常双向流动，因为许多系统（数据库，Hadoop）都是数据传输的源和目的地。这意味着我们最终将为每个系统建立两个管道：一个用于获取数据，另一个用于获取数据。

显然这将需要大量人员来建设，而且永远无法运作。当我们接近完全连接时，我们最终会得到 O(N^2)  条流水线。

相反，我们需要这样的通用名称：

![img](https://content.linkedin.com/content/dam/engineering/en-us/blog/migrated/datapipeline_simple.png)

我们需要尽可能地将每个使用者与数据源隔离开。理想情况下，它们应仅与单个数据存储库集成，从而使他们可以访问所有内容。

这样做的想法是，添加新的数据系统（无论是数据源还是数据目标）都应该创建集成工作，只将其连接到单个管道而不是每个数据使用者。

这种经验使我专注于构建 [Kafka](http://kafka.apache.org/)，以将我们在消息传递系统中看到的内容与在数据库和分布式系统内部流行的日志概念相结合。我们希望**某些东西首先充当所有活动数据的中央管道，并最终用于许多其他用途**，包括从Hadoop之外部署数据，监视数据等。

长期以来，Kafka作为基础架构产品还是有点独特（有些人会说奇怪），既不是数据库，不是日志文件收集系统，也不是传统的消息传递系统。但是最近，亚马逊提供了一种非常类似于Kafka的服务，称为[Kinesis](http://aws.amazon.com/kinesis)。相似之处在于处理分区，保留数据以及在Kafka API中高低级使用者之间相当奇怪的划分。我对此感到非常高兴。创建出良好的基础架构抽象的标志是，AWS提供了它作为服务！他们的愿景似乎与我所描述的完全相似：连接所有分布式系统（DynamoDB，RedShift，S3等）的管道，以及使用EC2进行分布式流处理的基础。

### 与ETL和数据仓库的关系

让我们谈谈数据仓库。数据仓库旨在作为结构清晰的集成数据的存储库，以支持分析。这是一个好主意。对于那些不了解的人，数据仓库方法包括定期从源数据库中提取数据，将其汇总为某种可以理解的形式，然后将其加载到中央数据仓库中。拥有一个包含所有数据的干净副本的中央位置，对于数据密集型分析和处理而言是一笔非常宝贵的资产。从较高的维度来看，无论您使用的是Oracle，Teradata还是Hadoop等传统数据仓库，这种方法都不会改变太多，尽管您可能会改变加载和处理的顺序。

包含干净的集成数据的数据仓库是一项了不起的资产，但是获取这些数据的机制有些过时了。

![img](https://content.linkedin.com/content/dam/engineering/en-us/blog/migrated/oracle.jpg)

以数据为中心的组织方式，存在的主要问题是**将干净的集成数据耦合到数据仓库**。数据仓库是一个批处理查询基础结构，非常适合于多种报告和临时分析，尤其是当查询涉及简单的计数，汇总和过滤时。但是将批处理系统作为唯一的干净完整数据存储库意味着该数据对于需要实时提要的系统（实时处理，搜索索引，监视系统等）是不可用。

在我看来，ETL实际上是两件事。首先，这是**一个提取和数据清除过程**——实质上**释放了组织中各种系统中锁定的数据，并删除了特定于系统的废话**。其次，**针对数据仓库查询对数据进行重组**（即使其适合关系数据库的类型系统，被强制为星型或雪花模式，可能分解为高性能[列格式](http://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.0.0.2/ds_Hive/orcfile.html)等）。混淆这两件事是有问题的。干净，集成的数据存储库应该实时可用，以便进行低延迟处理以及在其他实时存储系统中建立索引。

更好的方法是使用中央管道（日志）和定义明确的用于添加数据的API。**与该管道集成并提供干净且结构良好的数据输入，责任在于该数据提要的生产者**。这意味着，**在系统设计和实现的过程中，他们必须考虑将数据取出并以结构良好的形式传递到中央管道的问题**。新存储系统的添加对于数据仓库团队来说没有关系，因为它们具有集成的中心点。数据仓库团队仅处理以下较简单的问题：从中央日志加载结构化的数据输入，并执行针对其系统的特定转换。

![img](https://content.linkedin.com/content/dam/engineering/en-us/blog/migrated/pipeline_ownership.png)

当考虑采用传统数据仓库之外的其他数据系统时，关于**组织可伸缩性**的这一点变得尤为重要。举例来说，有人希望在组织的整个数据集上提供搜索功能。或者说，有人想提供具有实时趋势图和警报的亚秒级数据流监视。在这两种情况下，传统数据仓库甚至Hadoop集群的基础架构都是不合适的。更糟糕的是，为支持数据库负载而构建的ETL处理管道可能无法满足其他系统的需求，这使得引导这些基础设施的工作像采用数据仓库一样大。这不太可行，并且可能有助于解释为什么大多数组织不容易将这些功能用于所有数据。**相比之下，如果组织建立了统一的，结构良好的数据源，则使任何新系统完全访问所有数据仅需要一点点集成管道即可连接到管道**。

对于特定的清理或转换，此体系结构还提供了多种可选的方案：

- 由数据生产者完成
- 由日志上的实时转换来完成（这又会生成新的转换后的日志）
- 由数据消费者来完成

最好的模型是在由数据发布者在发布之前进行清理。这可以确保到日志系统中的数据都是规范格式的，并且不会保留产生该数据的特定代码或可能已维护该数据的存储系统的任何保留。这些细节最好由创建数据的团队处理，因为他们最了解自己的数据。在此阶段应用的任何逻辑都应该是无损且可逆的。

可以实时进行的任何类型的增值转换，都应作为对原始日志输入的后处理，包括事件数据的会话化，或添加其他通常令人感兴趣的字段。原始日志仍然可用，但是**这种实时处理会生成包含增强数据的派生日志**。

最后，在加载过程中，仅应执行特定于目标系统的聚合。这可能包括将数据转换为特定的星型或雪花模式，以便在数据仓库中进行分析和报告。因为此阶段（最自然地映射到传统ETL流程）现在是在更干净，更统一的流集上完成的，因此应大大简化。

### 日志文件与事件

让我们来谈谈这种体系结构的一个附带好处：支持解耦的，事件驱动的系统。

Web行为数据的典型方法是记录到文本文件，然后可以将其丢到数据仓库或Hadoop中进行聚合和查询。此问题与所有批处理ETL的问题相同：**它将数据流与数据仓库的功能和处理计划耦合在一起**。

在LinkedIn，我们以日志为中心的方式构建了事件数据处理。我们将Kafka用作中央的多用户事件日志系统。我们定义了数百种事件类型，每种类型都捕获有关特定类型的操作的独特属性。它涵盖了从页面浏览量，广告展示次数和搜索到服务调用和应用程序异常的所有内容。

要了解此方法的优点，请想象一个简单的事件——在职位页面上显示招聘启事。职位页面应仅包含展示该职位所需的逻辑。但是，在一个动态的站点中，很容易将其与与展示工作无关的其他逻辑放在一起。例如，假设我们需要集成以下系统：

1. 我们需要将此数据发送到Hadoop和数据仓库以进行脱机处理 
2. 我们需要计算观看次数，以确保观看者不会尝试抓取某些内容 
3. 我们需要汇总此次访问，以显示在职位发布者的分析页面中 
4. 我们需要记录该视图，以确保适当地限制该用户的各个职位推荐的展示次数（我们不想一遍又一遍地展示相同的内容） 
5. 我们的推荐系统可能需要记录访问以正确跟踪该职位的受欢迎程度 
6. 等等

很快，展示职位的简单任务就变得非常复杂。而且，当我们添加显示的其他位置（移动应用程序等）时，必须保留此逻辑，并且复杂性也会增加。更糟糕的是，我们现在需要与之交互的系统变得错综复杂——负责职位展示的开发者需要了解许多其他系统和功能，并确保它们正确集成。这还仅仅是问题的一个玩具版本，任何实际的应用程序的复杂性只会有过之而无不及。

“事件驱动”模式提供了一种简化方法。职位显示页面现在仅显示一个职位，并记录与该职位实际相关的各种属性，职位浏览者，以及其他与职位展示相关的其他有用信息。其他的所有对这些信息感兴趣的系统——推荐系统，安全系统，职位发布者分析系统和数据仓库——都只订阅该信息源并进行处理。职业展示代码不需要知道这些其他系统，并且添加新的数据使用方也不需要更改职位展示代码。

### 构建可拓展的日志

当然，将发布者与订阅者分开并不是什么新鲜事。但是，如果您想保留一个提交日志，以充当消费类网站上发生的所有事件的多用户实时日志，则可伸缩性将是主要挑战。如果我们无法构建足够快速，廉价且可扩展的日志以使其大规模实用，那么将日志用作通用集成机制绝不会是一种优雅的幻想。

系统人员通常将分布式日志视为缓慢而繁重的抽象（并且通常仅将其与Zookeeper可能适合的“元数据”使用类型相关联）。但是，如果考虑周到的实现专注于记录大型数据流，则不必如此。在LinkedIn，我们目前每天通过Kafka运行超过600亿个唯一消息写操作（如果从[数据中心之间的镜像](http://kafka.apache.org/documentation.html#datacenters)中计算写操作，则为数千亿个）。

我们在Kafka中使用了一些技巧来支持这种规模： 

1. 对日志做分区(Partitioning ) 
2. 通过批量读写来优化吞吐量 
3. 避免不必要的数据复制 

为了允许水平缩放，我们将日志分成多个分区：

![img](https://content.linkedin.com/content/dam/engineering/en-us/blog/migrated/partitioned_log.png)

**每个分区都是完全有序的日志，但是分区之间没有全局排序**（您的消息中可能包含一些挂钟时间）。**将消息分配给特定分区的分区策略可由编写者控制，大多数用户选择按某种键（例如，用户ID）进行分区**。分区允许日志追加在分片之间不协调的情况下发生，并且允许系统的吞吐量随Kafka集群大小线性扩展。

**每个分区在可配置数量的副本之间进行复制，每个副本具有相同的分区日志副本**。在任何时候，他们中的一个都会担任 Leader ；如果 Leader 失败，其中一个副本将接管 Leader  。

跨分区缺乏全局顺序是一个限制，但我们觉得这并不重要。**实际上，与日志的交互通常来自成百上千个不同的进程，因此谈论其行为的总体顺序是没有意义的。相反，我们提供的保证是每个分区都是按顺序保留的，而Kafka保证从单个发送者追加到特定分区的消息将按照其发送顺序进行分发**。(译者：可以配合topic和分区策略，将需要有序保证的数据子集 发送到同一分区)

最后，**Kafka使用一种简单的二进制格式，该格式在内存日志，磁盘日志以及网络数据传输之间维护**。这使我们能够利用多种优化方法，包括[零拷贝数据传输](最后，Kafka使用一种简单的二进制格式，该格式在内存日志，磁盘日志以及网络数据传输之间维护。这使我们能够利用多种优化方法，包括零拷贝数据传输。)。

这些优化的累积效果是，即使维护的数据集大大超出内存，通常也可以以磁盘或网络支持的速率写入和读取数据。

本文并非主要是关于Kafka的，因此我将不进一步讨论。您可以在[此处]((http://sites.computer.org/debull/A12june/pipeline.pdf))阅读[LinkedIn的方法的更详细概述]，在[此处](http://kafka.apache.org/documentation.html#design) 阅读[Kafka的设计的全面概述]。



## Part Three: 日志 与 实时流数据处理

到目前为止，我仅描述了什么是从一个地方到另一个地方复制数据的理想方法。但是在存储系统之间交换字节，故事还远没有结束。事实证明，“ log”是“ stream”的另一种表述，log是流处理的核心。

但是，等等，流处理到底是什么？

### 流处理

如果您喜欢90年代末和2000年代初的[数据库文献](如果您喜欢90年代末和2000年代初的数据库文献或半成功的数据基础结构产品，则可能将流处理与为事件驱动的处理构建SQL引擎或"框和箭"界面的努力相关联。)或半成功的[数据基础结构产品](http://en.wikipedia.org/wiki/StreamBase_Systems)，则可能将流处理与为事件驱动的处理构建SQL引擎或 “方格+箭头” 界面的努力相关联。

如果您关注开源数据系统的爆炸式增长，则可能会将流处理与该空间中的某些系统相关联，例如 [Storm](http://storm-project.net/)，[Akka](http://akka.io/)，[S4](http://incubator.apache.org/s4) 和Samza。但是大多数人将它们视为一种异步消息处理系统，它与可感知群集的RPC层没有什么不同（实际上，该领域中的某些功能就是这样）。

这两种观点都有些局限。流处理与SQL无关。它也不限于实时处理。没有内在的原因，您无法使用多种不同的语言来表示昨天或一个月前的数据流。

![img](https://content.linkedin.com/content/dam/engineering/en-us/blog/migrated/census.jpg)

我认为流处理的范围更广：用于连续数据处理的基础结构。我认为流处理计算模型可以像MapReduce或其他分布式处理框架一样通用，但是具有产生低延迟结果的能力。

**处理模型的选择关键在于数据收集的方法。分批收集的数据自然分批处理。连续收集数据时，自然会对其进行连续处理**。

美国人口普查为批量数据收集提供了一个很好的例子。人口普查定期开始，并让人们挨家挨户走动，对美国公民进行暴力发现和列举。在1790年人口普查首次开始时，这很有意义。当时的数据收集本来就是面向批处理的，它涉及骑马骑马并在纸上写下记录，然后将这批记录传输到一个中央位置，在此人员将所有计数加起来。如今，当您描述人口普查过程时，人们立即想知道为什么我们不保留出生和死亡的日志，而是连续地或按需要的粒度生成人口计数。

这是一个极端的例子，但是许多数据传输过程仍然依赖于进行定期转储以及批量传输和集成。处理批量转储的唯一自然方法是分批处理。但是，随着这些流程被连续馈送所取代，人们自然会开始朝着连续处理迈进，以平滑所需的处理资源并减少延迟。

例如，LinkedIn几乎根本没有批处理数据收集。我们的大部分数据是活动数据或数据库更改，两者都是连续发生的。实际上，当您考虑任何业务时，底层的机制几乎总是一个连续的过程，就像Jack Bauer告诉我们的那样，事件是实时发生的。批量收集数据时，几乎总是由于某些手动步骤或缺乏数字化，或者是某些非数字过程的自动化遗留下来的历史遗迹。当技术人员是邮件而人工进行处理时，数据的传输和响应通常非常慢。自动化的第一步总是保留原始过程的形式，因此这通常会持续很长时间。

每天运行的生产“批处理”作业通常也可以理解成一种窗口大小为一天的连续计算。当然，基础数据总是在变化。这些实际上在LinkedIn上非常普遍（以及使它们在Hadoop中工作的机制如此棘手），以至于我们实现了用于管理增量Hadoop工作流的整个框架。

从这个角度看，很容易对流处理有不同的看法：只是处理，它在要处理的基础数据中包括时间概念，不需要数据的静态快照，因此可以在输出时产生输出。用户控制的频率，而不是等待数据集的“结尾”。从这个意义上讲，流处理包含了批处理，并且在实时数据盛行的情况下，流处理是非常重要的概括。

那么，为什么传统上将流处理视为利基应用呢？我认为最大的原因是缺乏实时数据收集，使得连续处理成为学术界关注的问题。

我认为缺乏实时数据收集可能注定了商业流处理系统的失败。他们的客户仍在进行面向文件的ETL和数据集成的每日批处理。建立流处理系统的公司专注于提供附加到实时数据流的处理引擎，但事实证明，当时很少有人真正拥有实时数据流。实际上，在我加入LinkedIn的职业生涯的早期，一家公司曾试图向我们出售一个非常酷的流处理系统，但是由于当时所有数据都是按小时收集的，因此我们能想到的最好的应用程序是按小时进行传输一小时结束时，文件将进入流系统！他们指出，这是一个相当普遍的问题。例外实际上证明了这里的规则：金融是流处理取得一定成功的一个领域，而这正是实时数据流已经成为规范并且处理已成为瓶颈的领域。

即使已经存在一个健康的批处理生态系统，我认为流处理作为基础结构样式的实际适用性还是很广泛的。我认为它涵盖了实时请求/响应服务与脱机批处理之间的基础架构差距。对于现代互联网公司，我认为他们的代码中约有25％属于此类。

事实证明，该日志解决了流处理中一些最关键的技术问题，我将对其进行描述，但是它解决的最大问题是仅使数据在实时多用户数据馈送中可用。对于那些对更多细节感兴趣的人，我们开放了源代码 [Samza](http://samza.incubator.apache.org/)，这是一个明显基于许多想法的流处理系统。我们在[此处的文档](http://samza.incubator.apache.org/learn/documentation/0.7.0)中更详细地描述了许多这些应用程序。

#### 数据流图

![img](https://content.linkedin.com/content/dam/engineering/en-us/blog/migrated/dag.png)

流处理最有趣的方面与流处理系统的内部无关，而是与它如何扩展我们对早期数据集成讨论中的数据输送的概念有关。我们主要讨论了原始数据的提要或日志，即在执行各种应用程序时产生的事件和数据行。但是，流处理使我们还可以包括根据其他提要计算得出的提要。这些派生的提要对消费者而言没有什么区别，与从中计算得出的主要数据的提要一样。这些派生的提要可以封装任意复杂性。

让我们深入探讨一下。就我们的目的而言，流处理作业将是从日志读取并将输出写入日志或其他系统的任何内容。它们用于输入和输出的日志将这些过程合并为处理阶段流程图。确实，以这种方式使用集中式日志，您可以将组织的所有数据捕获，转换和流 ，展示为 一系列写入其中的日志和流程。 

流处理器根本不需要高级框架：它可以是从日志读取和写入的任何进程或一组进程，但是可以提供其他基础结构和支持来帮助管理处理代码。

集成日志有双重目的：

首先，它使每个数据集成为**多订户**和**有序**。回顾我们的“状态复制”原理，以记住顺序的重要性。为了更加具体，请考虑数据库中的更新流——如果在处理过程中对同一记录重新排序两个更新，则可能会产生错误的最终输出。该顺序比TCP之类的命令所提供的顺序更永久，因为它不仅限于单个点对点链接，而且在过程失败和重新连接后仍然可以生存。

其次，日志为进程提供缓冲。这是非常基本的。如果处理以不同步的方式进行，则可能发生上游数据生成作业生成数据的速度快于另一个下游作业消耗数据的速度。发生这种情况时，处理必须阻止，缓冲或丢弃数据。删除数据可能不是一种选择。阻塞可能会导致整个处理图变得停顿。日志充当非常大的缓冲区，使进程可以重新启动或失败，而不会减慢处理图的其他部分。当将数据流扩展到更大的组织时，这种隔离特别重要，在该组织中，处理工作是由许多不同的团队进行的。我们不能有一项错误的工作导致背压停止整个处理流程。

[Storm](http://storm-project.net/) 和 [Samza](http://samza.incubator.apache.org/) 都以这种方式构建，并且可以使用Kafka或其他类似系统作为日志系统。

### 有状态的实时处理

某些实时流处理只是一次无状态记录转换，但是许多应用则是更复杂的计数，聚合或流中窗口上的联接。例如，可能想要用有关执行点击的用户的信息来丰富事件流（例如点击流），实际上是将点击流添加到用户帐户数据库中。不管怎样，这种处理最终需要处理器维护某种状态：例如，在计算计数时，到目前为止要维护计数。如果处理器本身可能发生故障，如何正确维护这种状态？ 

最简单的选择是将状态保留在内存中。但是，如果进程崩溃，它将失去其中间状态。如果仅在窗口上保持状态，则该过程可能会退回到日志中窗口开始的位置。但是，如果要计数一个小时以上，则可能不可行。 

一种替代方法是简单地将所有状态存储在远程存储系统中，然后通过网络加入该存储。问题是没有数据的局部性和大量的网络往返。 

我们如何支持像“表”这样的东西，这些东西在处理过程中被划分了？ 

回想一下有关表和日志双重性的讨论。这给了我们确切的工具，它能够将流转换为与我们的处理位于同一位置的表，以及一种处理这些表的容错能力的机制。

流处理器可以将其状态保存在本地“表”或“索引”中- [bdb](http://www.oracle.com/technetwork/products/berkeleydb)，[leveldb](https://code.google.com/p/leveldb)甚至是更不寻常的东西（如[Lucene](http://lucene.apache.org/)或[fastbit](https://sdm.lbl.gov/fastbit)索引）。该存储库的内容从其输入流中传递（首先可能是在应用任意转换之后）。它可以记录该本地索引的更改日志，以使其在崩溃时恢复其状态并重新启动。这种机制允许一种通用机制，用于将任意索引类型中的共分区状态保持在传入流数据本地。

当过程失败时，它将从变更日志中恢复其索引。日志是一次备份将本地状态转换为某种增量记录的方式。 这种状态管理方法具有优雅的特性，即处理器的状态也保持为日志。我们可以像对待数据库表的更改日志一样来考虑该日志。实际上，处理器具有与它们一起维护的非常类似的共分区表。由于此状态本身是日志，因此其他处理器可以订阅它。在处理的目标是更新最终状态而该状态是处理的自然输出的情况下，这实际上可能非常有用。 当与出于数据库集成目的而从数据库中发出的日志结合使用时，日志/表双重性的功能将变得显而易见。更改日志可以从数据库中提取，并由各种流处理器以不同的形式索引以加入事件流。

我们将在Samza中详细介绍这种管理状态处理的样式，并在此处提供更多实际[示例](http://samza.incubator.apache.org/learn/documentation/0.7.0/container/state-management.html)。

### 日志压缩

![img](https://content.linkedin.com/content/dam/engineering/en-us/blog/migrated/log_compaction_0.png)

当然，我们不能指望一直为所有状态更改保留完整的日志。除非有人要使用无限空间，否则必须以某种方式清除日志。我将在Kafka中稍微介绍一下此实现，以使其更加具体。在Kafka中，清理有两个选项，具体取决于数据是包含键更新还是事件数据。对于事件数据，Kafka仅支持保留数据窗口。通常，将其配置为几天，但是可以根据时间或空间来定义窗口。但是，对于键控数据，完整日志的一个不错的属性是您可以重播它以重新创建源系统的状态（可能在另一个系统中重新创建它）。

但是，随着时间的流逝，保留完整的日志将占用越来越多的空间，并且重播将花费越来越长的时间。因此，在 kafka，我们支持其他类型的保留。我们不只是丢弃旧的日志，而是删除过时的记录，即记录其主键具有最近更新的记录。通过这样做，我们仍然可以保证日志包含源系统的完整备份，但是现在我们不再能够重新创建源系统的所有先前状态，而只能重新创建最近的状态。我们称此功能为[日志压缩](https://cwiki.apache.org/confluence/display/KAFKA/Log+Compaction)。

## Part Four: 系统构建

我要讨论的最后一个主题是日志在联机数据系统的系统设计中的作用。 在日志中，分布式数据库内部的数据流所起的作用与大型组织中的数据集成所起的作用之间存在类比。在这两种情况下，它都负责数据流，一致性和恢复。毕竟，一个组织，如果不是一个非常复杂的分布式数据系统，那又是什么？

### Unbundling?

因此，也许您稍作斜视，就可以将整个组织的系统和数据流视为一个分布式数据库。

- 您可以将所有单个面向查询的系统（Redis，SOLR，Hive表等）当作数据上的特定索引。
- 您可以将Storm或Samza之类的流处理系统视为一个非常完善的触发器并查看实现机制。

我已经注意到，古典数据库人员非常喜欢这种观点，因为它最终向他们解释了人们在使用所有这些不同数据系统的情况—它们只是不同的索引类型！ 不可否认，现在数据类型的爆炸式增长，但实际上，这种复杂性一直存在。即使在关系数据库的鼎盛时期，组织也有很多关系数据库！因此，自从大型机真正将所有数据都放在一个地方以来，可能就没有做到过真正的数据集成。将数据分为多个系统的动机很多：规模，地理位置，安全性和性能隔离是最常见的。但是，这些问题可以通过一个好的系统来解决：例如，一个组织有可能拥有单个Hadoop集群，该集群包含所有数据并服务于众多的选民。 因此，在迁移到分布式系统中时已经有可能简化数据的处理：将每个系统的许多小实例合并为几个大集群。许多系统尚不足以允许这样做：它们没有安全性，或者不能保证性能隔离，或者只是扩展性不够好。但是这些问题都是可以解决的。 我的观点是，不同系统的爆炸式增长是由构建分布式数据系统的困难引起的。通过减少到单个查询类型或用例，每个系统都可以将其范围缩小到可以构建的一组事物中。但是运行所有这些系统会产生太多的复杂性。 

我看到将来可能会遵循三个可能的方向。 

第一种可能性是现状的延续：系统的分离或多或少地保持了相当长的时间。之所以会发生这种情况，可能是因为分发的困难难以克服，或者是因为这种专业化使每个系统的便利性和功能性达到了新的水平。只要这仍然成立，对于成功使用数据，数据集成问题将仍然是最重要的重要事项之一。在这种情况下，集成数据的外部日志将非常重要。 

第二种可能性是，可能存在重新整合，即具有足够普遍性的单个系统开始在所有不同功能中合并回单个uber系统。这个超级系统从表面上看就像关系数据库，但是它在组织中的使用却大不相同，因为您只需要一个大数据库，而不是几个小数据库。在这个世界上，除了系统内部解决的问题之外，没有真正的数据集成问题。我认为建立这样一个系统的实际困难使这不太可能。 

但是，还有另一个可能的结果，我作为工程师实际上很有吸引力。新一代数据系统的一个有趣方面是，它们几乎都是开源的。开源提供了另一种可能性：数据基础架构可以捆绑为服务和面向应用程序的系统api的集合。您已经在Java堆栈中看到了这种情况：

- [Zookeeper](http://zookeeper.apache.org/) 处理大部分系统协调（也许在更高层次的抽象等帮助下，如 [Helix](http://helix.incubator.apache.org/) or [Curator](http://curator.incubator.apache.org/)).
- [Mesos](http://mesos.apache.org/) and [YARN](http://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/YARN.html)  处理虚拟化和资源管理
- 嵌入式库如 [Lucene](http://lucene.apache.org/) and [LevelDB](https://code.google.com/p/leveldb) 负责索引
- [Netty](http://netty.io/), [Jetty](http://www.eclipse.org/jetty) 和更高级别的封装，如 [Finagle](http://twitter.github.io/finagle) and [rest.li](http://rest.li/)  处理远程通信
- [Avro](http://avro.apache.org/), [Protocol Buffers](https://code.google.com/p/protobuf), [Thrift](http://thrift.apache.org/), and [umpteen zillion](https://github.com/eishay/jvm-serializers/wiki) 等其他库负责序列化
- [Kafka](http://kafka.apache.org/) and [Bookeeper](http://zookeeper.apache.org/bookkeeper)  提供支持日志。

如果将这些东西堆叠在一起并斜视一下，它看起来会有点像分布式数据系统工程的传统版本。您可以将这些成分拼凑在一起，以创建大量可能的系统。对于最终用户来说，这显然不是一个故事，他们大概更关心API，而不是如何实现，但是这可能是在不断发展的更加多样化和模块化的世界中实现单个系统的简单性的一条途径。 。如果分布式系统的实施时间从数年到数周不等，因为出现了可靠，灵活的构建基块，那么合并为单个整体系统的压力就消失了。

### 系统架构中，日志的定位

假设存在外部日志的系统允许单个系统放弃很多自己的复杂性，并依靠共享日志。这是我认为日志可以执行的操作：

- 通过对节点的并发更新进行排序来处理数据一致性（无论是最终的还是即时的） 
- 提供节点之间的数据复制 
- 向编写者提供“提交”语义（即，仅在保证不会丢失写内容时才进行确认） 
- 提供系统的外部数据订阅源
- 提供恢复丢失数据的失败副本或引导新副本的功能 
- 处理节点之间的数据重新平衡。

这实际上是分布式数据系统所做工作的重要部分。实际上，剩下的大部分与最终面向客户的查询API和索引策略有关。这恰好是应该随系统而异的部分：例如，全文搜索查询可能需要查询所有分区，而按主键查询可能只需要查询负责该键数据的单个节点。

它是这样工作的。该系统分为两个逻辑部分：日志和服务层。日志按顺序捕获状态更改。服务节点存储服务查询所需的任何索引（例如，键值存储可能具有btree或sstable之类的东西，搜索系统将具有反向索引）。写入可能直接进入日志，尽管它们可能会被服务层代理。写入日志会产生逻辑时间戳记（例如日志中的索引）。如果系统已分区（我假设已分区），则日志和服务节点将具有相同数量的分区，尽管它们的计算机数量可能非常不同。

![img](https://content.linkedin.com/content/dam/engineering/en-us/blog/migrated/system.png)

服务节点订阅日志，并按照日志存储日志的顺序将写入操作尽快应用于其本地索引。 

客户端可以通过在其查询中提供写入的时间戳来从任何节点获取您的写入语义-接收到此类查询的服务节点会将所需的时间戳与自己的索引点进行比较，并在必要时将请求延迟到它至少已经索引了那个时间，以避免提供过时的数据。 

服务节点可能具有或不必具有“主控权”或“领导者选举”的任何概念。对于许多简单的用例，由于日志是事实的来源，因此服务节点可以完全没有领导者。 分布式系统必须做的棘手的事情之一是处理故障节点的恢复或将分区从一个节点移到另一个节点。

一种典型的方法是使日志仅保留一个固定的数据窗口，并将其与分区中存储的数据快照结合起来。日志也有可能保留数据的完整副本，并垃圾回收日志本身。这将大量的复杂性从服务层（这是系统特定的）移出到日志中（这可以是通用的）。 

通过使用此日志系统，您可以获得针对数据存储内容的完整开发的订阅API，该订阅API将ETL馈送至其他系统。实际上，许多系统可以在提供不同索引的同时共享同一日志，如下所示：

![img](https://content.linkedin.com/content/dam/engineering/en-us/blog/migrated/full-stack.png)

请注意，这种以日志为中心的系统本身是如何立即为其他系统中的处理和加载提供数据流的。同样，流处理器可以使用多个输入流，然后通过另一个对输出进行索引的系统为它们提供服务。 

我发现这种对日志和查询api的系统视图十分重要，因为它使您可以将查询特征与系统的可用性和一致性方面分开。我实际上认为，这甚至是一种有用的方法，可以从心理上分析并非以这种方式构建的系统以更好地理解它。 

值得注意的是，尽管Kafka和Bookeeper是一致的日志，但这不是必需的。您可以轻松地将类似Dynamo的数据库分解为最终一致的AP日志和键值服务层。这样的日志有点麻烦，因为它将重新传递旧消息，并取决于订阅者来处理（类似于Dynamo本身）。 

在日志中拥有单独的数据副本（尤其是完整副本）的想法使许多人感到浪费。实际上，尽管有一些因素使这成为一个小问题。首先，日志可以是一种特别有效的存储机制。我们在生产的Kafka服务器上每个数据中心存储超过75TB的数据。同时，许多服务系统需要更多的内存才能有效地提供数据（例如，文本搜索通常全部位于内存中）。服务系统也可以使用优化的硬件。例如，我们大多数实时数据系统要么内存不足，要么使用SSD。相反，日志系统仅执行线性读写，因此使用大型多TB硬盘驱动器非常满意。最后，如上图所示，在数据由多个系统提供服务的情况下，日志成本将在多个索引上摊销。这种组合使外部日志的花费变得很小。 

这正是LinkedIn用来构建其自己的许多实时查询系统的模式。这些系统提供数据库（使用Databus作为日志抽象或来自Kafka的专用日志），并在该数据流之上提供特定的分区，索引和查询功能。这是我们实现搜索，社交图和OLAP查询系统的方式。实际上，将单个数据源（无论是实时源还是从Hadoop派生的源）复制到多个服务系统中进行实时服务是很普遍的。事实证明，这是一个巨大的简化假设。这些系统完全不需要外部可访问的写入api，Kafka和数据库用作记录系统，并且更改通过该日志流到适当的查询系统。写入由托管特定分区的节点在本地处理。这些节点将日志提供的提要盲目地转录到其自己的存储中。可以通过重播上游日志来恢复发生故障的节点。 

这些系统对日志的依赖程度有所不同。完全依赖的系统可以利用日志进行数据分区，节点还原，重新平衡以及一致性和数据传播的所有方面。在这种设置中，实际的服务层实际上就是一种“缓存”，其结构旨在通过直接进入日志的方式来实现特定类型的处理。



## The End

如果到目前为止，您将了解我对日志的大部分认知。

以下是一些您可能想查看的有趣参考。 

每个人似乎在同一事物上使用不同的术语，因此将数据库文献与分布式系统的内容连接到与开源世界有关的各种企业软件阵营，这有点令人困惑。尽管如此，这里还是一些大致的指示。

学术论文，系统，讲座和博客：

- A good overview of [state machine](http://www.cs.cornell.edu/fbs/publications/smsurvey.pdf) and [primary-backup](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.20.5896) replication
- [PacificA](http://research.microsoft.com/apps/pubs/default.aspx?id=66814) is a generic framework for implementing log-based distributed storage systems at Microsoft.
- [Spanner](http://static.googleusercontent.com/external_content/untrusted_dlcp/research.google.com/en/us/archive/spanner-osdi2012.pdf)—Not everyone loves logical time for their logs. Google's new database tries to use physical time and models the uncertainty of clock drift directly by treating the timestamp as a range.
- [Datanomic](http://www.datomic.com/): [Deconstructing the database](https://www.youtube.com/watch?v=Cym4TZwTCNU) is a great presentation by Rich Hickey, the creator of Clojure, on his startup's database product.
- [A Survey of Rollback-Recovery Protocols in Message-Passing Systems](http://www.cs.utexas.edu/~lorenzo/papers/SurveyFinal.pdf). I found this to be a very helpful introduction to fault-tolerance and the practical application of logs to recovery outside databases.
- [Reactive Manifesto](http://www.reactivemanifesto.org/)—I'm actually not quite sure what is meant by reactive programming, but I think it means the same thing as "event driven". This link doesn't have much info, but [this class](https://www.coursera.org/course/reactive) by Martin Odersky (of Scala fame) looks facinating.
- Paxos!
  - Original paper is [here](http://research.microsoft.com/en-us/um/people/lamport/pubs/lamport-paxos.pdf). Leslie Lamport has an interesting [history](http://research.microsoft.com/en-us/um/people/lamport/pubs/pubs.html#lamport-paxos) of how the algorithm was created in the 1980s but not published until 1998 because the reviewers didn't like the Greek parable in the paper and he didn't want to change it.
  - Even once the original paper was published it wasn't well understood. Lamport [tries again](http://research.microsoft.com/en-us/um/people/lamport/pubs/paxos-simple.pdf) and this time even includes a few of the "uninteresting details" of how to put it to use using these new-fangled automatic computers. It is still not widely understood.
  - [Fred Schneider](http://www.cs.cornell.edu/fbs/publications/SMSurvey.pdf) and [Butler Lampson](http://research.microsoft.com/en-us/um/people/blampson/58-consensus/Abstract.html) each give more detailed overview of applying Paxos in real systems.
  - A few Google engineers summarize [their experience](http://www.cs.utexas.edu/users/lorenzo/corsi/cs380d/papers/paper2-1.pdf) implementing Paxos in Chubby.
  - I actually found all the Paxos papers pretty painful to understand but dutifully struggled through. But you don't need to because [this video](https://www.youtube.com/watch?v=JEpsBg0AO6o) by [John Ousterhout](http://www.stanford.edu/~ouster/cgi-bin/papers/lfs.pdf) (of log-structured filesystem fame!) will make it all very simple. Somehow these consensus algorithms are much better presented by drawing them as the communication rounds unfold, rather than in a static presentation in a paper. Ironically, this video was created in an attempt to show that Paxos was hard to understand.
  - [Using Paxos to Build a Scalable Consistent Data Store](http://arxiv.org/pdf/1103.2408.pdf): This is a cool paper on using a log to build a data store, by Jun, one of the co-authors is also one of the earliest engineers on Kafka.
- Paxos has competitors! Actually each of these map a lot more closely to the implementation of a log and are probably more suitable for practical implementation:
  - [Viewstamped Replication](http://pmg.csail.mit.edu/papers/vr-revisited.pdf) by Barbara Liskov is an early algorithm to directly model log replication.
  - [Zab](http://www.stanford.edu/class/cs347/reading/zab.pdf) is the algorithm used by Zookeeper.
  - [RAFT](https://ramcloud.stanford.edu/wiki/download/attachments/11370504/raft.pdf) is an attempt at a more understandable consensus algorithm. The [video presentation](https://www.youtube.com/watch?v=YbZ3zDzDnrw), also by John Ousterhout, is great too.
- You can see the role of the log in action in different real distributed databases.
  - [PNUTS](http://www.mpi-sws.org/~druschel/courses/ds/papers/cooper-pnuts.pdf) is a system which attempts to apply to log-centric design of traditional distributed databases at large scale.
  - [HBase](http://hbase.apache.org/) and [Bigtable](http://research.google.com/archive/bigtable.html) both give another example of logs in modern databases.
  - LinkedIn's own distributed database [Espresso](http://www.slideshare.net/amywtang/espresso-20952131), like PNUTs, uses a log for replication, but takes a slightly different approach using the underlying table itself as the source of the log.
- If you find yourself comparison shopping for a replication algorithm, [this paper](http://arxiv.org/abs/1309.5671) may help you out.
- [Replication: Theory and Practice](http://www.amazon.com/Replication-Practice-Lecture-Computer-Theoretical/dp/3642112935) is a great book that collects a bunch of summary papers on replication in distributed systems. Many of the chapters are online (e.g. [1](http://disi.unitn.it/~montreso/ds/papers/replication.pdf), [4](http://research.microsoft.com/en-us/people/aguilera/stumbling-chapter.pdf), [5](http://www.distributed-systems.net/papers/2010.verita.pdf), [6](http://www.cs.cornell.edu/ken/history.pdf), [7](http://www.pmg.csail.mit.edu/papers/vr-to-bft.pdf), [8](http://engineering.linkedin.com/distributed-systems/www.cs.cornell.edu/fbs/publications/TrustSurveyTR.pdf)).
- Stream processing. This is a bit too broad to summarize, but here are a few things I liked.
  - [Models and Issues in Data Stream Systems](http://infolab.usc.edu/csci599/Fall2002/paper/DML2_streams-issues.pdf): probably the best overview of the early research in this area.
  - [High-Availability Algorithms for Distributed Stream Processing](http://cs.brown.edu/research/aurora/hwang.icde05.ha.pdf)
  - A couple of random systems papers:
    - [TelegraphCQ](http://db.cs.berkeley.edu/papers/cidr03-tcq.pdf)
    - [Aurora](http://cs.brown.edu/research/aurora/vldb03_journal.pdf)
    - [NiagaraCQ](http://research.cs.wisc.edu/niagara/papers/NiagaraCQ.pdf)
    - [Discretized Streams](http://www.cs.berkeley.edu/~matei/papers/2012/hotcloud_spark_streaming.pdf): This paper discusses Spark's streaming system.
    - [MillWheel](http://research.google.com/pubs/pub41378.html) is one of Google's stream processing systems.
    - [Naiad: A Timely Dataflow System](http://research.microsoft.com/apps/pubs/?id=201100)

Enterprise software has all the same problems but with different names, a smaller scale, and XML. Ha ha, just kidding. Kind of.

- [Event Sourcing](http://martinfowler.com/eaaDev/EventSourcing.html)—As far as I can tell this is basically the enterprise software engineer's way of saying "state machine replication". It's interesting that the same idea would be invented again in such a different context. Event sourcing seems to focus on smaller, in-memory use cases. This approach to application development seems to combine the "stream processing" that occurs on the log of events with the application. Since this becomes pretty non-trivial when the processing is large enough to require data partitioning for scale I focus on stream processing as a separate infrastructure primitive.
- [Change Data Capture](http://en.wikipedia.org/wiki/Change_data_capture)—There is a small industry around getting data out of databases, and this is the most log-friendly style of data extraction.
- [Enterprise Application Integration](http://en.wikipedia.org/wiki/Enterprise_application_integration) seems to be about solving the data integration problem when what you have is a collection of off-the-shelf enterprise software like CRM or supply-chain management software.
- [Complex Event Processing (CEP)](http://en.wikipedia.org/wiki/Complex_event_processing): Fairly certain nobody knows what this means or how it actually differs from stream processing. The difference seems to be that the focus is on unordered streams and on event filtering and detection rather than aggregation, but this, in my opinion is a distinction without a difference. I think any system that is good at one should be good at another.
- [Enterprise Service Bus](http://en.wikipedia.org/wiki/Enterprise_service_bus)—I think the enterprise service bus concept is very similar to some of the ideas I have described around data integration. This idea seems to have been moderately successful in enterprise software communities and is mostly unknown among web folks or the distributed data infrastructure crowd.

Interesting open source stuff:

- [Kafka](http://kafka.apache.org/) Is the "log as a service" project that is the basis for much of this post.
- [Bookeeper](http://zookeeper.apache.org/bookkeeper/) and [Hedwig](https://cwiki.apache.org/confluence/display/BOOKKEEPER/HedWig) comprise another open source "log as a service". They seem to be more targeted at data system internals then at event data.
- [Databus](https://github.com/linkedin/databus) is a system that provides a log-like overlay for database tables.
- [Akka](http://akka.io/) is an actor framework for Scala. It has an add on, [eventsourced](https://github.com/eligosource/eventsourced), that provides persistence and journaling.
- [Samza](http://samza.incubator.apache.org/) is a stream processing framework we are working on at LinkedIn. It uses a lot of the ideas in this article as well as integrating with Kafka as the underlying log.
- [Storm](http://storm-project.net/) is popular stream processing framework that integrates well with Kafka.
- [Spark Streaming](http://spark.incubator.apache.org/docs/0.7.3/streaming-programming-guide.html) is a stream processing framework that is part of [Spark](http://spark.incubator.apache.org/).
- [Summingbird](https://blog.twitter.com/2013/streaming-mapreduce-with-summingbird) is a layer on top of Storm or Hadoop that provides a convenient computing abstraction.

我会尽量跟上这个领域，因此，如果您知道我遗漏的一些内容，请告诉我。

我给你留下这样的信息：

https://youtu.be/2C7mNr5WMjA