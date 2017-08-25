# Sharing data between threads

## Problem with sharing data between threads

当多个线程的共享数据是只读*readonly*的情况，是不会产生额外要分析的问题的。但当多个线程其中包含又对共享数据写权限的线程时候，就会产生额外的问题。

在编程过程中，往往有许多不变式*invariants*帮助我们理解代码的含义，但是这种不变式往往会在数值更新时被破坏，越是复杂的数据类型，越是在更新时进行多的操作的类型，这个破坏不变式的过程越容易在多线程编程中引发问题。

比如一个双向链表*doubly linked list*包含不定式：

- 节点A的后向指针指向节点B，则有节点B的前向指针指向节点A。

在某些链表操作就会短时间内破坏这个不定式，如删除节点N：

1. 找到节点N
2. 将节点N的前节点的后向指针指向节点N的后节点
3. 将节点N的后节点的前向指针指向节点N的前节点
4. 删除节点N

在步骤2-3之间，双向链表的一个重要的不定式被破坏。若在其他线程的逻辑中有用到该不定式的代码，该代码将会产生未定义后果。

这是在并发编程中常见的BUG原因：*race condition*，也称*data race*。

### race condition and avoiding problematic race conditions

*race condition*通常指代那些会导致BUG的竞争条件，其带来的BUG通常是未定义行为。

*race condition*主要是各线程对共享数据的修改导致的。其带来的BUG对时间十分敏感，在DEBUG的模式下很容易被忽略。当修改操作被连续的计算机指令执行时，*race condition*的概率较低；随着计算机负荷的上升，有问题的竞争操作的发生概率也随之提高。

解决*race condition*的主要办法有以下几种：

1. 使用保护机制封装共享数据，保证只有一个线程能改变数据并且其他线程只能在该操作尚未发生或者已经结束的情况下访问数据，即*mutex*。
2. 改造共享数据结构的设计，使得该结构进行的修改操作不会破坏数据结构的不变性，即*lock-free programing*。
3. *software transactional memory*，STM

## Protecting shared data with mutexed




