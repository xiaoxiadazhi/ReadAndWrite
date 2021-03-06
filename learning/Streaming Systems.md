# Streaming Systems

## 内容导引

​	从概念上讲，本书有两个主要部分，每部分4章。**第一部分**是***Beam模型***（1~4章），致力于高阶的批式加流式数据处理模型，最初是为Google Cloud Dataflow开发的，后来作为Apache Beam贡献给了Apche基金会。

* **第一章**，***Streaming***。这一章涵盖了流式处理的基础知识，建立了一些术语，讨论了流式系统的能力，区分了两个重要的时间域（processing time 和 event time），最后看了一些通用的数据处理模式
* **第二章**，***The What, Where, When, and How of Data Processing***。这一章详细介绍了对无序数据进行鲁棒的流式处理的核心概念，每个数据都在具体运行示例的上下文中进行了分析，并且使用动画图表突出了时间维度。
* **第三章**，***Watermarks*** 。这一章提供了对时间进度指标的深入研究，它们如何被创建，他们如何通过管道传播。最后以检查两个实际的水位线实现来结束本章。
* **第四章**，***Advanced Windowing***。这一章捡起了第二章浅尝辄止的地方，深入一些advanced windowing和triggering的概念，例如processing-time窗口、sessions和continuation triggers。

​       在第一部分和第二部分之间，及时插入一些细节很重要，见**第五章**，***Exactly-Once and Side Effects***。这一章，他列举了提供端到端的exacty-once处理语义的挑战，并详细介绍了3种不同方式实现的exactly-once处理：Apache Flink， Apache Spark和Google Cloud Dataflow。

​	接下来是**第二部分**，***Streams and Tables***（6~9章），深入研究流式处理的低阶“流和表”的思考方式，最近由Apache Kafka社区的一些人推广的，不过 是几十年前数据库社区的人们发明的。

* **第6章**，***Streams and Tables***。这一章阐述流和表的基本思想，通过流和表的镜头分析经典的MapReduce方法，然后构建了一套足够通用的流和表的理论来完全包含Beam模型的方方面面。
* **第7章**，***The Practicalities of Persistent State***。 这一章考虑了在流管道中持久状态的动机，着眼于两种常见的隐式状态，然后分析了一个实际用户案例（广告属性）去说明一般状态管理机制的必要特征。
* **第8章**，***Streaming SQL***。这一章研究了流在关系代数和SQL上下文中的意义，对比了Beam模型和当前存在的经典SQL的固有的流和表的差异，并且提出一系列可能的方式去结合鲁棒的SQL中流的语义、
* **第9章**，***Streaming Joins***。这一章调研了各种类型的join，分析它们在流上下文中的行为，最后详细介绍了一个有用但不受支持的流式join用户案例：时间有效性窗口。

最后一张是**第10章**，***The Evolution of Large-Scale Data Processing***，概述了数据处理系统的MapReduce血缘历史，研究了将流式系统发展到今天这个样子的一些重要贡献。

作为最后一点指导，作者最希望读者能够从本书学到的是：

* 你能从本书学到的最重要的东西是：流和表的理论和他们之间的关系。其他所有一切都建立在这之上。我们在第6章之前都无法得到这个主题。但是它值得等待，之后你将会做出更好的准备去欣赏它的精彩。
* 时变关系是一个启示。它们是流式处理的具体化形式：流式计算建立用来获取一切的化身，从批量处理的世界到我们都知道和喜欢的熟悉的工具的强关联。我们第8章之前不会学习它们，但是同样的，这样的安排会帮助你更加欣赏它们。
* 一个写得好的分布式流式引擎是很魔幻的东西。这通常适用于分布式系统，但是随着你更多地了解这些系统如何被建立以提供它们表现出的语义（特别是第3章和第5章的研究），它们为你所做的将更加明显。
* LaTeX / Tikz是一个制作图表，动画或其他方面的神奇工具。



## Part I. The Beam Model

### 第一章 Straming 101

首先，我会介绍一些重要的背景信息来帮助构建我们将要讨论的其他主题。我分三步来做这件事：

* 术语（Terminology）
  * 为了准确地讨论负载的主题，我们需要准确的术语定义。对于当前使用中有过多解释的术语，我会试着确切地表达当我说它们时的含义
* 功能（Capabilities）
  * 我评论了流式系统经常出现的缺点。为了满足现代数据消费者未来的需求，我也提出了我认为数据处理系统建立者需要采纳的思想框架
* 时域（Time domains）
  * 我介绍了两个与数据处理相关的主要的时间域，说明了它们之间的关系，并且之处这两个时间域所带来的一些困难

#### 术语： 流是什么？

​	在进一步继续之前，我想先弄明白一件事：流是什么？当今，流这个术语常常意味着各种各样不同的东西，这可能会导致对流的真正含义或者流式系统实际上能做什么有些误解。因此，我更愿意在某些地方精确地定义这个术语。

​	问题的关键在于，很多东西应该被描述成*what* they are（无界数据处理，近似结果等），却逐渐被描述成它们历史上已经被怎样（*how*）完成（即通过流式执行引擎）。这种缺乏准确性的术语会影响流的真正含义，并且在某些情况下，它会给流失系统本身带来负担，暗示它们的能力仅限于历史上被描述成"streaming"的特征，例如近似或者推测结果。

​	鉴于精心设计的流式系统能和任何现有的批式引擎一样能够产生正确、一致、可重复的结果，我更倾向于将属于"streaming"隔离为非常具体的含义：

* 流式系统
  * 一种为无限数据集设计的数据处理引擎
  * A type of data processing engine that is designed with infinite datasets in mind 

如果我想讨论低延迟、近似或者推测的结果，我会使用那些特定的词，而不是不准确地称它们为"streaming"

​	准确的术语在讨论可能遇到的不同种类的数据时也是很有用的。从我的角度来看，有两个重要的维度来定义给定的数据集：*基数*和*构成*。

1. 数据集的基数表明了它的大小，基数最突出的方面是给定的数据集是有限还是无限的。下面有我更愿意用于描述数据集中粗粒度基数的术语

   * Bounded data（有界数据）
     * 一种大小有限的数据集合
   * Unbounded data（无界数据）
     * 一种大小无限的数据集合（至少在理论上）

   基数很重的原因是无界数据集的无界特性会给处理它们的数据处理框架带来额外负担。更多内容在下一节中介绍。

2. 数据集的constitution 表明它的物理表现。因此，constitution 定义了可以与问题中的数据交互的方式。在第6章之前我们不会深入研究constitution，但是为了简单介绍一下，有两个重要的主要constitution

   * Table
     * 在特定时间点的数据集合的整体视图。SQL系统按传统方式处理表
   * Stream
     * 数据集随着时间演进的逐元素的视图。数据处理系统的MapReduce血缘按传统方式处理流

​        我们将在第6/8/9章深入研究流和表的关系。在第8章中，我们还了解了将它们联系在一起的时变关系的统一基本概念。但在此之前，我们主要处理流，因为它是构成管道开发者在当今大多数数据处理系统（批+流）中直接交互的。它也是最自然地体现流处理独有的挑战的构成。

##### On the Greatly Exaggerated Limitations of Streaming

​	让我们谈一谈流式系统可以和不可以做什么，重点放在可以做什么上。在这一章我想了解的最重要的事情是精心设计的流式系统的性能如何。流式系统历来呗降级到一个niche市场，用来提供低延迟、不准确或推测的结果，通常与更强大的批示系统结合提供最终正确的结果，即Lambda架构。

​	对于那些还未熟悉Lambda架构的人来说，其基本思想是，在批式系统旁边运行流式系统，两者执行相同的计算。流式系统给你低延迟、不准确的结果（要么是因为祭祀算法，要么是因为流式系统本身不提供正确性），并且在一段时间后，批式系统一路回滚并为你提供正确的输出。最初由Twitter的Nathan Marz（storm的创建者）提出，它最终取得了相当的成功，因为事实上在当时它是一个很棒的想法。流式引擎在正确性方面让有有点失望，批式引擎又很笨重，所以Lambda给了你一种能够得到想要结果的方式。但是很不幸，维护Lambda系统非常麻烦：你需要构建、配置和维护两个独立版本的管道，然后以某种方式合并最后两个管道的结果。

​	作为花费数年时间研究强一致性流式系统的人，我也发现Lambda架构的整个原理有点令人讨厌。不出所料，当Jay Kreps的“Questioning the Lambda Architecure”论文发表出来的时候，我是它的超级粉丝。这是第一个高度可见反对双模执行必要性陈述之一。Kreps使用Kafka这样的可重放数据的系统来作为流相互连接的桥梁，解决了流式系统可重复性的问题。甚至提出了Kappa体系结构，这意味着一个设计良好的系统仅需要使用运行一个管道，且这个系统是针对当前的工作而构建的。 我原则上完全支持这个概念。

​	坦率地说，我会更进一步。我认为设计良好的流式系统实际上提供了批量功能的严格超集，即设计良好的流式系统完全可以处理批量任务。感谢Apache Flink成员把这个想法铭记于心，并建立了一套all-streaming-all-the-time系统，即使是在批量模式下也是如此。

