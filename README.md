# README

这一系列文章主要是我对[MIT 6.828 Operating System Engineering](https://pdos.csail.mit.edu/6.828/2018/schedule.html)课程学习的笔记。

你可以在我的github(todo)中找到源码，我尽量把每个exercise作为一次提交的内容(todo)，方便你在学习过程中更明确每一个exercise的内容，同时更易于理解。

## Why 6.828

在正式进入具体内容之前，首先得讲一下为什么在2023年选择做18年的6.828，而不是更近的[6.S081/6.1810](https://pdos.csail.mit.edu/6.828/2022/).

MIT对于6.S081与6.828的区别的描述为：
> This fall we will offer a new operating system class, as an experimental subject: 6.S081. It is intended for undergraduates who enjoyed 6.004 and want to learn about design and implementation of operating systems, and their use as a foundation for systems programming. A multi-processor operating system for RISC-V is used to illustrate these topics. Individual laboratory assignments involve extending an operating system kernel, for example to support sophisticated virtual memory features and network protocols. Programming experience is a prerequisite, ideally in the C language. 6.S081 will fulfill the AUS requirement.
>
> 6.828 will be offered as a graduate-level seminar-style class focused on research in operating systems. 6.828 will assume you have taken 6.S081 or an equivalent class.

简单来讲，区别有两点：

1. 难度。6.S081更简单，难度是本科生入门的第一门操作系统课程。而原有的6.828是研究生级别的课程，改了之后的[6.828](https://abelay.github.io/6828seminar/schedule.html)更集中在OS research相关的内容，也即是读论文奖论文。
2. 汇编。6.828用的是x86，6.S081用的是RISC V.

我选择做6.828的出发点很简单，正是第二点，**现实世界中x86的应用范围更广**。

## JOS vs XV6

在课程的内容中经常会出现两个不同的操作系统，jos与xv6。

xv6是MIT的老师重写的以前的Unix V6，是一个monolithic kernel的、具有完整功能可以正常运行的操作系统。6.S081就是修改xv6，增加新的功能。

jos是需要我们逐步实现、完善的操作系统。jos是一个exokernel的操作系统。kernel只提供部分基本的功能，其余的由library来实现。这一点在copy-on-write fork中体现的比较清楚。（todo：描述xv6与jos的架构）

xv6的功能的具体实现可以供学生在实现jos时作参考。

## Structure

每一个lab大致都会分为2-3个部分，分别是：对相应概念、原理的讲解，exercise的完成以及代码分析，最后加上一些补充内容。
