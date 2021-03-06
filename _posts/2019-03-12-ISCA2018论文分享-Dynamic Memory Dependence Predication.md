---
layout: post
title:  "ISCA2018论文分享-Dynamic Memory Dependence Predication"
categories: 芯片
---

* content
{:toc}





# Dynamic Memory Dependence Predication
>  - **Conferences:** ISCA, 2018
>  - **Authors:** Zhaoxiang Jin ; Soner Önder
>  - **Site:** https://ieeexplore.ieee.org/document/8416831
>  - **Keywords:** Memory Dependence Prediction, Store Queue Free Architecture, Memory Cloaking, Dependence Predication

## 简介
NoSQ 是 Store-queue-free 架构的一种方案，主要的思想是去除 store queue， 转而使用预测器预测 store-load 的依赖关系并采用 memory forward + re-execution 来实现 store-load 的交互。其平均性能较传统 store queue 架构提升了 2%. 

本篇论文提出 DMDP 架构（Dynamic Memory Dependence Predication） 对 NoSQ 架构进行了改进，通过在寄存器重命名阶段中插入指令去改变 load 的行为，减轻了内存依赖关系的错误预测，降低了 re-execution 的次数。实验表明，相比于 NoSQ 架构，DMDP 架构带来了整数 7.17%、浮点 4.48% 的性能提升以及 6.7% 的能耗节省。

DMDP 架构的基本思想是插入一个新的微指令，去比较被预测的load-store依赖对的地址。如果地址一致，则load直接使用store的寄存器；否则，load从cache中读数据。这样，执行的load和提交的store之间的错误依赖就被消除了。消除这种依赖是重要的，因为当考虑内存一致性模型时，store指令commit的延迟会大幅度的增加。

store-queue-free架构消除了store-queue，但store buffer仍然需要保存retired store直到store更新了缓存。这一组件也是合理实现内存一致性模型和处理大量store miss的必要组件。

## 研究背景
**1. 传统 store queue 架构**

超标量处理器中，store指令在commit阶段才更新内存子系统。因此，必须要一种机制处理in-flight store与load之间的交互。没有这种机制，in-flight load以及依赖它的指令就要等之前的所有store指令提交才能确定读到的值。目前，大部分的处理器采用联合搜索和年龄排序的store queue来处理这种交互。当load指令执行时，使用联合搜索store queue获取值的结果，同时对缓存进行访问。如果store queue中存在相同地址的值，则将最年轻（最新）数据作为load的值，否则使用缓存中的值。++但这种机制的缺点是，搜索需要时钟，还会增加in-flight store的数量。而为了更多的指令并行，每个核都需要一个很大的store queue，增加了微架构的复杂程度和空间。++

**2. NoSQ 架构**

NoSQ是一种能够完全消除store queu的一种机制。在NoSQ架构中，store在commit阶段执行并更新缓存。因此，store指令不需要发射到乱序引擎中。而inflight store-load的交互通过内存依赖预测器来完成。如果load A被预测依赖于store B，那通过重命名load的目标寄存器为store B的目标寄存器就能完成读写操作（这种技术叫memory cloaking）。这一过程将DEF-store-load-USE依赖链压缩成DEF-USE。为了防止预测错误，load在retire阶段必要时要进行re-exectue。++其优点是去除了store queue，但缺点是一旦预测错误，load必须要等store提交并且更新了cache才能进行re-execute。降低re-execution的延迟和提高预测准确性就成了NoSQ的关键。++

## 动机

**load-store 依赖关系可以分为三类：**

1. ++Never Colliding（NC）++
    
    load指令一直从cache中读数据。例如，大量对不变数组的访问。

2. ++Always Colliding（AC）++

    load指令一直从store queue中取前传数据。store-queue-free架构很适合处理这种情况，因为这种情况会有很高的预测准确性。例如，寄存器溢出，全局变量的访问，栈访问等等。