```
批式和流式效率差异
```

所有这一切的必然结果是：流式系统的广泛成熟，加上用于无限数据处理的健壮框架，最终将使得Lambda体系架构成为大数据历史上的老古董。我相信是时候实现这个目标了。因为这样做（即在Streaming自己的模式下击败Batch）你只需要两个东西：

1. Correctness（准确性）

   * 这方面streaming可以做到和批式相同的效果。核心在于，正确性归结于一致的存储。流式系统需要一个方法随时间推移来对checkpoint状态进行持久化（Kreps在他的“Why local state is a fundamental primitive in stream processing”论文中讨论的东西），并且它必须被设计足以在机器故障的情况下保持一致。几年前，当Spark Streaming首次出现在公共的大数据场景中，它就是黑暗的streaming世界中的一座灯塔，尤其是一致性方面。所幸，从那以后事情有所改善，但是很多流式系统仍然试图在没有强一致性的情况下获得成功。

   * **重申——因为这一点十分重要：exactly-once处理需要强一致性，正确性也需要强一致性，任何有机会满足或者超出批式系统能力的系统都需要强一致性**。除非你真的不关心你的结果，否则我恳请你避开所有不提供强一致性状态的流式系统。批式系统不需要你提前去验证它们是否能够生成正确的结果，不要浪费时间在那些不能满足相同标准的流式系统上。

   * 如果你想了解更多有关在流式系统中获取强一致性的所需的内容，我建议你查看MillWheel、Spark Streaming、Flink snapshotting论文。这3篇文章都花费了大量时间讨论一致性。Reuven将在第5章中深入探讨一致性保证，如果你仍然渴望更多内容，那么在文献和其他地方有关于该主题的大量高质量信息。

     [MillWheel]: 
     [Spark Streaming]: 

     [Flink snapshotting]: https://arxiv.org/pdf/1506.08603.pdf

2. Tools for reasoning about time

   * 这方面streaming将超越batch。好的时间推理工具对于处理event-time偏差不固定的无界、乱序的数据至关重要。越来越多的现代数据集表现出这些特征，而现有的批式系统（以及很多流式系统）缺少必要的工具去应对它们引入的问题。我们将花费本书的大部分来解释和关注这点的方方面面。
   * 首先，我们会对时域的重要概念有个基本了解，然后，我们会深入研究我说的处理event-time偏差不固定的无界、乱序的数据的含义。之后，我们花费本章的剩余部分通过使用批式和流式系统，来研究有界和无界数据处理的常用方法。

##### Event Time Versus Processing Time 

​	要谈论无界数据处理，需要对涉及的时间域有个清楚的理解。在任何数据处理系统中，我们通常关心两个时间域：

* event time
  * 事件真实发生的时间
* processing time
  * 系统中观察到事件的时间

​        不是所有的用例都关心event time，但是很多都需要关心。比如包含表征随时间推移的用户行为、大部分交易应用和很多类型的异常检测在内的例子。

​	理想情况下，event time和processing time总是相等的，事件已发生就立马被处理。然而，现实并非如此，event time和processing time之间的偏差不仅非0，而且通常是底层输入源、执行引擎和硬件的特性的高可变函数。可能影响偏差的事情包括：

* 共享资源限制，如网络拥塞，网络分区，或者非专用环境中共享的CPU
* 软件引起，如分布式系统逻辑、争用等等
* 数据本身特点，如key分布，吞吐量变化，或者无序的变化

​         因此，如果你在任何现实世界的系统汇总绘制event time和processing time的进度，你通常会得到类似于图1-1的红线的内容。

![1-1](..\picture\streaming\1-1.png)

图1-1中，斜率为1的黑色虚线代表理想值，也就是processing time和event time相等；红线表示真实值。这个例子中，processing time开始时滞后一些，中间又偏向理想值，然后又再次滞后一点。乍一看，这个图有两种skew，每种都在不同的时域中：

* processing time
  * 理想线和红线之间的垂直距离是processing-time域的滞后。这个距离告诉你在事件的发生时间和被处理时间之间所观测到的延迟。
* event time
  * 理想线和红线之间的水平距离是此时管道中event-time的偏移量。它告诉你当前管道落后理想值多久。

​        实际上，在任何给定点，processing-time lag和event-time skew是相同的，它们仅仅是观察同一件事的两种方式。关于lag/skew重要的内容是：由于事件时间和处理时间之间的整体映射不是静态的（即，lag/skew可能随时间任意变化），这就意味着如果你关心它们的事件时间（即实际发生事件的时间），就无法单独在你的管道观测到数据时的上下文中分析你的数据。不幸的是，这是很多系统为无界数据设计的方式。为了应对无界数据集的无限性，这些系统通常提供一些窗口化输入数据的概念。我们稍后会更深入地讨论窗口化，但它本质上是沿着时间边界把数据集切成有限的块。如果你关心正确性，并且有兴趣在事件时间的上下文中分析你的数据，你就不能像很多系统那样用processing time（即，processing-time窗口化）去定义那些时间边界；在处理时间和事件时间之间没有一致的关连的话，一些event-time数据将会在错误的processing-time窗口中结束（因为分布式系统固有的lag，多种输入源的在线/离线特性等等），然后缺乏正确性。我们将在接下来的章节通过一些例子来更详细地研究这个问题。

​	不幸的是，当按事件时间窗口化时，上图也不够乐观。在无界数据的上下文中，乱序和可变的skew会导致event-time窗口的完整性问题：缺少可预测的处理时间和事件时间之间的映射，你如何确定你何时观察到了给定事件时间X的所有数据？对于很多现实世界的数据源，你根本做不到。但是当今使用的绝大多数处理系统都依赖一些完整性概念，这使得它们在应用于无界数据集时处于严重劣势。

​	我建议不要试图去把无界数据切成最终完整的信息的有限批次，而应该设计工具，使我们能够生活在这些复杂数据集带来不确定性的世界中。新数据将到达，旧数据可能被撤销或者更新，我们构建的任何系统都应该能够独立处理这些事实，完整新概念应该是对特定和适当的用例的方便的优化，而不是语义上的必要。

​	在详细介绍这种方法可能是什么样子之前，让我们完成一个更有用的背景：常见的数据处理模式。

#### Data Processing Patterns

​	这个时候，我们已经建立了足够的背景知识，可以开始研究当今有界和无界数据处理常见的核心类型的使用模式。我们将在我们关系的2种主要引擎（批式和流式，我把microbatch当做streaming来讨论，因为在这个级别二者的区别不是很重要）的上下文中研究这两种处理方式

##### Bounded Data

​	处理有界数据	在概念上很简单，而且每个人都很熟悉。图1-2中，左边数据集由熵填满。我们通过一些如MapReduce的数据处理引擎（通常是批处理，尽管涉及良好的流式引擎也可以工作得很好）运行它，得到右边的具有更大内在价值的结构化数据集。

![1-2](..\picture\streaming\1-2.png)

​	虽然作为此方案的一部分，您实际可以计算的内容当然有无限变化，但整体模型非常简单。更有趣的是处理无界数据集的任务。现在让我来研究通常处理无界数据的各种方式，从传统的批式引擎使用的方法开始，到最后的你可以采用的专门为无界数据设计的方法，如大多数流式或者微批引擎。

##### Unbounded Data：Batch

​	虽然没有明确为无界数据设计，但批式引擎还是被用来处理无界数据集，因为批式系统是最先被构思的。如你所料，这种方法思想就是把无界数据切分成适合批处理的有界数据集的集合。

###### Fixed windows

​	使用批式引擎的重复运行来处理无界数据集最常见的方法就是：把输入数据窗口化成固定大小的窗口，然后将每个窗口作为一个独立、有界的数据源来处理（有时也称为翻滚窗口（*tumbling windows*）），如图1-3。尤其对于输入源如日志这种，事件可以被写入目录或者文件层次结构中，这些目录和文件层次结构的名称会对它们所对应的窗口进行编码，这种东西乍一看十分简单，因为您基本上已经执行了基于时间的shuffle来获取数据。

​	然而，实际上大多数系统仍然存在完整性的问题需要解决（如果因为网络分区导致一些事件在路由过程中延迟了，会怎么样？如果你的事件是被全局收集的，并且在处理之前必须转移到公共位置，会怎么样？如果你的事件来自移动设备会怎么样？），这就意味着某种缓和措施是必须的（例如，延迟处理直到你确认你已经收到所有事件，或者无论数据迟到何时都重新处理给定窗口整个批次）

![1-3](..\picture\streaming\1-3.png)

###### Sessions

​	当你试图使用批式引擎将无界数据处理成更复杂的窗口化策略时（sessions），这种方式更加糟糕。会话通常被定义为由不活动间隙终止的活动周期。当使用典型的批式引擎来计算会话时，你常常会遇到被跨批次分割的会话，如图1-4中红色标记所示。我们可以通过增加批次的大小来减少分割的数量，但代价是会增加延迟。另一种选择是添加额外的逻辑来拼接先前运行的会话，但代价是进一步复杂化。

​	无论哪种方式，使用经典的批式引擎去计算会话都不够理想。更好的方式是以流的方式建立会话，后面我们会研究。

