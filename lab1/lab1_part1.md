# Part 1: PC Bootstrap



## Getting Started with x86 Assembly

每一个lab都由若干个exercise组成，有些是简单的问答题，有些则需要动手写代码。第一个exercise的内容如下，要求熟悉x86汇编。

> **Exercise 1.** Familiarize yourself with the assembly language materials available on [the 6.828 reference page](https://pdos.csail.mit.edu/6.828/2018/reference.html). You don't have to read them now, but you'll almost certainly want to refer to some of this material when reading and writing x86 assembly.
>
> We do recommend reading the section "The Syntax" in [Brennan's Guide to Inline Assembly](http://www.delorie.com/djgpp/doc/brennan/brennan_att_inline_djgpp.html). It gives a good (and quite brief) description of the AT&T assembly syntax we'll be using with the GNU assembler in JOS.

如果你对x86汇编不熟悉，可以参考appendix中[x86_assembly](../appendix/x86_assembly.md)对x86汇编的简单介绍，主要包括x86汇编的基本用法，实模式与保护模式的基本概念等。

在6.828中使用的x86汇编格式是AT&T，而非Intel格式。

## Simulating the x86

6.828中使用qemu来模拟（80836）PC机，方便调试。

课程代码中已经为我们写好了编译jos与使用qemu运行jos的Makefile。编译只需要执行`make`:

```shell
$ make
+ as kern/entry.S
+ cc kern/entrypgdir.c
+ cc kern/init.c
+ cc kern/console.c
+ cc kern/monitor.c
+ cc kern/printf.c
+ cc kern/kdebug.c
+ cc lib/printfmt.c
+ cc lib/readline.c
+ cc lib/string.c
+ ld obj/kern/kernel
i386-jos-elf-ld: warning: section `.bss' type changed to PROGBITS
+ as boot/boot.S
+ cc -Os boot/main.c
+ ld boot/boot
boot block is 382 bytes (max 510)
+ mk obj/kern/kernel.img
```

 经过编译之后得到的是`obj/kern/kernel.img`，如果你有装系统的经验，这里的`kernel.img`类似于`iso`文件，是系统镜像。

用qemu运行这个“系统”只需要执行`make qemu-nox`，`-nox`会将qemu的内容直接输出到当前的terminal。

```shell
$ make qemu-nox
sed "s/localhost:1234/localhost:26000/" < .gdbinit.tmpl > .gdbinit
***
*** Use Ctrl-a x to exit qemu
***
qemu-system-i386 -nographic -drive file=obj/kern/kernel.img,index=0,media=disk,format=raw -serial mon:stdio -gdb tcp::26000 -D qemu.log
6828 decimal is XXX octal!
entering test_backtrace 5
entering test_backtrace 4
entering test_backtrace 3
entering test_backtrace 2
entering test_backtrace 1
entering test_backtrace 0
leaving test_backtrace 0
leaving test_backtrace 1
leaving test_backtrace 2
leaving test_backtrace 3
leaving test_backtrace 4
leaving test_backtrace 5
Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
K>
```

到这里我们成功的boot了jos，如果在整个过程中出错，大概率是工具链配置错误，最简单的解决方法是选择使用虚拟机:)。

### What happens when make & make qemu ?

如果你对`make`与`make qemu-nox`的具体细节不感兴趣，可以跳过。但我强烈建议你读一读这一部分内容。

执行`make clean`:

```shell
$ make clean
rm -rf obj .gdbinit jos.in qemu.log
```

`make clean`删掉了`obj`文件夹，obj文件夹里面有一些编译的中间产物(.o或者.out文件)以及方便查看的反汇编文件(.asm)。同时也删除了其余用于调试或者测试的中间文件。

重新执行`make`，这次使用`make -n`或者`make V=1`

> `make -n`会打印出所有将要执行的命令，但并不执行。执行`man make`去读make的相关文档：
> -n, --just-print, --dry-run, --recon
> Print the commands that would be executed, but do not execute them (except in certain circumstances).
>
> `make V=1`打印并执行所有命令。

```shell
$ make V=1
echo "   -O1 -fno-builtin -I. -MD -fno-omit-frame-pointer -std=gnu99 -static -Wall -Wno-format -Wno-unused -Werror -gstabs -m32 -fno-tree-ch -fno-stack-protector -DJOS_KERNEL -gstabs" | cmp -s obj/.vars.KERN_CFLAGS || echo "   -O1 -fno-builtin -I. -MD -fno-omit-frame-pointer -std=gnu99 -static -Wall -Wno-format -Wno-unused -Werror -gstabs -m32 -fno-tree-ch -fno-stack-protector -DJOS_KERNEL -gstabs" > obj/.vars.KERN_CFLAGS
+ as kern/entry.S
i386-jos-elf-gcc -pipe -nostdinc    -O1 -fno-builtin -I. -MD -fno-omit-frame-pointer -std=gnu99 -static -Wall -Wno-format -Wno-unused -Werror -gstabs -m32 -fno-tree-ch -fno-stack-protector -DJOS_KERNEL -gstabs -c -o obj/kern/entry.o kern/entry.S
+ cc kern/entrypgdir.c
i386-jos-elf-gcc -pipe -nostdinc    -O1 -fno-builtin -I. -MD -fno-omit-frame-pointer -std=gnu99 -static -Wall -Wno-format -Wno-unused -Werror -gstabs -m32 -fno-tree-ch -fno-stack-protector -DJOS_KERNEL -gstabs -c -o obj/kern/entrypgdir.o kern/entrypgdir.c
echo "" | cmp -s obj/.vars.INIT_CFLAGS || echo "" > obj/.vars.INIT_CFLAGS
+ cc kern/init.c
i386-jos-elf-gcc -pipe -nostdinc    -O1 -fno-builtin -I. -MD -fno-omit-frame-pointer -std=gnu99 -static -Wall -Wno-format -Wno-unused -Werror -gstabs -m32 -fno-tree-ch -fno-stack-protector -DJOS_KERNEL -gstabs  -c -o obj/kern/init.o kern/init.c
+ cc kern/console.c
i386-jos-elf-gcc -pipe -nostdinc    -O1 -fno-builtin -I. -MD -fno-omit-frame-pointer -std=gnu99 -static -Wall -Wno-format -Wno-unused -Werror -gstabs -m32 -fno-tree-ch -fno-stack-protector -DJOS_KERNEL -gstabs -c -o obj/kern/console.o kern/console.c
+ cc kern/monitor.c
i386-jos-elf-gcc -pipe -nostdinc    -O1 -fno-builtin -I. -MD -fno-omit-frame-pointer -std=gnu99 -static -Wall -Wno-format -Wno-unused -Werror -gstabs -m32 -fno-tree-ch -fno-stack-protector -DJOS_KERNEL -gstabs -c -o obj/kern/monitor.o kern/monitor.c
+ cc kern/printf.c
i386-jos-elf-gcc -pipe -nostdinc    -O1 -fno-builtin -I. -MD -fno-omit-frame-pointer -std=gnu99 -static -Wall -Wno-format -Wno-unused -Werror -gstabs -m32 -fno-tree-ch -fno-stack-protector -DJOS_KERNEL -gstabs -c -o obj/kern/printf.o kern/printf.c
+ cc kern/kdebug.c
i386-jos-elf-gcc -pipe -nostdinc    -O1 -fno-builtin -I. -MD -fno-omit-frame-pointer -std=gnu99 -static -Wall -Wno-format -Wno-unused -Werror -gstabs -m32 -fno-tree-ch -fno-stack-protector -DJOS_KERNEL -gstabs -c -o obj/kern/kdebug.o kern/kdebug.c
+ cc lib/printfmt.c
i386-jos-elf-gcc -pipe -nostdinc    -O1 -fno-builtin -I. -MD -fno-omit-frame-pointer -std=gnu99 -static -Wall -Wno-format -Wno-unused -Werror -gstabs -m32 -fno-tree-ch -fno-stack-protector -DJOS_KERNEL -gstabs -c -o obj/kern/printfmt.o lib/printfmt.c
+ cc lib/readline.c
i386-jos-elf-gcc -pipe -nostdinc    -O1 -fno-builtin -I. -MD -fno-omit-frame-pointer -std=gnu99 -static -Wall -Wno-format -Wno-unused -Werror -gstabs -m32 -fno-tree-ch -fno-stack-protector -DJOS_KERNEL -gstabs -c -o obj/kern/readline.o lib/readline.c
+ cc lib/string.c
i386-jos-elf-gcc -pipe -nostdinc    -O1 -fno-builtin -I. -MD -fno-omit-frame-pointer -std=gnu99 -static -Wall -Wno-format -Wno-unused -Werror -gstabs -m32 -fno-tree-ch -fno-stack-protector -DJOS_KERNEL -gstabs -c -o obj/kern/string.o lib/string.c
echo "-m elf_i386 -T kern/kernel.ld -nostdlib" | cmp -s obj/.vars.KERN_LDFLAGS || echo "-m elf_i386 -T kern/kernel.ld -nostdlib" > obj/.vars.KERN_LDFLAGS
+ ld obj/kern/kernel
i386-jos-elf-ld -o obj/kern/kernel -m elf_i386 -T kern/kernel.ld -nostdlib obj/kern/entry.o obj/kern/entrypgdir.o obj/kern/init.o obj/kern/console.o obj/kern/monitor.o obj/kern/printf.o obj/kern/kdebug.o  obj/kern/printfmt.o  obj/kern/readline.o  obj/kern/string.o /usr/local/lib/gcc/i386-jos-elf/4.6.4/libgcc.a -b binary
i386-jos-elf-ld: warning: section `.bss' type changed to PROGBITS
i386-jos-elf-objdump -S obj/kern/kernel > obj/kern/kernel.asm
i386-jos-elf-nm -n obj/kern/kernel > obj/kern/kernel.sym
+ as boot/boot.S
i386-jos-elf-gcc -pipe -nostdinc    -O1 -fno-builtin -I. -MD -fno-omit-frame-pointer -std=gnu99 -static -Wall -Wno-format -Wno-unused -Werror -gstabs -m32 -fno-tree-ch -fno-stack-protector -DJOS_KERNEL -gstabs -c -o obj/boot/boot.o boot/boot.S
+ cc -Os boot/main.c
i386-jos-elf-gcc -pipe -nostdinc    -O1 -fno-builtin -I. -MD -fno-omit-frame-pointer -std=gnu99 -static -Wall -Wno-format -Wno-unused -Werror -gstabs -m32 -fno-tree-ch -fno-stack-protector -DJOS_KERNEL -gstabs -Os -c -o obj/boot/main.o boot/main.c
+ ld boot/boot
i386-jos-elf-ld -m elf_i386 -N -e start -Ttext 0x7C00 -o obj/boot/boot.out obj/boot/boot.o obj/boot/main.o
i386-jos-elf-objdump -S obj/boot/boot.out >obj/boot/boot.asm
i386-jos-elf-objcopy -S -O binary -j .text obj/boot/boot.out obj/boot/boot
perl boot/sign.pl obj/boot/boot
boot block is 382 bytes (max 510)
+ mk obj/kern/kernel.img
dd if=/dev/zero of=obj/kern/kernel.img~ count=10000 2>/dev/null
dd if=obj/boot/boot of=obj/kern/kernel.img~ conv=notrunc 2>/dev/null
dd if=obj/kern/kernel of=obj/kern/kernel.img~ seek=1 conv=notrunc 2>/dev/null
mv obj/kern/kernel.img~ obj/kern/kernel.img
```

这里的命令表面上有一点复杂，但仔细看发现大致可以分为三个部分。

第一个部分就是将所有的c语言源代码编译成obj对应目录下的.o文件。第二个部分是用ld链接这些.o文件。最后是生成img文件。我们一个部分一个部分来看：

#### 编译

第一个部分的编译是trivial的，主要是编译的CFLAGS需要大概了解，RTFM `man gcc`:

`-fno-builtin`: Don't recognize built-in functions that do not begin with __builtin_ as prefix. 不使用gcc的builtin的function。

`-I .`: 在`.`文件下找header files。

`-MD`:  The driver determines file based on whether an -o option is given.  If it is, the driver uses its argument but with a suffix of .d, otherwise it takes the name of the input file, removes any directory components and suffix, and applies a .d suffix. 控制输出`.d`文件

`-fno-onmit-frame-pointer`: 不函数调用时的frame pointer信息。

`-fno-stack-protector`: 不check buffer overflow。

`-fno-tree-ch`：某种编译优化。

#### 链接

编译得到`obj`目录下的`.o`文件，然后经过第二步的链接。与链接相关的一共有两条命令：

```shell
i386-jos-elf-ld -o obj/kern/kernel -m elf_i386 -T kern/kernel.ld -nostdlib obj/kern/entry.o obj/kern/entrypgdir.o obj/kern/init.o obj/kern/console.o obj/kern/monitor.o obj/kern/printf.o obj/kern/kdebug.o  obj/kern/printfmt.o  obj/kern/readline.o  obj/kern/string.o /usr/local/lib/gcc/i386-jos-elf/4.6.4/libgcc.a -b binary
i386-jos-elf-ld -m elf_i386 -N -e start -Ttext 0x7C00 -o obj/boot/boot.out obj/boot/boot.o obj/boot/main.o
```

第一条命令指定了输出文件为`obj/kern/kernel`，根据`kern/kernel.ld`的指令来进行链接。

`kern/kernel.ld`中我们只需要关注两个地方:

`	. = 0xF0100000;` 定义了kernel的起始地址，kernel被放入内存的位置，这里的地址是虚拟地址。

`PROVIDE(end = .);` 定义了`end`符号，kernel结束的地方。

第二条命令指定了输出文件为`obj/boot/boot.out`，同时指定了这个ELF文件的入口为start符号(`-e start`，`start`在`boot/boot.S`中定义)，还指定了text段的起始地址为0x7C00(`-Ttext 0x7C00`)。

#### 生成img

得到两个ELF文件之后，接着要生成`kernel.img`系统镜像，qemu以`kernel.img`作为输入。与`kerenel.img`相关的命令有

```shell
i386-jos-elf-objcopy -S -O binary -j .text obj/boot/boot.out obj/boot/boot
perl boot/sign.pl obj/boot/boot
boot block is 382 bytes (max 510)
+ mk obj/kern/kernel.img
dd if=/dev/zero of=obj/kern/kernel.img~ count=10000 2>/dev/null
dd if=obj/boot/boot of=obj/kern/kernel.img~ conv=notrunc 2>/dev/null
dd if=obj/kern/kernel of=obj/kern/kernel.img~ seek=1 conv=notrunc 2>/dev/null
mv obj/kern/kernel.img~ obj/kern/kernel.img
```

链接得到的`obj/boot/boot.out`中的text段被取出来copy到`obj/boot/boot`，在执行`boot/sign.pl`,其作用为将`obj/boot/boot`填充到512个字节，并将最后两个字节写为`0x55 0xAA`。

```shell
# perl
$buf .= "\0" x (510-$n) # 将$n到510byte全部填充为0
$buf .= "\x55\xAA";	# 最后两个byte写为55AA
# shell
$ file obj/boot/boot
obj/boot/boot: DOS/MBR boot sector
```

`obj/boot/boot`就成为MBR，主引导扇区。主引导扇区的最后两个字节为magic number 0xAA55。而上面写入的是0x55AA，这是由于80386是小端。

```shell
$ xxd -l 512 obj/boot/boot
00000000: fafc 31c0 8ed8 8ec0 8ed0 e464 a802 75fa  ..1........d..u. 
00000010: b0d1 e664 e464 a802 75fa b0df e660 0f01  ...d.d..u....`..
...                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           
000001e0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001f0: 0000 0000 0000 0000 0000 0000 0000 55aa  ..............U.
```

接着是几条`dd`开头的命令

从input file: /dev/zero,复制10000个bs (大小默认为512 byte) 到output file: obj/kern/kernel.img~，即kernel.img~中内容全为0。

将`obj/boot/boot`中的内容复制到kernel.img~，conv=notrunc表示不要截断output文件。

将`obj/ker/kernel`中的内容复制到kernel.img~，seek=1 表示跳过第一个bs，即512 byte。

即将`obj/boot/boot`放在第一个扇区，作为MBR，`obj/kern/kernel`放在随后的扇区中。

```
 kernel.img
    0       1        2
 +--------+--------+--------+
 |  boot  | kernel | kernel |....
 +--------+--------+--------+
```

