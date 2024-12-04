# 前置知识
call 指令
ret 指令
寄存器rip
栈

**栈向下生长，栈顶在低地址，栈底在高地址**

**指令寄存器（RIP）包含下一条将要被执行的指令的逻辑地址。**

通常情况下，每取出一条指令后，RIP会自增指向下一条指令。在x86_64中RIP的自增也即偏移一定字节。（可通过disassemble 查看下一个地址，字节大小不一定等长）

但是RIP并不总是自增，也有例外，例如call 指令和ret指令。call指令会将当前RIP的内容压入栈中，将程序的执行权交给目标函数；ret指令则执行出栈操作，将之前压入栈的8个字节的RIP地址弹出，重新放入RIP。

更多信息参考
[X86/ARM 寄存器](https://www.cnblogs.com/tzj-kernel/p/17960135)
[[register]-ARMV8系统中通用寄存器和系统寄存器的介绍和总结](https://www.bilibili.com/opus/913806141088071748)

# 前置准备工作
下载实验包Attack Lab
可以在这个网站下下周 [Lab Assignments](https://csapp.cs.cmu.edu/3e/labs.html)

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
(注：layout asm ; layout regs这两条目前是可选选项)

这个时候输入``x/80xb $rsp``打印栈的内容发现401976地址在栈顶，再往前看是401f24，这两个地址刚好都是分别调用完``getbuf()``函数和``test()``函数的下一个指令的地址。
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