##### Unbounded Data：Sreaming

​	与大多数基于批的无界数据处理的方式的临时性质相反，流式系统就是为无界数据建立的。如我们之前讨论的，对于很多现实世界的分布式输入源，你会发现你不仅需要处理无界数据，还需处理以下数据：

* 在事件时间方面高度乱序（Highly unordered with respect to event times ），意味着如果你想在这些数据发生的环境下分析数据，你就需要在管道中进行一些基于时间的shuffle
* Of varying event-time skew，意味着你不能仅仅假设你将总是在某个恒定时间Y中看到给定的事件时间的大部分数据

在处理具有这些特征的数据时，您可以采取一些方法。我通常把这些方法分为4组：

* time-agnostic
* approximation
* windowing by processing time
* windowing by event time 

现在让我们花点时间来研究这几种方法。

###### Time-agnostic

​	Time-agnostic处理用于时间基本无关的情况；也就是说，所有相关逻辑都是数据驱动的。因为有关此类用例的所有内容都是由更多数据的到来决定的，所以除了基本数据传输之外，流引擎实际上没有什么特别需要支持的。因此，基本上所有流式系统都支持开箱即用的时间不可用的用例（当然，如果你关心正确性，模块系统到系统的一致性保证差异）。批式系统也适用于无界数据源time-agnostic处理，可以通过把无界数据源切分成任意序列的有界数据集，然后独立处理这些数据集。我们在本节中研究一些具体的例子，但考虑到Time-agnostic处理的简单性（至少从时间角度来看），我们不会花太多时间在它上面。

###### *Filtering*

​	Time-agnostic处理的一种非常基本的形式就是filtering，图1-5中呈现的例子。加入你要处理web流量日志，你想过滤掉不是来源于某个特定域名的所有流量。你将观察每个到达的记录，看是否属于那个域名，如果不是就丢掉。由于这种情况在任意时间都仅仅只是依赖单个因素，因此数据源是无界、乱序和变化的event-time skew是无关紧要的。

![1-5](..\picture\streaming\1-5.png)

###### *Inner joins*

​	另一个时间无关的例子是inner join，如图1-6。当join两个无界数据源时，如果你只关心来自于两个数据源的元素到达后join的结果，则没有逻辑中没有时间因。当一个数据源的值来了以后，你可以简单地把它缓存起来，只有来自于另一个数据源的值到达以后，才需要发出join后的记录。（事实上，你可能想要一些垃圾回收策略去回收未发送部分的连接，这可能是基于时间的。但是对于基本没有未完成join的用例来说，这应该不是问题因

![1-6](..\picture\streaming\1-6.png)

​	将语义切换到一些外连接会有我们之前讨论的数据一致性问题：在你已经看到该连接的一方后，你如何知道另一方是否会到达？你没法知道，说实话你需要引入timeout的概念，这将引入时间因素。时间因素本质上是窗口化的一种形式。

###### *Approximation algorithms*

​	第二种主要的方法是近似算法，如近似Top-N，streaming k-means等。它们的输入是无界数据源，输出数据看起来或多或少有点像你想要的得到结果，如图1-7。近似算法的有点在于，开销低并且为无界数据设计。缺点是算法通常比较复杂，并且算法的近似性质限制了它们的效果。

![1-7](..\picture\streaming\1-7.png)

​	值得注意的是，这些算法的设计中通常包含一些时间元素（比如某种内置衰变）。由于它们在元素一到达时就开始处理，所以时间元素通常是基于processing-time的。这对于在其近似上提供某种可证明的误差界限的算法尤其重要。如果那些误差界限在数据有序到达时是可预测的，那么当您向算法提供具有变化event-time skew的无序数据时，它们基本上没有任何意义。

​	近似算法本身是一个引人入胜的主题，但由于它们本质上是时间不可知处理的另一个例子（以算法本身的时间特征为模），因此我们可以非常直接地使用它们，因此，鉴于我们目前的重点，它们不值得进一步关注。

###### Windowing

​	剩下两种无界数据处理的方法都是窗口化的变形。在深入研究这二者的区别之前，我应该清楚地说明白窗口化的确切含义，因为我们在上一节中只是简单地提到了它。窗口化只是获取数据源的概念（要么是无界的，要么是有界的），根据时间把数据源切分成多个有限的数据块来处理。图1-8展现了3种不同的窗口模式。

![1-8](..\picture\streaming\1-8.png)

​	让我们仔细研究下每种策略：

* Fixed windows (aka tumbling windows)/固定窗口（又叫翻滚窗口）
  * 我们之前讨论过固定窗口。固定窗口将时间切为固定时长的分段。通常（如图1-9显示），固定窗口的分段被均匀地应用于整个数据集，这就是对齐（*aligned*）窗口的一个例子
* Sliding windows (aka hopping windows)/滑动窗口（又称跳跃窗口）
  * 滑动窗口是固定窗口的一般化形式，它具有固定长度和固定周期。如果周期小于长度的话，则窗口会重叠。如果周期和长度相等，你就得到了固定窗口。如果周期大于长度，则会得到一个奇怪的采样窗口，该窗口仅查看数据的子集随时间的变化。与固定窗口一样，滑动窗口通常是对齐的，但在某些用例中它们可能会因为性能优化而变成不对齐的。请注意，图1-8中的滑动窗口是按原样绘制的，以提供滑动感; 实际上，所有五个窗口都将应用于整个数据集。
* Sessions 
  * 会话是动态窗口的例子，会话是由大于某个超时的不活动间隙终止的事件序列组成。会话通常用于分析随时间变化的用户行为，通过将一系列时间相关的事件分成一组。会话很有意思，因为它们的长度不能被先验地定义，它们是取决于涉及到的实际数据。它们也是未对齐窗口的规范示例，因为会话实际上在不同数据子集（例如，不同用户）之间从不相同。

​        我们之前讨论的两个时域（处理时间和事件时间）基本是我们最关心的。窗口化在这两个时域中都有意义，所以让我们详细研究下它们，然后理解如何区分它们。因为processing-time窗口在历史上比较常见，我们从它开始。

###### *Windowing by processing time*

​	当根据处理时间来窗口化时，系统实际上是把输入数据缓存到窗口中，知道经过一定量的处理时间。比如，有个长度为5min的固定窗口，系统将数据缓存5分钟的处理时间，之后它会将在这五分钟内观察到的所有数据视为一个窗口并将它们发送到下游进行处理。

![1-9](..\picture\streaming\1-9.png)

​	处理时间窗口化有几个好的特性：

* 它很简单。其实现极其简单，因为你不用担心随时间推移的shuffling数据。你只要在它们到达时把它们缓存起来，然后再窗口关闭时将它们发往下游。
* 判断窗口完整性很简单。由于系统完全知道一个窗口的所有输入是否都已被看到，所以它可以对给定窗口是否完成做出完美的决定。这意味着根据处理时间来窗口化时是不需要能够处理“迟到”数据的。
* 如果您想要根据观察到的数据推断信息源，processing-time窗口化正是你想要的。许多监控方案都输入这一类。想象一下，跟踪每秒发送到一个全球的web服务的请求数。为了检测中断而计算这些请求的速率是处理时间窗口的完美使用。

除了优点之外，processing-time窗口化有一个非常大的缺点：

* 如果相关数据具有事件时间，那么如果processing time窗口要反映这些事件实际发生的时间，则这些数据必须按照event time顺序到达。

不幸的是，在许多实际的分布式输入源中event time有序的数据并不常见。

​	举个简单的例子，想象一下，任意一个收集使用的统计数据以供后续处理的移动app。如果给定的移动设备脱机一段时间（），在这期间被记录的数据将不会被上传，直到该设备再次联机才会上传。这意味着数据可能以分钟、小时、天、周或更长时间的 event time skew到达pipeline。在使用processing time窗口化时，基本上不可能从这样的数据集中得出任何有用的推论。

​	另一个例子是，很多分布式输入源在整体系统健康的时候，*似乎*可以提供event-time有序的数据。不幸的是，事实上健康时，虽然输入源的event-time skew很低，但并不意味着它将始终保持这种状态。考虑一个全球服务，它处理多个洲收集来的数据。 如果网络在带宽受限的跨大陆线路上出现问题(遗憾的是，这种情况非常普遍)会进一步降低带宽和/或增加延迟，那么你的一部分输入数据可能会突然出现比之前更大的skew而到达。如果你是根据处理时间来窗口化的那些数据，那么你的窗口不再代表其所对应时间段内实际发生的数据；相反，它们代表事件到达处理管道时的时间窗口，这是一些任意组合的旧数据和当前数据。

​	在这两种情况下，我们真正想要的是通过event time来窗口化数据，这种方式对事件到达的顺序是可以保证的， 即我们真正想要的是event time窗口。

###### *Windowing by event time*

​	event time窗口是在需要以有限个时间块来观察数据源时使用的窗口，这些数据源反映了这些事件实际发生的时间。这是划分窗口的黄金标准。2016年之前，大多数在使用的数据处理系统都缺少对事件时间的原生支持（即使任何具有良好一致性模型的系统，如Hadoop或Spark Streaming 1.x，都可以作为构建这种窗口化系统的合理基础）。我很高兴地说今天的世界看起来非常不同，有多个系统，从Flink到Spark，Storm到Apex，都原生支持某种事件时间窗口。

