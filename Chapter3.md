@[TOC](《深入理解计算机系统》第三章)
# 练习题
## 练习题3.1 :假设下面的值存放在指明的内存地址和寄存器中:
|地址|值|
|--|--|
|0x100|0xFF|
|0x104|0xAB|
|0x108|0x13|
|0x10C|0x11|

|寄存器|值|
|--|--|
|%rax|0x100|
|%rcx|0x1|
|%rdx|0x3|

填写下表，给出所示操作数的值:
|操作数|值|
|--|--|
|%rax||
|0x104||
|$0x108||
|(%rax)||
|4(%rax)||
|9(%rax,%rdx)||
|260(%rcx,%rdx)||
|0xFC(,%rcx,4)||
|(%rax,%rdx,4)||

|操作数|值|
|--|--|
|%rax|R[%rax] = 0x100|
|0x104|M[0x104] = 0xAB|
|$0x108|0x108|
|(%rax)|M[R[%rax]] = M[0x100] = 0xFF|
|4(%rax)|M[4 + R[%rax]] = M[0x104] = 0xAB|
|9(%rax,%rdx)|M[9 + R[%rax] + R[%rdx]] = M[9 + 0x100 + 0x3] = M[0x10C] = 0x11|
|260(%rcx,%rdx)|M[260 + R[%rcx] + R[%rdx]] = M[ 260 + 0x1 + 0x3 ] = M[0x108] = 0x13|
|0xFC(,%rcx,4)|M[0xFC + R[%rcx] * 4] = M[0xFC + 0x1 * 4] = M[0x100] = 0xFF|
|(%rax,%rdx,4)|M[R[%rax] + R[%rdx] * 4] = M[0x10C] = 0x11|

## 练习题 3.2: 对于下面汇编代码的每一行，根据操作数，确定适当的指令后。
(例如，mov可以被重写成movb、movw、movl或者movq。)

mov_ %eax,(%rsp)
mov_ (%rax)，%dx
mov_ $0xFF，%bl
mov_ (%rsp,%rdx,4)，%dl
mov_ (%rdx)，%rax
mov_ %dx,(%rax)

1. mov_ %eax,(%rsp)

源 : %eax 是双字  ; 目的: (%rsp) 无法明确判断
所以选择 movl

2. mov_ (%rax)，%dx
目的: %dx 是单字
movw

3. mov_ $0xFF，%bl
属于立即数传给目的:单字节
movb

4. mov_ (%rsp,%rdx,4)，%dl
目的 %dl 是单字节
movb

5. mov_ (%rdx)，%rax
目的 %rax 是四字
movq

6. mov_ %dx,(%rax)
源是 %dx 单字
movw

## 练习题 3.3 当我们调用汇编器的时候，下面代码的每一行都会产生一个错误消息解释每一行都是哪里出了错

movb \$0xF，(%ebx)
movl %rax,(%rsp)
movw (%rax),4(%rsp)
movb %al,%sl
movq %rax,$0x123
movl %eax,%rdx
movb %si,8(%rbp)

### movb \$0xF，(%ebx)
%ebx 是 32位(双字) ， 不能用来作为地址寻址

Cannot use "ebx as address register

### movl %rax,(%rsp)
%rax 四字，应该用 movq

Mismatch between instruction suffix and register ID

### movw (%rax),4(%rsp)
~~两个都是寄存器寻址(或者变种)，应该用movq~~

Cannot have both source and destination be memory references

实际上错误是 不能一条指令直接从内存到内存

### movb %al,%sl
找书上的图发现没有寄存器叫%sl

No register named %sl

### movq %rax,$0x123

目的不能是立即数

Cannot have immediate as destination

### movl %eax,%rdx

movl 会把目标 %rdx 的高16位变为 0

Destination operand incorrect size

### movb %si,8(%rbp)

寄存器%si是一字的，应该用 movw

Mismatch between instruction suffix and register ID

## 练习题 3.4 假设变量sp和dp被声明为类型
```c
src_t *sp;
dest_t *dp;
```
这里 src_t和 dest_t是用 typedef声明的数据类型。
我们想使用适当的数据传送指令来实现下面的操作.
```*dp=(dest_t)*sp```
假设 sp和 dp 的值分别存储在寄存器%rdi 和%rsi中。
对于表中的每个表项，给出实现指定数据传送的两条指令。
其中第一条指令应该从内存中读数，做适当的转换，并设置寄存器%rax的适当部分。
然后，第二条指令要把%rax的适当部分写到内存。
在这两种情况中，寄存器的部分可以是%rax、%eax、%ax或%al，两者可以互不相同。
记住，当执行强制类型转换既涉及大小变化又涉及C语言中符号变化时，操作应该先改变大小(2.2.6节)。

|src_t|dest_t|指令|
|--|--|--|
|long| long|movq (%rdi),%rax</br> movq %rax,(%rsi)|
|char| int|movsbl (%rdi),%eax</br>movl %eax,(%rsi)|
|char| usigned|movsbl (%rdi),%eax</br>movl %eax,(%rsi)|
|usigned char| long|movzbq (%rdi),%rax</br>movq %rax,(%rsi)|
|int| char||
|usigned| usigned char||
|char| short||
## 练习题 3.5 已知信息如下。将一个原型为

