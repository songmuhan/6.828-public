# Part 2: The Boot Loader

在Part1中BIOS的最后执行了`int 0x19`，其作用为寻找bootable的磁盘(等)，并将其第一个扇区(MBR)，内容为bootloader(`obj/boot/boot`)的512个字节读入到内存`0x7C00-0x7DFF`处，并执行对应的代码，在JOS中即为从`start`开始执行。

JOS中的bootloader由`boot/boot.S`与`boot/main.c`共同编译而来。`boot/boot.S`完成了从real mode到 32-bit protected mode的转换。`boot/main.c`将kernel读入对应的内存位置。

## boot.S

`boot/boot.S`中的注释写的比较详细：

```
.globl start
start:
  .code16                     # Assemble for 16-bit mode
  cli                         # Disable interrupts
  cld                         # String operations increment
  
  # Set up the important data segment registers (DS, ES, SS).
  xorw    %ax,%ax             # Segment number zero
  movw    %ax,%ds             # -> Data Segment
  movw    %ax,%es             # -> Extra Segment
  movw    %ax,%ss             # -> Stack Segment
  # Enable A20:
  #   For backwards compatibility with the earliest PCs, physical
  #   address line 20 is tied low, so that addresses higher than
  #   1MB wrap around to zero by default.  This code undoes this.
seta20.1:
  inb     $0x64,%al               # Wait for not busy
  testb   $0x2,%al
  jnz     seta20.1
  movb    $0xd1,%al               # 0xd1 -> port 0x64
  outb    %al,$0x64
seta20.2:
  inb     $0x64,%al               # Wait for not busy
  testb   $0x2,%al
  jnz     seta20.2
  movb    $0xdf,%al               # 0xdf -> port 0x60
  outb    %al,$0x60
```

关中断，将ds,es,ss置零，然后enable A20地址线(原因在[x86 assembly](../appendix/x86_assembly.md))

```
  # Switch from real to protected mode, using a bootstrap GDT
  # and segment translation that makes virtual addresses 
  # identical to their physical addresses, so that the 
  # effective memory map does not change during the switch.
  lgdt    gdtdesc
  movl    %cr0, %eax
  orl     $CR0_PE_ON, %eax
  movl    %eax, %cr0
```

先看`lgdt gdtdesc`，手册里对lgdt/lidt的描述为：

```
IF instruction = LIDT
THEN
   IF OperandSize = 16
   THEN IDTR.Limit:Base := m16:24 (* 24 bits of base loaded *)
   ELSE IDTR.Limit:Base := m16:32
   FI;
ELSE (* instruction = LGDT *)
   IF OperandSize = 16
   THEN GDTR.Limit:Base := m16:24 (* 24 bits of base loaded *)
   ELSE GDTR.Limit:Base := m16:32;
   FI;
FI;
```

即lgdt会读取gdtdesc地址处的6个byte，0-15位作为GDTR的limit，16-39位作为base，40-47位unused。结合`gdtdesc`和`gdt`：

```
# obj/boot/boot.asm
00007c4c <gdt>:
	... 
    7c54:	ff                   	(bad)  
    7c55:	ff 00                	incl   (%eax)
    7c57:	00 00                	add    %al,(%eax)
    7c59:	9a cf 00 ff ff 00 00 	lcall  $0x0,$0xffff00cf
    7c60:	00 92 cf 00 17 00    	add    %dl,0x1700cf(%edx)

00007c64 <gdtdesc>:
    7c64:	17                   	pop    %ss
    7c65:	00 4c 7c 00          	add    %cl,0x0(%esp,%edi,2)
    7c69:	00 90 90 55 89 e5    	add    %dl,-0x1a76aa70(%eax)
 # boot/boot.S
 gdt:
  SEG_NULL				# null seg
  SEG(STA_X|STA_R, 0x0, 0xffffffff)	# code seg
  SEG(STA_W, 0x0, 0xffffffff)	        # data seg
gdtdesc:
  .word   0x17                            # sizeof(gdt) - 1
  .long   gdt                             # address gdt
```