​	图1-10展示了把无界数据源窗口化成时长为1小时的固定窗口的例子

![1-10](..\picture\streaming\1-10.png)

​	图1-10中的黑色箭头显示的是在processing-time窗口到达的数据，它们所在的processing time窗口与它们所属的event time窗口不匹配。因此，如果这些数据被窗口化到processing time窗口中，而processing time窗口又与event time有关，那么计算出来的结果就不正确了。因此使用event-time窗口的一个好处就是，保证event-time的正确性。

​	对无界数据源进行event-time窗口化的另一个好处是，你可以创建动态大小的窗口，比如sessions， 没有像之前那种，在固定窗口上生成session时所产生的对session的任意分割（正如前面session例子中所看到的）。

![1-11](..\picture\streaming\1-11.png)

数据被收集到会话窗口，根据相应事件发生的时间捕获活动的突发事件。白色箭头再次指出将数据放入正确的event time位置所需的时间shuffer

​	当然，强大的语义是需要代价的，event-time窗口也不例外。因为窗口的生存时间(在处理过程中)通常比窗口本身的实际长度长（因为会有迟到的数据），event-time窗口有两个明显的缺点：

* Buffering
  * 由于窗口寿命的延长，需要缓存更多的数据。所幸，持久化存储通常是大多数数据处理系统所依赖的资源类型中最便宜的一种（其他的主要是CPU、网络带宽和RAM）。因此，在使用任何设计良好的数据处理系统(具有强一致的持久状态和良好的内存缓存层)时，这个问题通常不像人们想象的那么严重， 此外，许多有用的聚合不需要缓冲整个输入集(例如，sum或average)，而是可以使用存储在持久化状态中的更小的中间聚合数据增量地执行。
* Completeness
  * 鉴于我们通常没有好的办法知道何时看到了给定窗口的所有数据，那么我们怎么知道何时能得到该窗口的结果呢？事实上，我们无法知道。对于许多类型的输入，系统可以通过诸如MillWheel，Cloud Dataflow和Flink（我们在第3章和第4章中讨论的更多）中的watermark之类的东西给出窗口完整性的合理准确估计。但对于要求绝对正确的例子（例如计费），唯一真正的选择是为管道构建器提供一种方法，以便在pipeline builder想要物化windows的结果时表达这些结果。

#### Summary

​	在详细研究Beam Model方法之前，让我们简单回顾一下到目前为止所学的内容。本章，我们完成了：

* 澄清了术语，将"streaming"的定义聚焦于无界数据的系统建立上，同时使用更多的描述性术语（如近似/推测的结果），这些不同概念的术语通常被分类到"streaming"的范畴
* 评估了设计良好的批式系统和流式系统的相对功能，断定实际上流式是批式的严格超集，类似于Lambda架构的概念（认为基于流式不如基于批式），随着流式系统的成熟，注定要被淘汰
* 提出了流式系统赶上并最终超过批式系统所必须的两个高级概念，分别是正确性和时间推理工具
* 确定了事件时间和处理时间之间的重要差异，描述了在数据发生时分析数据时那些差异所带来的困难，并提出了一种从完整性概念转向简单适应数据随时间变化的方式
* 研究了当今普遍用于有界和无界数据的主要数据处理方式，通过批式和流式引擎，大致将无界方式分成：time-agnostic，approximation，windowing by processing time和windowing by event time

​        接下来，我们深入研究Beam Model的细节，从概念上看看如何通过4个相关轴来分解数据处理的概念：what、where、when、how。我们还详细研究了处理多个场景中的简单、具体的示例数据集，突出Beam模型启用的多个用例，以及一些具体的APIs以实现我们的基础。这些示例将有助于推动本章介绍的事件时间和处理时间的概念，同时另外探索水印等新概念。

### Chapter 2. The What, Where, When, and How of Data Processing 

第一章主要关注三个方面：

* 术语。准确定义当我使用类似于"streaming"的overloaded术语我是指什么意思；
* 批式和流式系统。对比两种系统理论上的能力，并假设了流式系统超过批式系统所必须的两个东西：准确性和时间推理工具；
* 数据处理模式。研究使用批式和流式系统处理有界和无界数据的方式

本章，我们将通过实具体示例进一步关注第一章中的数据处理模式。当我们看完时，我们将涵盖我所认为的鲁棒的乱序数据处理所需的核心原则和概念集；这些是真正让您超越经典的批处理的推理时间的工具。

​	为了让您了解操作中的内容，我使用Apache Beam代码的片段，再加上延时图，以提供概念的直观表示。Apache Beam是用于批处理和流处理的统一编程模型和可移植层，具有各种语言（例如，Java和Python）的一组具体SDK。然后，可以在任何受支持的执行引擎（Apache Apex，Apache Flink，Apache Spark，Cloud Dataflow等）上移植运行使用Apache Beam编写的管道。

​	我在这里使用Apache Beam的例子不是因为这是一本Beam书（它不是），而是因为它最完整地体现了本书中描述的概念。回溯到最初编写“Streaming 102”时（当它仍然是来自Google Cloud Dataflow的数据流模型而不是来自Apache Beam的Beam Model时），它实际上是现存唯一的提供了我们将在这里介绍的所有例子所必须的表达能力的系统。一年半之后，我很高兴地说已经发生了很大变化，并且大多数主要系统已经朝着支持看起来很像本书中描述的模型的方向发展。

#### Roadmap

​	为了帮助为本章奠定基础，我想列出将支持其中所有讨论的五个主要概念，实际上，对于第一部分的大部分内容，我们已经涵盖了其中的两个。

​	在第一章中，我首先确定了event time（事件发生时间）和processing time（处理过程观察到事件的时间）的主要区别。这为本书中提出的主要论点之一提供了基础：如果您关心事件实际发生的正确性，则必须分析与其固有事件时间相关的数据，而不是分析过程中遇到它们的处理时间。

​	之后我介绍了*windowing*的概念（即，沿着时间边界划分数据集），它是用于应对无界数据源技术上可能永远不会终止的事实的一种常见的方式。窗口化策略的简单一点的例子是*fixed*和*sliding*窗口，复杂一点的窗口类型，如会话(窗口由数据本身的特征定义，例如根据不活动的间隙来捕获每个用户惠东的会话)也被广泛使用。

​	除了这两个概念之外，我们现在来学习另外3个：

* Triggers
  * 触发器是一种由一些外部信号触发，来表明何时计算某个窗口结果的机制。触发器可以让我们灵活地选择何时发送输出。在某种意义上，你可以把触发器当做表明何时计算输出的流控机制。作用跟照相机的快门相同，按下去，就能拿到某个时间点计算结果的快照
  * 触发器还能够在数据不停到达的过程中多次观察某个窗口的输出。这样可以在数据到达时提供推测结果，以及随着时间的推移处理上游数据（修订）的变化或迟来的数据。
* Watermarks
  * watermark是有关于事件时间的输入完整性的概念。时间值为X的水位线表示：所有事件时间小于X的输入数据都会被观测到。因此，当观察不知何时结束的无界数据时，watermark可以作为进度的一项指标。
* Accumulation
  * accumulation模式表示同一个窗口观测到的多个结果之间的关系。这些结果可能完全独立；也就是说，它们的增量是独立的，或者它们之间可能重叠。不同accumulation模式有不同的语义和计算成本，适用于不同场景

​        我们通过回答以下4个问题这样来重新审视旧的概念并探索新的概念，因为我认为这样会让我们更容易理解这些概念之间的关系，我提出的这4个问题对每一个无界数据处理问题都很重要。

* *what* results are calculated?  
  * 计算什么结果？这是用户在管道中定义的转换类型决定的，包括诸如求和、构建直方图、训练机器学习模型等等。这也是批处理解决的经典问题。
* *where* in event time are results calculated?  
  * 在事件时间的哪个地方计算结果？这是用户在管道中使用的event-time窗口决定的。这包括第一章中常见的窗口化例子（fixed、sliding、sessions）；似乎没有窗口化的概念的用例（如，time-agnostic处理；经典的批式处理一般也被分为此类）；和更复杂的窗口化，如限时拍卖。另外注意，如果将入口时间指定为记录到达系统时的事件时间，那么也可以包括处理时窗口
* *when* in processing time are results materialized? 
  * 在什么处理时间点输出结果？使用触发器和watermark可以解决这个问题。这个主题有无限变种，不过最常见的是那些涉及重复更新（即物化视图语义）的场景，和那些在相应输入被认为是完整的以后利用watermark为每个窗口提供一个输出的场景（即，传统的批式处理语义应用于每个窗口的基础上），或者两者结合
* *how* do refinements of results relate? 
  * 如何改进相关结果（更新结果）？这个问题可以使用三种方式来解决：discarding（其中结果都是相互独立和不同的），accumulating（迟到的数据会加到之前的结果），accumulating和retracing（累加值和先前触发的值的撤销都会被发送）

我们将在本书剩余部分详细研究这些问题。

#### Batch foundations：*what* and *where*

​	批处理的基础：*what*和*where*