```c
void decode1(long *xp,long *yp, long *zp);
```
的函数编译成汇编代码，得到如下代码:
```c
void decode1(long *xp, long *yp, long *zp);
xp in %rdi, yp in %rsi, zp in %rdx
decode1:
    movq (%rdi)，%r8
    movq (%rsi)，%rcx
    movq (%rdx)，%rax
    movq %r8，(%rsi)
    movq %rcx,(%rdx)
    movq %rax，(%rdi)
    ret 
```

参数xp、yp和zp分别存储在对应的寄存器%rdi、%rsi和%rdx中
请写出等效于上面汇编代码的 decode1的C代码

```c
void decode1(long *xp, long *yp, long *zp);
xp in %rdi, yp in %rsi, zp in %rdx
decode1:
    movq (%rdi)，%r8    t1 = *xp
    movq (%rsi)，%rcx   t2 = *yp
    movq (%rdx)，%rax   t3 = *zp
    movq %r8，(%rsi)    *yp = t1
    movq %rcx,(%rdx)    *zp = t2
    movq %rax，(%rdi)   *xp = t3
    ret 
```
```c
void decode1(long *xp, long *yp, long *zp)[
    t1 = *xp
    t2 = *yp
    t3 = *zp
    *yp = t1
    *zp = t2
    *xp = t3
]
```
## 练习题 3.6 假设寄存器%rax的值为x,%rcx的值为 y。填写下表，指明下面每条汇编代码指令存储在寄存器%rdx中的值:
|表达式|结果|
|--|--|
|leaq 6(%ax),%rdx|6+(short)x|
|leaq (%rax,%rcx),%rdx|x + y|
|leaq (%rax,%rcx,4),%rdx|4y+x|
|leaq 7(%rax,%rax,8),%rdx|7+9x|
|leaq 0xA(,%rcx,4),%rdx|0xA + 4y|
|leaq 9(%rax,%rcx,2),%rdx|9 + 2y + x|

## 练习题 3.7
```c
long scale2(long x,long y,long z){
    long t = ________;
    return t;
}
```
```gcc
x in %rdi , y in %rsi , z in %rdx
scale2:
    leaq (%rdi,%rdi,4),%rax
    leaq (%rax,%rsi,2),%rax
    leaq (%rax,%rdx,8),%rax
ret
```

填写出C代码中缺失的表达式
```gcc
scale2:
    leaq (%rdi,%rdi,4),%rax  %rax = x + 4x
    leaq (%rax,%rsi,2),%rax  %rax = %rax + 2y
    leaq (%rax,%rdx,8),%rax  %rax = %rax + 8z
ret
```

```c
long scale2(long x,long y,long z){
    long t = 5x + 2y + 8z;
    return t;
}
```

# 课后习题
## 3.58
一个函数原型如下:
```c
long decode2(long x,long y,long z);
```
```gcc
decode2:
    subq    %rdx,%rsi
    imulq   %rsi,%rdi
    movq    %rsi,%rax
    salq    $63,%rax
    sarq    $63,%rax
    xorq    %rdi,%rax
```
```text
x in %rdi , y in %rsi , z in %rdx
```
参数x、y和z通过寄存器号%rdi、%rsi和%rdx传递。
代码将返回值存放在寄存器%rax中写出等价于上述汇编代码的decode2的C代码

分析
```gcc
decode2:
    subq    %rdx,%rsi y = y - z
    imulq   %rsi,%rdi x = x * y
    movq    %rsi,%rax t = y
    salq    $63,%rax  t <<= 63
    sarq    $63,%rax  t >>= 63 (算术右移)
    xorq    %rdi,%rax t = t ^ x
```

```c
long decode2(long x,long y,long z){
    t = y - z;
    x *= y - z;
    t <<= 63;
    t >>= 63;
    return t^x;
}
```
## 3.59 下面的代码计算两个64位有符号值x和y的128位乘积，并将结果存储在内存中:
```c
typedef _int128 int128_t;
void store_prod(int128_t *dest,int64_t x,int64_t y){
    *dest=x*(int128_t)y;
} 
```
GCC产出下面的汇编代码来实现计算:
```c 
store_prod:
    movq %rdx，%rax
    cqto
    movq %rsi,%rcx
    sarq $63,%rcx
    imulq %rax,%rcx
    imulq %rsi,%rdx
    addq %rdx,%rcx
    mulq %rsi
    addq %rcx,%rdx
    movq %rax,(%rdi)
    movq %rdx,8(%rdi)
    ret
```
为了满足在64位机器上实现128位运算所需的多精度计算，这段代码用了三个乘法。
描述用来计算乘积的算法，对汇编代码加注释，说明它是如何实现你的算法的。
提示:在把参数x和y扩展到128位时，它们可以重写为$x=2^{64}·x_h+x_l$和$y = 2^{64}+y_h+y_l$，这里$x_h,x_l,y_h,y_l$都是64位值。类似地，128位的乘积可以写成$p=2^{64} * p_h + p_l$，这里$p_h,p_l$都是64位值。
请解释这段代码是如何用$x_h,x_l,y_h,y_l$的。