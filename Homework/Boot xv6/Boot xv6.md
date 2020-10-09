# Boot xv6

这个作业要求我们启动 `xv6` 系统，并且在进入内核的内存位置处 `0x10000c` 打上断点，运行到该处后查看寄存器的内容，并分析当前栈上的值的含义。

```
(gdb) b *0x10000c
Breakpoint 1 at 0x10000c
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0x10000c:    mov    %cr4,%eax

Thread 1 hit Breakpoint 1, 0x0010000c in ?? ()
(gdb) info reg
eax            0x0      0
ecx            0x0      0
edx            0x1f0    496
ebx            0x10054  65620
esp            0x7bbc   0x7bbc
ebp            0x7bf8   0x7bf8
esi            0x100000 1048576
edi            0x1144a8 1131688
eip            0x10000c 0x10000c
eflags         0x46     [ PF ZF ]
cs             0x8      8
ss             0x10     16
ds             0x10     16
es             0x10     16
fs             0x0      0
gs             0x0      0
```

```
(gdb) x/24x $esp
0x7bbc: 0x00007db8      0x00100000      0x00009516      0x00001000
0x7bcc: 0x00000000      0x00000000      0x00000000      0x00000000
0x7bdc: 0x00010054      0x00000000      0x00000000      0x00000000
0x7bec: 0x00000000      0x00000000      0x00000000      0x00000000
0x7bfc: 0x00007c4d      0x8ec031fa      0x8ec08ed8      0xa864e4d0
0x7c0c: 0xb0fa7502      0xe464e6d1      0x7502a864      0xe6dfb0fa
```

在 `0x7c00` 处打上打断，并单步执行，在这一步观察到设置了栈指针，对应的 `bootasm.S` 的该条指令中：

```c
// gdb output
=> 0x7c43:      mov    $0x7c00,%esp

// bootasm.S
movl    $start, %esp
```

即将 `esp` 寄存器的值初始化为 `bootloader` 第一条指令的地址。所以栈会从 `0x7c00` 开始，向低地址增长。

单步运行到 `call bootmain` 指令，该指令调用另外一个函数，且 `call` 指令会把返回地址（即下一条指令 `movw $0x8a00, %ax` 的地址 `0x7c4d` 压入栈中），在执行了call指令之后查看栈中的值如下：

```
(gdb) i r
eax            0x0      0
ecx            0x0      0
edx            0x80     128
**ebx            0x0      0**
**esp            0x7bfc   0x7bfc**              // 可以看到esp比0x7c00小4，因为存储了一个返回地址
**ebp            0x0      0x0**
**esi            0x0      0**
**edi            0x0      0**
eip            0x7d2a   0x7d2a
eflags         0x6      [ PF ]
cs             0x8      8
ss             0x10     16
ds             0x10     16
es             0x10     16
fs             0x0      0
gs             0x0      0
(gdb) x/24x $esp
0x7bfc: **0x00007c4d**      0x8ec031fa      0x8ec08ed8      0xa864e4d0
0x7c0c: 0xb0fa7502      0xe464e6d1      0x7502a864      0xe6dfb0fa
0x7c1c: 0x16010f60      0x200f7c78      0xc88366c0      0xc0220f01
0x7c2c: 0x087c31ea      0x10b86600      0x8ed88e00      0x66d08ec0
0x7c3c: 0x8e0000b8      0xbce88ee0      0x00007c00      0x0000dde8
0x7c4c: 0x00b86600      0xc289668a      0xb866ef66      0xef668ae0
```

而且我们可以看到 `0x7bfc` 存储的就是返回地址 **`0x00007c4d` 。 `ebx` , `ebp` , `esi` , `edi` 四个寄存器是之后 `bootmain` 函数的需要使用的寄存器，会先将寄存器的值压入栈中保存。**

在进入了 `bootmain` 函数后，由于calling convention的关系，会先将 `ebp` 的值压入栈，并将 `bootmain` 中需要使用到的寄存器的值也压入栈中：

```
void
bootmain(void)
{
    7d2a:	55                   	push   %ebp
    7d2b:	89 e5                	mov    %esp,%ebp
    7d2d:	57                   	push   %edi
    7d2e:	56                   	push   %esi
    7d2f:	53                   	push   %ebx
```

所以此时 `esp` 应该变为 `0x7bec` ，且之后的4个双字都是0，分别表示被压入的四个寄存器的值：

```
(gdb) i r $esp
esp            0x7bec   0x7bec
(gdb) x/5x $esp
**0x7bec: 0x00000000      0x00000000      0x00000000      0x00000000**
0x7bfc: 0x00007c4d
```

同时，下一条指令为 `bootmain` 预留了 `0x2c` 作为函数保存局部变量的空间，则此时 `esp` 寄存器的值变为 `0x7bc0` ：

```assembly
7d30:	83 ec 2c             	sub    $0x2c,%esp
```

因此，在当前 `bootmain` 函数中， `esp` 不会再向下增长。

接下来的指令是将 `readseg` 函数需要的三个参数压入栈中，并调用 `readseg` 函数，这里 `call` 指令会把下一条指令压入栈中，但是我们这里并不关心 `bootmain` 中进入 `entry` 之前的其他的函数调用，因为都会在进入 `entry` 之前返回：

```assembly
7d33:	c7 44 24 08 00 00 00 	movl   $0x0,0x8(%esp)
7d3a:	00 
7d3b:	c7 44 24 04 00 10 00 	movl   $0x1000,0x4(%esp)
7d42:	00 
7d43:	c7 04 24 00 00 01 00 	movl   $0x10000,(%esp)
```

在执行了 `entry()` 之后， `eip` 寄存器的值会变为 `0x10000c` ，且 `esp` 由于压入了返回地址 `0x7db8` 之后，会再减去4，因此 `esp` 的值会变为 `0x7bbc` ：

```c
// Call the entry point from the ELF header.
// Does not return!
entry = (void(*)(void))(elf->entry);
entry();
```

```assembly
(gdb) i r $esp
esp            0x7bbc   0x7bbc
(gdb) i r $eip
eip            0x10000c 0x10000c
```

```asm
(gdb) x/24x $esp
0x7bbc: 0x00007db8      0x00100000      0x00009516      0x00001000
0x7bcc: 0x00000000      0x00000000      0x00000000      0x00000000
0x7bdc: 0x00010054      0x00000000      0x00000000      0x00000000
0x7bec: 0x00000000      0x00000000      0x00000000      0x00000000
0x7bfc: 0x00007c4d 栈底  0x8ec031fa      0x8ec08ed8      0xa864e4d0
0x7c0c: 0xb0fa7502      0xe464e6d1      0x7502a864      0xe6dfb0fa
```

需要注意的是：

1. `0x7bbc` : 存储了 `bootmain` 中调用了 `entry()` 后的下一条指令的地址 `0x7db8` 
2. `0x7bc0` ~ `0x7bc8`：这三个地址主要是用来存放 `bootmain` 函数中的一些临时变量与传递给 `bootmain` 内调用的其他函数的一些参数
3. `0x7bdc` :  用来存储 `bootmain.c` 中的 `eph` 的值，其值为 `0x10054` 

其他的空间则是预留出来且没有使用的。