​	我们第一站是：批式处理。

##### *What*： Transformations

​	应用于经典批式处理的transformations（变换）回答了问题"What results are calculated? " 即使您可能已经熟悉经典的批处理，我们仍然会从它开始，因为它是我们添加所有其他概念的基础。

​	本章的剩余部分，我们来看一个例子：计算一个由9个值组成的数据集的整数和。假设我们已经编写了一个基于团队的手机游戏，我们希望建立一个管道，通过对用户手机报告的个人得分求和来计算团队得分。如果我们要在名为“UserScores”的SQL表中捕获我们的九个示例分数，它可能看起来像这样：

![1-12](..\picture\streaming\1-12.png)

​	注意本例中所有分数都来自于同一个团队的成员；考虑到我们的图中的维度数量有限，这是为了让例子简单。另外，由于我们按team分组，我们真正关心的只有最后3列：

* Score
  * 成员得分
* EventTime
  * 得分的事件时间，也就是得到该分数的时间
* ProcTime
  * 得分的处理时间，也就是该分数在管道中被观察到的时间（数据进入系统被处理的时间）

对于每个示例管道，我们将看一个数据随时间变化的时延图。这些图表在我们关心的两个时间维度上绘制了我们的九个分数：x轴的事件时间和y轴的处理时间。图2-1展示了输入数据的静态图。

![2-1](..\picture\streaming\2-1.png)

​	随后的延时图是动画（Safari）或一系列帧（打印和所有其他数字格式），让您可以看到数据随着时间变化是如何被处理的（在我们得到第一张时延图后不久会有更多内容）

​	为了让管道的定义更具体，我们在每个示例之前放置了Apache Beam Java SDK伪代码的简短片段。为了让示例更清晰，我有时候会省略一些细节（如使用具体的I/O源），或者简化名称（Beam Java 2.x及更早的版本中的触发器名称太冗长了，为了清晰我简化了名称）。

​	如果你已经对Spark或者Flink很熟悉了，你应该很容易理解Beam代码在做什么。Beam中有两个基本操作

* PCollections

  * 可以被并发处理的数据集合。名字前面的P代表perform

* PTransforms

  * 用于从PCollections创建新的PCollections的操作。PTransforms可以对逐个元素执行转换，可以把多个元素分组/聚合到一起，或者它们还可以和其他PTransforms进行组合，如图2-2.

  ![2-2](..\picture\streaming\2-2.png)

​        出于我们示例的目的，我们先假设一个预加载的名为"input"的PCollection<KV<Team, Integer>>  。现实世界中，我们将从某个I/O源把原始数据（如日志记录）读入PCollection<String>  ，然后通过解析成键值对的方式把它变换成PCollection<KV<Team, Integer>>。

​	因此，对于这样一个管道：从某个I/O源简单读取数据，解析team/score键值对，和计算每组的总分，我们得到Example2-1所示：

![Example2-1](..\picture\streaming\Example2-1.png)

​        从I/O源读取键值数据，Team为key，Integer为value。每个key的values都会被加到一起生成按key求和的输出集合。

​	对于接下来所有的示例，我们在看到一段描述我们正在分析的管道的代码片段之后，我们将看到一个实验图，显示该管道在具体的数据集上的单个key的执行。在一个真正的管道中，你可以想象类似的操作将在多台机器上并行发生，但是我们这里尽量保持简单。

​	如前所述，Safari版本将完整的执行呈现为动画电影，而打印和所有其他数字格式使用静态的关键帧序列，以便了解管道随时间推移的进展情况。在这两种情况下，我们还在www.streamingbook.net上提供了完整动画版本的URL

​	每个图表绘制输入和输出都是通过两个维度：event time（在x轴上）和processing time（在y轴上）。在processing-time轴上随时间推移的黑线，表示从底部推移到顶部的管道观察到数据的真实时间。输入是圆圈，圆圈内的数字表示该记录的值。 它们从浅灰色开始，当管道观察到它们之后就会变暗。

​	当管道观察到值时，它把这些值累加到中间状态，最终把聚合结果最为输出。状态和输出由矩形表示（灰色表示状态，蓝色表示输出），聚合值靠近顶部，矩形覆盖的区域表示事件时间和处理时间累加到结果中的部分。当在典型的批式引擎上执行时，Example 2-1中的管道如图2-3所示？

![2-3](..\picture\streaming\2-3.png)

由于这是批式管道，所以它会累加状态一直到看到所有输入，这个时候它产生了值为48的输出。这个例子中，由于我们没有应用任何制定的窗口化变换去基于所有事件时间求和，所以状态和输出的矩形覆盖了整个X轴。然而，如果我们想处理无界数据源，经典的批式处理将无法满足，我们无法等待输入结束，因为它实际上永远不会结束。我们想要的概念之一是第一章中介绍的窗口化，因此，在我们第二个问题——“Where in event time are results calculated?” 的背景下，我们将简要回顾窗口化。

##### *Where*：windowing

​	如第一章中讨论，窗口化是把数据在时间上做切分。常见的窗口化策略包括固定窗口、滑动窗口和会话窗口，如图2-4：

![2-4](..\picture\streaming\2-4.png)

​	为了更好地了解实际中窗口化的样子，来看一下上面的整数求和管道，并把它划分为时长为2分钟的固定窗口。在Beam中，其改变只是简单地加上`Window.into`变换，Example 2-2有高亮显示：

![Example2-2](..\picture\streaming\Example2-2.png)

​	回想一下，Beam提供了可以在批式和流式处理中工作的统一模型，因为在语义上批式处理只是流式处理的子集。因此，首先在批处理引擎上执行此Pipeline; 在批处理引擎上执行此Pipeline，原理简单明了，可以很好的跟在流处理引擎上执行做对比。图2-5展示了结果。

​	如前所述，输入不断地被累加到状态中，直到它们被完全消费，然后产生输出。然而，在这种情况下，我们得到的不是一个输出，而是4个：在时间上切分成了4个事件时间窗口，每个窗口产生一个输出。

​	现在我们已经回顾了第一章中介绍的两个主要概念：event-time和processing-time域之间的关系，和windowing。如果想更进一步，我们需要开始添加本节开头提到的新概念：Watermark，trigger和accumulation 。

#### Going Straming：When and How

​	我们刚才观察了在批式引擎上的窗口化pipeline的执行。但是，我们的理想期望是：结果有更低的延迟、能原生处理无界数据源。切换到流式引擎是朝着正确方向迈出的一步，但是我们之前等到全部输入都被消费完再产生输出的策略不再具有可行性。进入triggers和watermarks。

##### When: The Wonderful Thing About Triggers Is Triggers Are Wonderful Things! 

​	触发器回答了问题“When in processing time are results materialized? ”触发器声明了在处理时间中某个窗口的结果何时被输出。某个窗口的每个指定的输出被称为窗口的一个*pane*（窗格）。

​	虽然触发语义可能十分广泛，但是从概念上讲只有两类普遍使用的触发器，实际的应用程序几乎总是使用其中的一个或二者的组合：

* Repeated update triggers（重复更新触发器）
  * 这类触发器会定期生成更新过的窗口窗格。这些更新可能发生在每条新纪录到达时，也可能发生在一些处理时间延迟后，比如一分钟一次。对重复更新触发器的周期的选择主要是平衡延迟和成本的经验值。
* Completeness triggers（完整性触发器）
  * 这类触发器只有在窗口的输入被认为是完整的之后，才会输出一个窗格（pane）。这种类型的触发器与我们在批处理中熟悉的类似：只有在输入是完整的之后，我们才提供结果。基于触发器的方式的不同之处在于，完整性概念是作用于单个窗口，而不是受限于整个输入的完整性。

​        重复更新触发器是流式系统中最常见的触发器。它们容易实现、容易理解，并且为一种特定类型的用例提供了有用的语义：对可无话的数据集的重复（并最终一致）的更新，类似于数据库世界中物化视图所获得的语义。

​	完整性触发器很少遇到，但提供的流式语义更接近经典批处理世界的流式语义。它们还提供了推理如缺失数据和迟到数据之类的工具，这两者我们稍后都会在探索驱动完整性触发器的基础原语（watermarks）时讨论。

​	但是首先，让我们从简单的开始，看看一些基本的重复更新触发器。为了让触发器的概念更加具体，让我们继续向我们的示例pipeline中添加最简单的触发器：每来一条新纪录都会触发的触发器，如Example 2-3所示。

```java
Example 2-3. Triggering repeatedly with every record

PCollection<KV<Team, Integer>> totals = input
.apply(Window.into(FixedWindows.of(TWO_MINUTES))
.triggering(Repeatedly(AfterCount(1))));
.apply(Sum.integersPerKey()); 
```

​	如果我们在流式引擎运行这个pipeline，结果将如图2-6所示

​	你可以看到我们怎么为每个窗口获取多个输出（panes）：每个相应的输入一次。当输出流被写到某类表时这类触发器很有效，你可以简单地轮询结果。

​	每条记录都触发的一个缺点是性能很差。当处理大规模数据时，求和等聚合提供了很好的机会在不丢失信息的前提下减少流的基数。在你拥有大数据量的key时尤其明显。

