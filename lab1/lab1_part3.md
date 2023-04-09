# Part 3: The Kernel

至此，我们终于进入了kernel

在正式进入kernel的代码之前，我们需要处理part2遗留的问题，即kernel被读入的物理内存地址为0x00100000，对应的虚拟地址为0xf0100000，是如何确定的。

这两个值在ELF text段对应的program header中给出(VMA=0xf0100000, LMA=0x00100000):

```
Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         0000178e  f0100000  00100000  00001000  2**2
```

这两个值是在`kern/kernel.ld`中被设置的:

```
	/* Link the kernel at this address: "." means the current address */
	. = 0xF0100000;
	
	/* AT(...) gives the load address of this section, which tells
	   the boot loader where to load the kernel in physical memory */
	.text : AT(0x100000) {
		*(.text .stub .text.* .gnu.linkonce.t.*)
	}
```

jos中的[物理内存布局](lab1_part1.md/#physical-memory-layout)我们已经知道了，虚拟内存布局在`inc/memlayout.h`中。总体而言，所有的物理内存被映射到了以`kernelbase(0xf0000000)`为起点的对应位置。

![kernel in virtual memory & physical memory](picture/kernel.png)

完成物理地址到虚拟地址的转换需要页表的参与，在kernel启动初期，我们需要手动的设置好对应的页表，然后enable 80386的paging功能，下面的汇编完成了这些事：

```
	# Load the physical address of entry_pgdir into cr3.  entry_pgdir
	# is defined in entrypgdir.c.
	movl	$(RELOC(entry_pgdir)), %eax
f0100015:	b8 00 70 11 00       	mov    $0x117000,%eax
	movl	%eax, %cr3
f010001a:	0f 22 d8             	mov    %eax,%cr3
	# Turn on paging.
	movl	%cr0, %eax
f010001d:	0f 20 c0             	mov    %cr0,%eax
	orl	$(CR0_PE|CR0_PG|CR0_WP), %eax
f0100020:	0d 01 00 01 80       	or     $0x80010001,%eax
	movl	%eax, %cr0
f0100025:	0f 22 c0             	mov    %eax,%cr0
```

这里的RELOC定义在`entry.S`中，即将我们在c语言中定义的符号从虚拟地址转换到物理地址。

```c
#define	RELOC(x) ((x) - KERNBASE)
```

`	movl $(RELOC(entry_pgdir)), %eax`，将entry_pgdir的物理地址写入`%eax`，然后写入`%cr3`。

接着我们来看`entry_pgdir`与`entry_pgtable`的具体实现。这里默认你具备(多级)页表的相关知识，即能够理解利用页表进行虚实地址转换的过程。

![paging](picture/paging.png)