从汇编以及对应的反汇编可以看到，gdtdesc处一共定义了6个byte，低地址的2个byte是0x17，也即是GDTR的limit，后面4个byte中存放着gdt的地址。

`lgdt`之后GDTR(48 bit的寄存器)的内容如下：

``` 
47      40 39    32 31    24 23    16 15     8 7      0
+--------+--------+--------+--------+--------+--------+
|   00   |   00   |   7c   |   4c   |   00   |   17   |
+--------+--------+--------+--------+--------+--------+
                  |<---- base ----->|<---- limit ---->|
```

GDT中的每个entry都是8 byte，JOS提供了宏`SEG(type,base,lim)`来设置entry。entry的每个bit定义在`struct Segdesc`中。

```c
struct Segdesc {
	unsigned sd_lim_15_0 : 16;  // Low bits of segment limit
	unsigned sd_base_15_0 : 16; // Low bits of segment base address
	unsigned sd_base_23_16 : 8; // Middle bits of segment base address
	unsigned sd_type : 4;       // Segment type (see STS_ constants)
	unsigned sd_s : 1;          // 0 = system, 1 = application
	unsigned sd_dpl : 2;        // Descriptor Privilege Level
	unsigned sd_p : 1;          // Present
	unsigned sd_lim_19_16 : 4;  // High bits of segment limit
	unsigned sd_avl : 1;        // Unused (available for software use)
	unsigned sd_rsv1 : 1;       // Reserved
	unsigned sd_db : 1;         // 0 = 16-bit segment, 1 = 32-bit segment
	unsigned sd_g : 1;          // Granularity: limit scaled by 4K when set
	unsigned sd_base_31_24 : 8; // High bits of segment base address
}; 
```

`gdt`中一共有三个entry，第一个为空，后两个分别是code段与data段。

```c
// inc/mmu.h
#define STA_X		0x8	    // Executable segment
#define STA_W		0x2	    // Writeable (non-executable segments)
#define STA_R		0x2	    // Readable (executable segments)
// boot/boot.S
 gdt:
  SEG_NULL				# null seg
  SEG(STA_X|STA_R, 0x0, 0xffffffff)	# code seg
  SEG(STA_W, 0x0, 0xffffffff)	        # data seg
```

设置的code与data段的base均为0。

```
  lgdt    gdtdesc
  movl    %cr0, %eax
  orl     $CR0_PE_ON, %eax
  movl    %eax, %cr0
```

要完成实模式到保护模式的切换，需要：

- Disable Interrupt
- Enable A20
- Load GDT for code, data and stack
- Set CR0_PE_ON = 1

因此在执行完`  movl %eax, %cr0`后，就从实模式切换到了保护模式，寻址方式发生变化，用selector作index找GDT中的base。接着执行`ljmp`指令。

```
 .set PROT_MODE_CSEG, 0x8         # kernel code segment selector
 .set PROT_MODE_DSEG, 0x10        # kernel data segment selector
 .set CR0_PE_ON,      0x1         # protected mode enable flag
 
  # Jump to next instruction, but in 32-bit code segment.
  # Switches processor into 32-bit mode.
  ljmp    $PROT_MODE_CSEG, $protcseg
    .code32                     # Assemble for 32-bit mode
protcseg:
  # Set up the protected-mode data segment registers
  movw    $PROT_MODE_DSEG, %ax    # Our data segment selector
  movw    %ax, %ds                # -> DS: Data Segment
  movw    %ax, %es                # -> ES: Extra Segment
  movw    %ax, %fs                # -> FS
  movw    %ax, %gs                # -> GS
  movw    %ax, %ss                # -> SS: Stack Segment
 
  # Set up the stack pointer and call into C.
  movl    $start, %esp
  call bootmain
```

在`ljmp`中的`$PROT_MODE_CSEG`为0x8，选择GDT的第二个entry，其中的base为0，所以进入到protcseg label对应的代码，接着设置好对应的segment selector与stack pointer，跳转到bootmain。bootmain将kernel读入内存。

## bootmain.c

在前面kernel.img已经讲过了kernel是怎么生成出来的。这里的kernel在磁盘的第二个扇区以ELF的形式存在。第一个扇区是bootloader，即boot.S与bootmain.c。