​	触发中处理时间延迟有两种不同的方法：

* 对齐延迟

  * 延迟将processing time切分成每个key和窗口都对齐的固定区域

* 非对齐延迟

  * 延迟与窗口内观察到的数据相关

  对齐延迟的pipeline如Example 2-4，结果如图2-7

  ```java
  Example 2-4. Triggering on aligned two-minute processing-time boundaries

  PCollection<KV<Team, Integer>> totals = input
  .apply(Window.into(FixedWindows.of(TWO_MINUTES))
  .triggering(Repeatedly(AlignedDelay(TWO_MINUTES)))
  .apply(Sum.integersPerKey());
  ```

​	你能够从如Spark Streaming这类微批流失系统总获取这类对齐延迟触发器。它最大的好处就是可预测性，您可以同时在所有修改过的窗口中定期更新。当然也有缺点：所有的更新一次性发生，会导致需要更高的峰值配置才能处理这种突发性的工作服在。另一种方法是使用非对齐延迟，如Example 2-5。

```java
Example 2-5. Triggering on unaligned two-minute processing-time
boundaries

PCollection<KV<Team, Integer>> totals = input
.apply(Window.into(FixedWindows.of(TWO_MINUTES))
.triggering(Repeatedly(UnalignedDelay(TWO_MINUTES))
.apply(Sum.integersPerKey());
```

​	对比图2-8种的非对齐延迟和2-6种的对齐延迟，很容易看出来非对齐延迟如何在一段时间内更均匀地分散负载。任何给定窗口所涉及的实际延迟在两者之间有所不同，有时更多，有时更少，但最终平均延迟将保持基本相同。从这个角度来看，非对齐延迟通常是大规模处理的更好选择，因为它们会随着时间推移产生更均匀的负载分布。

​	重复更新触发器非常适用于我们只希望随着时间的推移定期更新结果的用例，并且这些更新收敛于正确性并且没有明确指示何时实现正确性。但，正如我们第一章所说，分布式系统的变幻莫测经常导致事件发生的时间与管道实际观察到的时间之间存在不同程度的偏差，这意味着很难推断出输出何时提供输入数据的准确完整视图。对于输入完整性很重要的情况，重要的是要有一些推理完整性的方法，而不是盲目地信任计算结果，这些结果是由恰好已经找到通向管道的数据子集计算出来的。接下来研究watermarks

##### When: Watermarks

​	Watermarks能够回答问题"When in processing time are results materialized? " watermarks是event-time域中输入完整性的时间概念。换句话说，它们是系统衡量事件流中正在被处理的记录的事件时间的进度和完整性的方式。（有界无界都可以，虽然它们在无界的例子中的作用更明显）

​	回顾下第一章中的这张图，图2-9稍作了修改，我把event time和processing time之间的偏差描述成大多数真实世界的分布式数据处理系统中不断变换的时间函数。

![2-9](..\picture\streaming\2-9.png)

​	我说的代表真实世界的红线本质上就是watermark；它随着processing time的推移捕获event-time完整性的进度。从概念上讲，你可以把watermark想象成一个函数，F(P)->E，它传入processing time的一个点，返回event time的一个点。event time中的点E，是系统认为观察到了事件时间小于E的所有输入的点。换句话说，它是一个断言：不会再看到事件时间小于E的数据。由于watermark可以是perfect（完美）或者heuristic（启发式），所以该断言可以是严格的保证或者有根据的猜测。

- Perfect watermarks
  - 对于我们对所有输入数据完全了解的这种情况，是可以构建出perfect watermark的。这种情况没有late date这一说，所有数据都按时到达。
- Heuristic watermarks
  - 对于大多分布式输入源，完全了解输入数据是不现实的，对于这种情况最好的选择就是提供一个heuristic watermark。heuristic watermark使用输入的任何可用信息（分区，分区内有序，文件的增长率等）来提供尽可能准确的进度评估。在许多案例中，这样的watermarks可以在他们的预测中非常准确。即便如此，使用启发式watermark就意味着它可能是错的，这会导致late data。我们稍后介绍处理late data的方法。

由于它们提供了输入完整性的概念，所以watermarks形成了前面提到的第二类触发器：*完整性 触发器*的基础。watermarks本身是非常有意思且复杂的主题，第3章中将会深入探讨。但是现在，让我们通过更新我们的示例管道以利用基于watermarks的完整性触发器来研究下它们，如Example 2-6所示

![Example2-6](..\picture\streaming\Example2-6.png)

​	Watermarks有一个有趣的性质是：它们是一类函数。也就是说有多个不同的满足watermark特性的函数F(P) -> E表示不同程度的成功。我之前提到的，你完全了解输入数据的场景很可能建立一个完美的watermark，这是理想情况。而对于你无法完全了解输入数据或者计算完美watermark成本太高的场景，你可能会选择启发式来定义你的watermark。我想在这里提出的观点是使用中的watermark算法是独立于pipeline本身的。我们不打算在这里详细讨论实现watermark的意义（第三章说了）。现在，为了推动给定的输入集可以应用不同的watermarks这个想法，让我们看一下示例2-6中的pipeline，当在同一数据集上使用两个不同的watermark实现时（图2-10） ）：左边是perfect watermark; 右边是heuristic watermark。

​	在这两种情况下，当watermark通过窗口的末端时，窗口被计算。如你所料，perfect watermark随着时间的推移可以完美地捕获管道的event-time完整性。相比之下，用于右侧启发式水印的特定算法未考虑9的值，这大大改变了物化输出的形状，无论是在输出延迟还是正确性方面（如错误答案所示5为[12:00,12：02]窗口提供的）。



#### Summary

随着本章结束，你现在理解了强大的流式处理的基础只是，并准备好进入世界并做出惊人的事情。当然，还有八个章节焦急地等着你的注意力，所以希望你现在不会像现在这样，这一刻。 但无论如何，让我们回顾一下我们刚才所涵盖的内容，以免你在匆忙中忘记任何一个问题。 首先，我们谈到的主要概念：

* 事件时间与处理时间
  * 时间何时发生和事件何时被你的数据处理系统观测到之间最重要的区别
* 窗口化
  * 通过沿着时间边界（processing time或者event time，即使我们把Beam Model中的窗口化定义缩小到仅仅为沿着event time切分）切分的方式来管理无界数据的常见方法
* 触发器
  * 用于精确指定何时物化输出对你的特定用例有意义的声明机制
* Watermarks
  * event time中进度的强大概念，提供一种在处理无界数据的无序处理系统推理完整性的方法
* Accumulation

第二，我们用来构建探索的4个问题：

* what：什么结果会被计算？=transformations
* where：在事件时间的什么位置计算结果？=windowing
* When：处理时间中何时物化结果？=triggers + watermarks
* How：结果的细化如何做相关？=accumulation

第三，实现流式处理的这种模型提供的灵活性（因为最终，这就是真正的意义所在：平衡竞争紧张，如正确性，延迟和成本）。回顾一下我们能够在相同数据集上实现的主要输出变化，只需要进行最少量的代码更改：

![z1](..\picture\streaming\z1.png)

![z2](..\picture\streaming\z2.png)

我们只看了一种窗口化类型：事件时间中的固定窗口。如我们所知，窗口化有许多维度，我想在此时接触其中至少2个。然而，首先我们将绕道去深入了解watermarks，因为这个知识将帮助我们构建后面的讨论（本身也很有吸引力）。

### Chapter 3. Watermarks

到目前为止，我们一直在从管道作者或者数据科学家的角度来看待流式处理。Chapter 2介绍watermarks作为处理发生在event-time的何处和在processing tiem的何时物化结果（where in event-time processing is taking place and when in processing time results are materialized）问题的一部分答案。本章，我们回答同样的问题，但是从流式处理系统的底层机制的角度来看。研究这些机制将有助于我们激发，理解和应用watermakers的概念。我们将讨论如何在数据的入口处创建watermarks，它们如何在数据处理管道中传播，它们如何影响输出时间戳。我们也演示了在处理无界数据时，watermarks如何保留回答问题——事件时间中何处处理数据和数据何时被物化所必需的保证。

#### Definition

考虑任意持续获取数据和输出结果的管道。我们希望解决这样常见的问题：何时可以安全地关闭事件时间窗口，这意味着这个窗口不再需要更多数据。为此，我们将描述管道处理无界输入数据的进展。

解决这个事件时间窗口问题的一个简单方法是简单地将我们的event-time窗口基于当前的processing time。如我们在第一章所见，我们很快陷入麻烦——数据处理和传输不是即时的，所以处理时间和事件时间几乎不会相等。我们管道中的任何尖峰都有可能让我们错误地把信息分配给窗口。最终，这种策略失败的原因是，我们没有强有力的方式来保证这种窗口。

