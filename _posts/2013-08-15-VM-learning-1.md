---
layout: post
title: VM学习之一
category: detail
tags: VM 学习笔记
---

之前断断续续地在工作和学习中接触了一些虚拟机(Virtual Machine)的概念
和实现层面的细节，但这种接触和认知总是不系统，不成体系化，要想建立起
大至全局(big picture)小至各种细节的知识体系，还是需要像在校园中学习一样
通过课本来学习理论，通过习题（编程）来了解工程和实践。所以，开始系统地学习
VM并建立起自己的知识体系。

首先，需要一个课本类的书籍入门，显然[Virtual Machine: Versatile Platforms for Systems and Processes][Virtual Machine: Versatile Platforms for Systems and Processes]是个
不错的选择，在学习中做出一些自己的学习理解也是巩固知识的重要手段，我的学习
笔记便会记录于此。

## 第一章 Introduction to VM

这是一个概述性的章节，主要是在大的方面来介绍概念，分类和大的架构。

VM, Virutal Machine这个概念出来不晚了（60年代），最早先是System VM，是为了解决
大型主机客户对于不同OS要求建立起来的，显然那时计算机还比较昂贵，VM可以很好是为
不同的客户提供所需的OS和相关环境，并且让用户感觉是独占，而其实是分时的，并且由VM
来分配资源。90年代时一些高级语言不断出现，Java, Python等，跨平台性的要求产生了
Process VM的概念和实现，最常见的就是JVM，CLI等实现。当然这里面基本都是stack based
的实现，作者并没有提到Lua的register based实现。

所以VM可以分为Process VM和System VM，前者通常是一种高级语言的runtime，用于解释执行
高级语言的bytecode，而后者通常是一个虚拟完整的计算机，例如VMWare的产品，VirtualBox等。
记得几年前，自己的理解还是很片面，基本上将VM＝System VM,而当别人提到JVM时很纳闷为什么
也叫虚拟机。

Process VM的基本架构可以简单如图示：

![PVM](/assets/images/PVM-1.png)

对于Guest(高级语言的bytecode)并不需要关注具体的操作系统，它只需要知道的是解释执行它的VM
即可。而在VM的具体实现上，则不同的OS，处理器有不同的ISA，因而中立的bytecode会在虚拟机
层面进行不同的map。

而System VM的基本架构如图：

![SVM](/assets/images/SVM-1.png)

对于运行于System VM中的Guest OS上的应用，它其实是不知道背后的Host OS的，它对于硬件的操作
也是通过Guest OS->Host OS->Hardware来传导完成的，但整个过程对于此应用是透明的。

说到VM的好处，有如下几点：

1. 跨平台性（portable）
2. 性能优化的可能（VM层面可以针对具体的机器进行指令的优化等）
3. 安全性（Guest与Host之间的独立，Guest之间的独立）

另外像JVM这样比较完整和成熟的虚拟机实现，上层的语言只要编译后的bytecode符合JVM规范，则可以
直接使用JVM的虚拟机，例如Jython等JVM之上的实现。

Chapter1中有一句关于人类如何搞定复杂事物的说明很是精彩，摘录如下:

<pre>
The key to manage complexity in computer systems is their divsion into <i>levels of abstraction</i>,
seperated by <i>well-defined interfaces</i>.
</pre>



## 参考资料
1. [Virtual Machine: Versatile Platforms for Systems and Processes][Virtual Machine: Versatile Platforms for Systems and Processes]


[Virtual Machine: Versatile Platforms for Systems and Processes]: http://book.douban.com/subject/1790419/

