---
layout: post
title: "机器指令"
category: Csapp (cmu 15-213)
---
CSAPP 机器指令基本概念

## Machine Level Programming - I Basic

### Moving Data

moveq [source],[dest]    (by the way  moveq的q代表quad)

source:立即数， 寄存器，内存

dest： 寄存器 内存

(r)-->mem[reg[r]]

d(r)--->mem[reg[r]+d]  

d(rb,ri,s) --->mem[reg[rb]+s*reg[ri]+d] rb代表基地址，ri代表索引，s代表索引长度，一般用于数组

例如leaq 4(%rdi,%rdx) ,%rcx ====>rdi+rdx+4存到rcx中

### 地址计算指令

leaq [src] [dest]   src一个地址，dest寄存器，直接算，不用从地址取值

1. 相当与c语言中的&  

2. 计算x+k*y，k=1，2，4，8    例如  leal (%eax, %eax, 2) , % eax

**当用法第一种时**

%eax = 0x1

adress : 0x1---> value: 0x88

adress:0x3---> value 0x77

那么 

 leal 2(%eax) , % eax 执行完之后， eax的值是0x3（指针）

movl 2(%eax) , % eax 执行完之后，eax的值是0x77（指针的值）

**当用法第二种时**

x*12   

 leaq(%rdi,%rdi,2),%rax

sqlq $2, %rax

rax=rdi+2rdi=x*3

然后再rax的值左移两位

## Machine Level Programming - II-Control

### 条件码

CF carry flag  进位

 ZF  zero flag 0标志

SF  sing flag 负数

OF  补码溢出

用途

cmpq src2 src1，根据两个数之差来设置条件码

### 条件分支

第一种使用有条件jump，判断然后跳转到指定代码行



第二种使用conditional move，两个结果计算出来，然后选择我们需要的

(由于流水线技术，if中的判断，if的结果，else的结果都可以计算出来，性能更好)



## Machine Level Programming - III - Procedures

1. 传递控制（方法调用
2. 传递值  (传参数
3. 内存管理 （分配内存，回收内存

### Stack

用栈来管理procedure的过程，stack越下面地址越大，越往上越小

POP : read  by %rsp， %rsp++,  value to dest

PUSH: fetch  from src,  %rsp--, value to %rsp所指的地方

### 方法调用过程

%rsp指向当前栈里面某处地址，%rip指向程序执行的地址

当发生调用做两件事情，

	1. %rsp--，栈里保存的是方法调用完成后的下一条指令的地址（等方法返回继续从这里执行），
	1. %rip(pc)指向被调用代码的位置

###  1. <a name='register1'></a>寄存器分类

 %rbx,%rbp, %r12–%r15 are classified as **callee-saved registers**

%rdi  %rsi  %rdx  %rcx   %r8 %r9   **caller-saved registers**

![image-20231113114233452]({{"/assets/procedure/image-20231113114233452.png" | absolute_url }})

%rsp  %rbp栈指针：%rbp% 指向栈底， %rsp指向栈顶来，限制栈帧的范围

%rax 保存function 的返回结果

%rip 下一条将要被执行的指令的逻辑地址

###  2. <a name='Stack'></a>Stack

![image-20231114155735683]({{ "/assets/procedure/image-20231114155735683.png" | absolute_url }})
![image-20231114155112689]({{ "/assets/procedure/image-20231114155112689.png" | absolute_url }})

存三种值，假如P call Q

1. Arguement区：当前栈帧的为下一个被调用函数栈帧构造参数的区域
2. LocalVariables区：当前栈帧的函数里的局部变量
3. Saved regiisters区: 当前栈帧的调用函数的寄存器值，当P再使用rsi和rsi，又调用了Q，Q也要使用rdi rsi，所以需要先保存P的寄存器值，防止被重写

### 3. <a name='ControlTransfer'></a>Control Transfer

call：被调用方法返回后的下一条指令地址（return address）push 到stack，然后pc设置为被调用方法的首地址

ret：pop出return address，然后pc设置为return address

###  4. <a name='DataTransferarguments'></a>Data Transfer（1.Arguement区）

![image-20231113135005671]({{ "/assets/procedure/image-20231113135005671.png" | absolute_url }})

例如proc(a1,a1p,a2,a2p,a3,a3p,a4,a4p)

首先前6个参数  a1,a1p,a2,a2p,a3,a3p 存在寄存器中

之后超过6个的参数先  a4p 进栈， 在a4进栈(Arguement区), （从后往前进）

**注意：Arguement区是为被调用函数的构造参数的区域，下一个函数需要哪些参数从这里面取**

![image-20231113135024111]({{ "/assets/procedure/image-20231113135024111.png" | absolute_url }})

###  5. <a name='LocalStorageonStackLocalvariables'></a>Local Storage on Stack（2.Local variables区）

局部变量存在stack的例子，局部变量也是按倒序来存的



![image-20231113152245467]({{ "/assets/procedure/image-20231113152245467.png" | absolute_url }})

![image-20231113152301378]({{ "/assets/procedure/image-20231113152301378.png" | absolute_url }})

上图中分为两个部分

1.  x1, x2, x3,x4,四个局部变量所以，这四个参数存在local variable区，如下图 **局部变量在stack不用对齐**

2. 之后还要构造八个参数x1, x2, x3,x4，&x1,& x2,&x3,&x4，6个在寄存器中，剩下两个在**Local variables 区,参数变量在stack要对齐**

![image-20231113152437223]({{ "/assets/procedure/image-20231113152437223.png" | absolute_url }})

###  6. <a name='LocalStorageinRegistersSavedregisters'></a>Local Storage in Registers（3.Saved registers区）

Saved registers分两种 callee-save和calleer-save，用哪种都可以。



P调用Q，P的寄存器(例如rdi,rsi), 可能也会被Q用到，那么Q就会覆盖掉P寄存器的内容，所以使用rbx或rbp（calleer-save regster），这种Q用不到的（无法覆盖）寄存器先去保存（rdi，rsi）到P的栈帧中，然后等Q用完再弹出来即可恢复数据。



下图中5-6行，把rdi和rsi交给rbx和rbp，rbx和rbp又存在栈上的，13-14行再弹出来恢复即可。


![image-20231113192710103]({{ "/assets/procedure/image-20231113192710103.png" | absolute_url }})

### 总结

栈帧压栈的顺序是

先看有没有需要保存的被调用者寄存器，有的话压栈，（Saved registers）

随后看有没有多余的不满足条件的局部变量，有的话压栈，（Local variables，寄存器用完了存stack）

随后看有没有满足条件的局部变量，有的话压栈，（Local variables，指针，数组存stack）

随后看有没有多余的需要构造的参数，有的话压栈，（arguments大于6个存stack）

随后用call来调用函数并保存返回地址，函数返回后继续运行

运行到最后时，若之前有需要保存的被调用者寄存器，则把值从栈中弹回到对应的寄存器。