另一种直观但是不正确的方法是考虑管道处理消息的速率。虽然这是个有趣的指标，但是速率可能会随着输入的变化、预期结果和可用于处理的资源等的多样性而随机变化。更重要的是，速率无益于回答完整性这样的基本问题。具体来说就是，速率并没有告诉我们何时看到了某个特定时间间隔的所有消息。在现实世界的系统中，还会存在消息通过系统后并没有被处理的情况。这可能是是因为瞬态错误（例如崩溃，网络故障，机器停机）或持久性错误（例如需要更改应用程序逻辑或其他手动干预以解决的应用程序级故障）。当然，如果发生大量故障，处理速率指标可能是检测此问题的好方法。然而速率指标永远也无法告诉我们单个消息通过我们的管道却未能被处理。甚至单个这样的消息会输出结果的正确性产生任意影响。

我们需要更好地衡量进度。为了达到这个目的，我们对流数据做了一个基本的假设：*每条消息有一个相关的逻辑事件时间戳*。这个假设在持续到达无界数据的上下文中是合理的，因为这意味着会一直产生输入数据。在大多数情况下，我们可以把原始的事件发生时间当做它的逻辑事件时间戳。所有的输入消息都包含事件时间戳的话，我们就可以检查任意管道中的此类时间戳的分布。这样的管道可以被分发到很多agent上并发处理，并且在各分片间消费输入消息时并不保证顺序。因此，处于此管道中的诸多in-flight的消息的事件时间戳将形成如图3-1所示的分布。

消息由管道获取、处理，最终被标记为completed。每条消息要么是"in-flight"，表示它被接收到但还不是completed；要么是"completed"，表示这条消息不再需要被处理了。如果我们根据事件时间来查看消息的分布，将会像图3-1所示。随着时间推移，更多的消息将被添加到右侧的"in-flight"分布中，更多来自"in-flight"分布部分的消息将变成completed，并移到"completed"分布中。

![3-1](..\picture\streaming\3-1.png)

[3-1动图]: http://www.streamingbook.net/fig/3-1

在这个分布上有个关键的点，位于"in-flight"分布的最左侧边缘，对应我们管道中未被处理的消息的最老的时间戳（距离现在最老的时间戳其实就是最小的时间戳）。我们用这个值去定义watermark：

* *The watermark is a monotonically increasing timestamp of the oldest work not yet completed*
* watermark是一个单调递增的时间戳，这个时间戳是尚未完成的最老的负载的时间戳（时间戳最小值）

这个定义包含两个基础特性：

* Completeness
  * 如果watermark迁移越过了某个时间戳T，我们可以根据它的单调性保证：不会再对T之前的及时（非延迟数据）事件进行处理。因此，我们可以在T处或之前准确地发送任何聚合结果。换句话说，watermark允许我们知道何时能正确地关闭窗口
* Visibility 
  * 如果某条消息由于某些原因卡在管道中，watermark则不能前进。另外，我们还可以通过检查阻止watermark前进的消息来找到问题的根源。

#### Source Watermark Creation

这些watermarks从哪来？为某个数据源确定一个watermark，我们必须在来自这个数据源的每条消息进入管道的时候时给它们分配一个逻辑事件时间戳。正如第2章所述，所有watermark的创建都属于2大类：perfect和heuristic。为了提醒我们自己pefect和heuristic watermarks的区别，让我们看下图3-2，它展示了第2章中的窗口求和示例

![3-2](..\picture\streaming\3-2.png)

注意，区分特征是pefect watermarks确保watermark对所有数据进行计算，而heuristic watermarks只允许一些迟到数据。

在watermark被创建为pefect或heuristic后，watermarks在管道的其余部分保持不变。至于创建的watermark是完美的还是启发式的，很大程度上取决于被消费的数据源的性质。 为了了解原因，让我们看一下每种watermark创建的几个例子。

#####  Perfect Watermark Creation

完美式watermark的创建使用这种方式给到来的消息分配时间戳：最终的watermark严格保证管道将不会看到来自该数据源的、事件时间小于watermark的数据。使用完美式watermark创建的管道不再处理迟到的数据。然而，完美式watermark的创建需要完全了解输入数据，这对很多现实世界的分布式输入源来说是不切实际的。下面是两个能创建完美式watermark的例子：

* Ingress timestamping（进入时间戳）
  * 那些将数据进入系统的事件时间设置为Ingress timestamping的源，可以创建完美式watermark。在这种情况下，源watermark只是跟踪管道所观察到的当前处理时间。几乎所有支持窗口化的流式系统在2016年之前都使用这个方法。
  * 因为事件时间是从单个单调增加的源分配的（实际上是处理时间），所以系统完全知道该数据流接下来到达的是哪些时间戳。最终，event-time 进度和窗口化语义会十分容易推理。当然，缺点是watermark与数据本身的事件时间没有关系；数据本身的事件时间被丢弃了，而watermark只是跟踪关于数据到达系统的进度。
* Static sets of time-ordered logs 
  * 静态大小的时序日志输入源（如一个静态分区集的Kafka topic，输入源的每个分区包含单调递增的事件时间）将是创建完美式watermark相对简单的数据源。为此，数据源将简单追踪整个已知和静态源分区集上未处理数据的最小事件时间（即，每个分区中最近读取记录的事件时间的最小值）
  * 与Ingress timestamping类似，系统完全知道接下来哪些时间戳将到达，这要归功于已知整个静态分区集的事件时间是单调递增的。这实际上是一种有界乱序处理的形式；已知的分区组中乱序的量受这些分区中观察到的最小事件时间的限制。
  * 通常，你能保证分区中的时间戳单调递增的唯一方法是那些分区中的时间戳是否在数据写入分区时被分配的，例如，web前端直接记录事件到Kafka。即便这仍然是个有限的用例，但它无疑比基于数据到达处理系统的进入时间戳更有用，因为watermark追踪了有意义的基础数据事件时间

#####  Heuristic Watermark Creation

启发式watermark创建的watermark仅仅是不会看到事件时间小于watermark的数据的一个*估计*。使用启发式watermark创建的管道可能需要处理一定数量的*late data*（延迟数据）。延迟数据是在watermark已经推移越过了这个数据的事件时间之后到达的数据（watermark认为时间戳小于它的数据都已经被处理，那这里是仍然会处理？）。延迟数据只可能伴随启发式watermark创建。如果启发式算法相当不错，延迟数据的量可能会很小，并且这个watermark可以用作完成估计。如果是需要支持要求准确性的用例（如交易），系统还需要为用户提供处理延迟数据的方法。

对于很多真实世界的分布式输入源，构建完美式watermark在计算上和操作上都是不切实际的，但是还是有可能利用输入数据源的结构化特征来建立一个高度精确的启发式watermark。接下来是两个启发式watermarks的例子：

* Dynamic sets of time-ordered logs
  * 动态的结构化日志文件集合（每个文件中包含的记录是的事件时间是单调递增的，但是文件之间的事件时间没有固定关系），在运行时是不知道这些日志文件（即kafka分区）的全集的。这样的输入通常见于由多个独立的组构建和管理的全球规模的服务。在这个用例中，虽然在输入上创建完美式watermark是很难的，但是创建准确的启发式watermark可能性却很大。
  * 通过追踪已有的日志文件集合中未处理数据的最小事件时间，监控增长率和利用像网络拓扑和带宽可用性这样的外部信息，你可以创建一个非常准确的watermark，即使缺少对所有输入的完全了解。这种输入源是Google中最常见一类无界数据集，所以我们对这些场景创建和分析watermark质量拥有丰富的经验，并且已经看到它们在大量用例中使用中具有良好效果
* Google Cloud Pub/Sub（发布订阅）
  * Cloud Pub/Sub目前不确保有序传递；即使某个publisher有序发布两条消息，仍然有可能乱序传递（这是因为底层框架的动态性质，它允许在没有用户干预的情况下扩展到非常高的吞吐量水平）。最终，无法为Cloud Pub/Sub保证完美式watermark。然而，Cloud Dataflow组已经通过利用Cloud Pub / Sub中有关数据的可用知识建立了一个非常准确的启发式watermark。本章稍后将详细讨论这个启发式的实现。

考虑一个用户玩手机游戏的例子，它们的分数会被发送到我们的管道处理：通常你可以假设使用移动设备进行输入的源一般不可能提供完美式watermark。由于设备长时间脱机的问题，没有办法对这种数据源提供任何合理的绝对完整性估计。但是，您可以设想构建一个watermark，准确跟踪当前在线设备的输入完整性，类似于刚才描述的Google Pub / Sub watermark。从提供低延迟结果的角度来看，活跃在线的用户很可能是最相关的用户子集，所以它通常不会像你最初想象的那样，有那么多缺点。

从广义上讲，对于启发式watermark creation，你越了解源，启发式结果越好，能看到的延迟数据也越少。考虑到源的类型、时间的分布和使用模式将差异巨大，没有一个通用的方案。但在任何一种情况下（完美或启发式），在输入源处创建水印之后，系统可以完美地通过管道传播watermark。 这意味着完美式watermarks将在下游保持完美，启发式水印将严格保持与启动时一样的启发式。 这是watermark方法的好处：您可以将管道中完整性跟踪的复杂性完全降低到在源头创建watermark的问题。

#### Watermark Propagation

到目前为止，我们只考虑了在单个操作或阶段上下文中的输入的watermark。 但是，大多数现实世界的管道都包含多个阶段。 了解watermark是如何在独立阶段传播的对于了解它们如何影响整个管道以及观察到的结果的延迟非常重要。

