# README

Lab1一共分为三个部分，分别是：

- [Part 1: PC Bootstrap](lab1_part1.md)
- [Part 2: The Boot Loader](lab1_part2.md)
- [Part 3: The Kernel](lab1_part3.md)

Lab 1因为涉及到PC boot的过程，不可避免的会遇到x86汇编相关的知识，如果你对这些知识感到陌生，也不用担心。Lab涉及到的汇编不多，而且都是最基础的部分。

Part 1 主要是描述80386的物理地址空间、相关的BIOS知识，以及如何用[QEMU](http://www.qemu.org/)模拟X86。

Part 2 主要ELF格式以及bootloader，将kernel读入到内存对应的位置。Part 1与Part 2都不需要写代码（理论上可以跳过这两个部分，但是最好不要）。

Part 3 主要是了解printf的实现以及gcc的calling convention。这一部分需要写很少量的代码。