3. ++Occasionally Colliding（OC）++

    load指令可能从cache也可能从store中读数据。OC这种情况很难预测，因为仅通过内存依赖的历史信息难以做出正确的预测。例如图1所示：
    
    ![image](https://note.youdao.com/yws/api/group/89373453/file/435828243?method=getImage&WLP=true&width=640&height=640&version=1&cstk=i04bGl3-)
    
    图中的箭头，表示两个指令之间是冲突的(Colliding)。
    
    OC情形是处理load-store依赖问题的关键。一个普通的store-queue-free机制，比如NoSQ，首先从cache中读数据。当第一次colliding发生时，依赖关系加入，并且之后的递增指令将会预测从之前迭代的store中获得数据，而不再是cache。但是，当指针地址改变时，前传的数据大概率是错的，即内存依赖预测是错的。经常的错误预测会导致load按照严格的程序序执行，这个load只能等待别名的store全都提交完才能被执行。图1（c）就表示递增指令只能等前一个指令执行完才能执行。这种严格的序关系能够确保load指令不管是否store-load地址匹配也能读到正确的值，但却严重地影响了程序的性能。
    
    **那么首先确定的是，如果load-store地址不一致，load和预测的store造成的延迟就是不必要的。** 而即使地址是一致的，load也不得不等到store commit才能执行，这也是不必要的。而大量不相关事件，比如cache miss还可能会延迟store的提交，进而影响load的执行时间。

    ![image](https://note.youdao.com/yws/api/group/89373453/file/435828436?method=getImage&WLP=true&width=640&height=640&version=1&cstk=i04bGl3-)
    
    图2表明了三种load在NoSQ中的分布情况。Direct access表示从cache中读；ByPassing表示通过memory cloaking前传数据；Delayed access表示load不能直接从cache中读直到冲突的store提交。
    
    可以看到有些程序中，有超过10%的Delayed access。文章同时比较Delayed access和Bypassing的平均执行时间，如图3：
    
    ![image](https://note.youdao.com/yws/api/group/89373453/file/435828498?method=getImage&WLP=true&width=640&height=640&version=1&cstk=i04bGl3-)

    我们会发现Delayed access在绝大部分的benchmark中都占有更多的时间，而且大概是Bypassing access的7倍。

    所以，DMDP的重点就在于如何减少这部分（OC）的执行时间。
    
## 基本原理

NoSQ架构使用预测器预测load-store依赖关系，但当load-store不一致时，load需要等带store提交。这类似于分支预测，而如何预测进入哪个分支是困难的。因此，DMDP动态地插入预测确认，即比较load-store的地址。这一比较的结果可以用来指导load是从cache中或者未提交store中获取正确数据，就类似于条件转移语句。

![image](https://note.youdao.com/yws/api/group/89373453/file/435828659?method=getImage&WLP=true&width=640&height=640&version=1&cstk=i04bGl3-)

图4.表示了DMDP中load如何通过三种方式获取数据。可以看到第三种方式，由于预测的置信度低，load的地址需要和store地址进行比较，然后确定load从cache还是P7中获取数据。

表格I详细地说明了NoSQ与DMDP之间的相同点与不同点。

![image](https://note.youdao.com/yws/api/group/89373453/file/435828691?method=getImage&WLP=true&width=640&height=640&version=1&cstk=i04bGl3-)

store的数据和地址寄存器在原有程序语义中是不存在的，而这些寄存器可能被释放或重分配给其他指令。**在DMDP中，需要延迟store寄存器的释放时间，直到store被提交或者更新了cache。为了这个目的，DMDP对于每个内存操作都设置了一个额外寄存器去保存计算的物理地址（不一定要保存所有）**。而这一改变还能简化地址的比较操作，同时可以直接获取物理地址而不用重新计算。

![image](https://note.youdao.com/yws/api/group/89373453/file/435828865?method=getImage&WLP=true&width=640&height=640&version=1&cstk=i04bGl3-)

图5表示，低置信度下load-store实际内存依赖的分布。IndepStore表示，load被预测依赖于store而实际不依赖与任意未提交store；DiffStore表示，load被预测依赖于store而实际依赖于另一个store；Correct表示预测是正确的。我们会发现绝大部分的预测错误都发生在IndepStore。**而一般的想法是，每次出现低置信度的预测，就直接读取cache的值，错误之后再进行re-execution。这样的平均错误预测率为11.4%。而DMDP优化了这一部分，使得错误率在3.7%**。

## 微架构

DMDP的微架构如图6所示，每个store使用SSN（store sequence number）来追踪其状态，还有三个全局可见的物理寄存器，SSN_rename，SSN_retire，SSN_commit。

![image](https://note.youdao.com/yws/api/group/89373453/file/435841400?method=getImage&WLP=true&width=640&height=640&version=1&cstk=i04bGl3-)

当store被rename，SSN_rename自增，并作为其SSN，因此更年轻的store会有更大的SSN。当store被retire或者commit，SSN_retire = SSN 或 SSN_commit = SSN。**Store Buffer**像队列一样工作，但load从来不会对Store buffer进行搜索。**Store Register Buffer**保存每一个未提交store对应的物理寄存器号，包括store的数据寄存器号和物理地址寄存器号。

在DMDP中，store指令有额外的物理寄存器生命周期，因为即使在store指令retire后，这个寄存器还可能被读取。因此，DMDP还包括一个**Physical Register Reference Counter**去管理寄存器的释放。

当load被rename时，预测依赖于冲突store的SSN将会被保存起来。这个SSN_pyb = SSN_rename - dist, 其中dist通过**Store Distance Predictor**和**branch predictor**来预测计算。

当load被执行时，先读取cache中的数据，并保存当时的SSN_commit为SSN_nvul。SSN_nvul表示load为确定的窗口中最年轻的并且已经commit的store。

当load被retire，投机的load仅需要确认是否读到正确的值（对比store和load的地址）。必要情况下，load需要进行re-execution。为了最小化re-execution的次数，DMDP建立了一个对应地址上的SSN历史，叫**Tagged Store Sequence Bloom Filter（T-SSBF）**，其结构类似一个hash table。

这里列出load需要re-execution的条件：

![image](https://note.youdao.com/yws/api/group/89373453/file/435841578?method=getImage&WLP=true&width=640&height=640&version=1&cstk=i04bGl3-)

具体分析参考RAW等读写错误的情况。

## 实验及结果

论文采用PISA作为指令集进行了仿真实验，通过GCC来插入指令。最后得到了如下结果。

![image](https://note.youdao.com/yws/api/group/89373453/file/435841640?method=getImage&WLP=true&width=640&height=640&version=1&cstk=i04bGl3-)

如图所示，DMDP得到了比NoSQ更好的IPC，但相比于最好的预测结果还有微小的差距。相信还能通过改进预测器来获取更好的效果。