```
PIPELINE STAGES
通常，每次你的管道根据一些新维度对数据进行分组需要多个阶段。比如，你有这样一个管道，消费原始数据、计算每个用户的聚合结果，然后这些聚合结果去计算每个团队的聚合结果，你最终可能会得到一个包含3个阶段的通道：
* 一个消费原始、未分组数据
* 一个按用户分组数据并计算每个用户的聚合结果
* 一个按团队分组数据并计算每个团队的聚合结果
我们将在第6章中详细了解分组对管道形状的影响。
```

如上节所述，watermarks被创建于输入源处。之后，随着数据通过系统，watermarks从概念上流经系统。您可以以不同的粒度级别跟踪水印。对于由多个阶段构成的管道，每个管道可以追踪它自己的watermark，watermark的值是其前面的所有输入和阶段的函数。因此，在管道中稍后出现的阶段将具有过去进一步的watermarks（因为它们已经看到较少的总体输入）。

我们可以在管道中的任何单个操作或阶段的边界定义watermark。这不仅有助于了解管道中每个阶段所取得的相对进展，而且有助于为每个阶段及时分发结果。我们对阶段边界的watermark给出以下定义：

* *输入watermark*。捕获该阶段上游所有内容的进度（即，该阶段输入的完成程度）。对于源阶段，输入watermark是为输入数据创建watermark的指定源函数。对于非源阶段，输入watermark被定义为所有上游源和阶段的所有分片/分区/实例的输出水印的最小值。
* *输出watermark*。捕获阶段本身的进度，基本上被定义为该阶段输入watermark和该阶段所有非延迟数据active消息的事件时间的最小值。“活动”包含的内容在某种程度上取决于给定阶段实际执行的操作以及流处理系统的实现。它通常包括缓冲用于聚合但尚未实现下游的数据，等待传输到下游阶段的输出数据，等等。

给指定的阶段定义一个输入和输出watermark的一个好处是：我们可以使用它们来计算某个阶段引入的事件时间延迟的量。输入水印的值与输出水印的值之差为该阶段引入的事件时间延迟或滞后的量。这个滞后是每个阶段的输出将在实时之后延迟多久的概念。举个例子，某个阶段执行10s窗口聚合将有10s或更多的滞后，意味着该阶段的输出至少在输入和实时之后延迟那么多。输入和输出水印的定义提供了整个管道中水印的递归关系。管道中每个后续阶段根据该阶段的事件时间滞后在必要时延迟水印。
每个阶段的处理也不是单一的。 我们可以将一个阶段内的处理分成具有多个概念组件的流程，让每个概念组件都有助于输出watermark。 如前所述，这些组件的确切性质取决于阶段执行的操作和系统的实现。 从概念上讲，每个这样的组件都充当缓冲区，其中活动消息可以保留到某些操作完成。 例如，当数据到达时，它被缓存处理。 然后，处理可以将数据写入状态以供稍后延迟聚合。 触发时，延迟聚合可能会将结果写入输出缓冲区，等待下游阶段的消费，如图3-3所示：

![3-3](..\picture\streaming\3-3.png)

我们可以使用每个这样的缓冲区拥有的watermark来追踪它自己。每个阶段所有缓冲区中最小的watermark构成这个阶段的输出watermark。这样输出watermark可能是下面所列的最小值：

* *Per-source* watermark——为每个发送阶段
* Per-external input watermark—for sources external to the pipeline
* Per-state component watermark—for each type of state that can be written
* Per-output buffer watermark—for each receiving stage 

##### Understanding Watermark Propagation 

为了对输入和输出watermark之间的关系和它们如何影响watermark propagation有更直观的感受，让我们看个例子。让我们看看游戏计分，不过我们将尝试衡量用户参与度而不是统计队伍总分。我们先假设用户玩游戏的时长等价于他们享受它的程度，之后，先计算每个用户的时长。在回答了计算会话长度（游戏时长）的4个问题后，我们会以计算固定时间段内的平均会话长度再次回答它们。

为了使我们的示例更有趣，我们假设我们正在使用两个数据集，一个用于移动分数，另一个用于控制台分数。我们希望通过这两个独立数据集上的整数求和来执行相同的得分计算。一个管道正在计算在移动设备上玩的用户的分数，而另一个管道用于在家庭游戏控制台上玩的用户，可能是由于针对不同平台采用的不同数据收集策略。重要的一点是，这两个阶段在不同的数据上执行相同的操作，因此具有非常不同的输出水印。

我们从Example 3-1开始，看看这个管道的第一部分代码是什么样的.

![Example3-1](..\picture\streaming\Example3-1.png)

这里，我们独立地读每个输入，并且按用户分组。之后，每个pipeline的第一个阶段，我们进入会话窗口，并调用了一个叫CalculateWindowLength的自定义Ptransform。这个Ptransform按用户分组，然后通过将当前窗口的大小视为该窗口的值来计算每个用户会话长度。在这种情况下，我们可以使用默认触发器（AtWatermark）和累积模式（discardingFiredPanes）设置，但我已明确列出它们的完整性。 两个特定用户的每个管道的输出可能如图3-4所示。

由于我们需要跨多个阶段追踪数据，为了区分，我们用红色代表Mobile Scores相关的东西，蓝色代表Console Socres相关的东西，而黄色代表Average Session Lengths的watermark和输出，见图3-5。

我们已经回答了那4个问题 —— what、where、when、how计算单个会话长度。接下来我们将再次回答它们，将这些会话长度转换为固定时间窗口内的全局会话长度平均值。这要求我们首先将我们的两个数据源拼合为一个，然后重新进入固定窗口;我们已经在我们计算的会话长度值中捕获了会话的重要精髓，现在我们想要在一天中的一致时间窗口内计算这些会话的全局平均值。 例3-2显示了这个代码。

![Example3-2](..\picture\streaming\Example3-2.png)

如果我们看到这个管道正在运行，它将如图3-5所示。如前所述，两个输入管道分别为移动和客户端玩家计算单个会话长度。然后那些会话长度将进入管道的第二阶段，在固定窗口中计算全局会话长度。

这里有两点比较重要：

* Mobile Sessions和Console Sessions各阶段的***输出watermark***至少与每个的相应输入watermark一样久，实际上会更久点。这是因为在实际系统中，计算结果需要花费时间，并且在给定输入的处理完成前。我们不允许输出watermark前移。
* Average Session Lengths阶段的***输入watermark***是直接上游两个阶段的输出watermark中的较小值

结果是下游输入watermark是上游输出watermark的最小组成的别名。注意这和本章前面的那两类watermarks的定义是相匹配的。还要注意过去下游的水印是如何进一步的，捕捉直观的概念，即上游阶段将比其后续阶段更进一步提前。

### Chapter 4. Advanced Windowing

现在我们已经深入了解了水印，我想深入研究一些与what/where/when/how问题相关的更高级的主题。

首先我们看看*processing-time windowing*，它是where和when的有趣组合。我们了解下它和事件时间窗口化的关系，以及什么时候使用它才是正确的。之后，我们深入一些更高级的事件时间窗口化的概念，详细研究*session windowing*，并且最后通过探索三种不同类型的自定义窗口来说明为什么常见的*custom windowing*是有用的概念。

#### When/Where：Processing-Time Windows

处理时间窗口化很重要，因为：

* 对于某些用例，比如使用率监控（如，web服务流量QPS），你想在观察到到来的数据流时就处理它，这个时候使用处理时间窗口是绝对正确的。
* 对于那些事件发生时间很重要的用例（如，分析用户行为趋势、计费、算分等），使用处理时间窗口化是绝对错误的，而且能够识别这些用例十分重要

因此，值得深入了解处理时间窗口和事件时间窗口之间的差异，特别是考虑到当今许多流式系统中处理时间窗口的普遍存在。

当一个模型是严格基于事件时间的，如本书介绍的模型，有两种方法你可以用来获取处理时间窗口：

* Triggers
  * 忽略事件时间，使用触发器去提供窗口在处理时间轴上的快照
* Ingress time
  * 把ingress time设置为数据到达时的时间时间，并从那里开始使用正常的事件时间窗口。 这基本上就像Spark Streaming 1.x那样

请注意，上述两个方法差不多是等效的，但它们在多级管道下略有不同：在triggers方法中，多级管道将在每一阶段独立地切分处理时间窗口，所以对于一个阶段，窗口N中的数据可能反而在下一阶段的窗口N-1或N + 1中结束；而在ingress time中，在将数据合并到窗口N之后，由于阶段之间通过水印（在云数据流情况下），微量分配器边界（在 Spark Streaming case），或引擎级别涉及的任何其他协调因素。

处理时间窗口的一大缺点是当输入的观察顺序发生变化时，窗口的内容会发生变化。为了更具体地推动这一点，我们将看看这三个用例：事件时间窗口，通过触发器处理时间窗口，以及通过入口时间处理时间窗口。

##### Event-Time Windowing

为了确定一个基线，我们先



#### 总结

高级窗口是一个复杂多变的主题，本章，我们主要介绍3个高级概念

* 处理时间窗口
* 会话窗口
* 自定义窗口





图和表见：http://www.streamingbook.net/fig