如果你对ELF不太了解，可以参考Appendix中的[ELF](../appendix/elf.md)一章。

首先将ELF的header从磁盘中读入到内存中：

```c
#define ELFHDR		((struct Elf *) 0x10000) // scratch space	
// read 1st page off disk
	readseg((uint32_t) ELFHDR, SECTSIZE*8, 0);
```

这里ELFHDR存放在内存的地址应该是没被占用的任意地址(?)。ELF header的大小是固定的, 7f 45 4c 46是ELF文件的magic number，对应的ascii码为`elf.`。

```shell
$ readelf -h obj/kern/kernel
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Intel 80386
  Version:                           0x1
  Entry point address:               0x10000c
  Start of program headers:          52 (bytes into file)
  Start of section headers:          108968 (bytes into file)
  Flags:                             0x0
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         2
  Size of section headers:           40 (bytes)
  Number of section headers:         11
  Section header string table index: 8
```

我们需要将Program Header中指明的段读入内存对应位置，具体位置由program header给出

```shell
$ i386-jos-elf-objdump -h obj/kern/kernel

obj/kern/kernel:     file format elf32-i386

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         0000178e  f0100000  00100000  00001000  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .rodata       00000704  f01017a0  001017a0  000027a0  2**5
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .stab         000044d1  f0101ea4  00101ea4  00002ea4  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .stabstr      00008afa  f0106375  00106375  00007375  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .data         0000a300  f010f000  0010f000  00010000  2**12
                  CONTENTS, ALLOC, LOAD, DATA
  5 .bss          00000648  f0119300  00119300  0001a300  2**5
                  CONTENTS, ALLOC, LOAD, DATA
```

`.stab, .stabstr`段用于debug。

将每一个段都读到program header中指定的`ph->p_pa`处，这里的`ph->p_pa`即为`LMA`，Linear Memory Address，是经过segment selector 映射后的地址，可以理解为memory的物理地址。


```c
	ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
	eph = ph + ELFHDR->e_phnum;
	for (; ph < eph; ph++)
		// p_pa is the load address of this segment (as well
		// as the physical address)
		readseg(ph->p_pa, ph->p_memsz, ph->p_offset);

	// call the entry point from the ELF header
	// note: does not return!
	((void (*)(void)) (ELFHDR->e_entry))();
```

读完之后，利用一个C语言的小trick，跳转到`e_entry`处，elf header中指明了`Entry point address:             0x10000c `，即`e_entry = 0x10000c`，跳转到kernel中的entry处。

```
.globl entry
entry:
	movw	$0x1234,0x472			# warm boot
f0100000:	02 b0 ad 1b 00 00    	add    0x1bad(%eax),%dh
f0100006:	00 00                	add    %al,(%eax)
f0100008:	fe 4f 52             	decb   0x52(%edi)
f010000b:	e4 66                	in     $0x66,%al

f010000c <entry>:
f010000c:	66 c7 05 72 04 00 00 	movw   $0x1234,0x472
f0100013:	34 12 
```

注意这里的虚拟地址是`f010000c`而非entry的物理地址`010000c`，这一点在下一个部分中会详细说明。

==todo: warm boot代码是什么意思？为什么需要warm boot== 

接着来看一下这个C语言的小trick，一层一层来拆：

```c
((void (*)(void)) (ELFHDR->e_entry))();
```

这里是函数指针的运用，最后一个括号是函数的参数，为空，是对这个函数的调用，因此这个函数是

```c
(void (*)(void)) (ELFHDR->e_entry)
```

这里是将`ELFHDR->e_entry`转化成一个函数指针，类似于类型转换，只看转化的类型：

```c
void (*)(void)
int (*pf)()
```

这个就跟常见的函数指针的声明类似了，只不过这个函数指针并没有名字，这个函数指针对应的函数的返回值类型为void，参数类型为void。

所以这一句可以替换成简单的inline assembly

```
asm volatile("jmp *%0" : : "r" (ELFHDR->e_entry));
```

跳转之后进入kernel。
