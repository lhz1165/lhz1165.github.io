---
layout: post
title: "机器指令"
---
CSAPP第二部分

# Machine Level Programming - I Basic

## register

rax    r代表64

eax   e代表32位  

rdi和rsi一般保存地址（指针类型）

特殊寄存器%rsp,用于栈的指针，不能用来存放数据

## Moving Data

moveq [source],[dest]    (by the way  moveq的q代表quad)

source:立即数， 寄存器，内存

dest： 寄存器 内存

(r)-->mem[reg[r]]

d(r)--->mem[reg[r]+d]  

d(rb,ri,s) --->mem[reg[rb]+s*reg[ri]+d] rb代表基地址，ri代表索引，s代表索引长度，一般用于数组

例如leaq 4(%rdi,%rdx) ,%rcx ====>rdi+rdx+4存到rcx中

## 地址计算指令

leaq [src] [dest]   src一个地址，dest寄存器，根据地址的值计算，然后实际结果的地址写入的是该地址，而不是内存的值，即把指针写入寄存器



用来 

1. 相当与c语言中的&  

2. 计算x+k*y，k=1，2，4，8    例如  leal (%eax, %eax, 2) , % eax

**当用法第一种时**

adress : 0x1---> value: 0x88

adress:0x3---> value 0x77

那么 

 leal 2(%eax) , % eax 执行完之后， eax的值是0x3（指针）

movl 2(%eax) , % eax 执行完之后，eax的值是0x77（地址）

**当用法第二种时**

x*12   

 leaq(%rdi,%rdi,2),%rax

sqlq $2, %rax

先把%rdi的值=rdi+2rdi=x*3,结果的地址存入rax

然后再rax的值左移两位

# Machine Level Programming - II-Control

## 条件码

CF carry flag  进位

 ZF  zero flag 0标志

SF  sing flag 负数

OF  补码溢出

用途

cmpq src2 src1，根据两个数之差来设置条件码

## 条件分支

第一种使用有条件jump，判断然后跳转到指定代码行



第二种使用conditional move，两个结果计算出来，然后选择我们需要的

(由于流水线技术，if中的判断，if的结果，else的结果都可以计算出来，性能更好)



# Machine Level Programming - III - Procedures

1. 传递控制（方法调用
2. 传递值  (传参数
3. 内存管理 （分配内存，回收内存

## Stack

用栈来管理procedure的过程，stack越下面地址越大，越往上越小

POP : read  by %rsp， %rsp++,  value to dest

PUSH: fetch  from src,  %rsp--, value to %rsp所指的地方

## 方法调用过程

%rsp指向当前栈里面某处地址，%rip指向程序执行的地址

当发生调用做两件事情，

	1. %rsp--，栈里保存的是方法调用完成后的下一条指令的地址（等方法返回继续从这里执行），
	1. %rip(pc)指向被调用代码的位置

##  1. <a name='register1'></a>寄存器分类

 %rbx,%rbp, %r12–%r15 are classified as **callee-saved registers**

%rdi  %rsi  %rdx  %rcx   %r8 %r9   **caller-saved registers**

![image-20231113114233452]({{"/assets/procedure/image-20231113114233452.png" | absolute_url }})

%rsp  %rbp栈指针：%rbp% 指向栈底， %rsp指向栈顶来，限制栈帧的范围

%rax 保存function 的返回结果

%rip 下一条将要被执行的指令的逻辑地址

##  2. <a name='Stack'></a>Stack

![image-20231114155735683]({{ "/assets/procedure/image-20231114155735683.png" | absolute_url }})
![image-20231114155112689]({{ "/assets/procedure/image-20231114155112689.png" | absolute_url }})


例如p(int.....,int 7x,int 8x,int 9x)->先9x进栈在8x进栈,从后往前



##  3. <a name='ControlTransfer'></a>Control Transfer

call：被调用方法返回后的下一条指令地址（return address）push 到stack，然后pc设置为被调用方法的首地址

ret：pop出return address，然后pc设置为return address

##  4. <a name='DataTransferarguments'></a>Data Transfer（arguments）

![image-20231113135005671]({{ "/assets/procedure/image-20231113135005671.png" | absolute_url }})


假设p调用proc，p栈帧里传给proc的参数，超过6个的，存储在stack里面，并且最后 a4p 先压入栈，再a4，每个参数要对齐，所以下面实际是p的栈帧结构。

![image-20231113135024111]({{ "/assets/procedure/image-20231113135024111.png" | absolute_url }})

##  5. <a name='LocalStorageonStackLocalvariables'></a>Local Storage on Stack（Local variables）

局部变量存在stack的例子，局部变量也是按倒序来存的

![image-20231113145956959]({{ "/assets/procedure/image-20231113135024111.png" | absolute_url }})

结合data transfer和local storage例子

![image-20231113152245467]({{ "/assets/procedure/image-20231113152245467.png" | absolute_url }})


![image-20231113152301378]({{ "/assets/procedure/image-20231113152301378.png" | absolute_url }})

为local variable设置栈帧，参数和加载方法的参数到register



![image-20231113152437223]({{ "/assets/procedure/image-20231113152437223.png" | absolute_url }})

上图表示local variables x1-x4在 stack 的分配，**变量在stack不用对齐**； parameters x1-x4和&x1-&x4 6个分配在register，2个分配在stack,**参数在stack要对齐**

##  6. <a name='LocalStorageinRegistersSavedregisters'></a>Local Storage in Registers（Saved registers部分）

（使用**callee-saved registers**来保存变量，必须在callee()里面保存之前caller()的寄存器数据到stack，再使用，callee()执行完返回后，把stack的变量写回到寄存器，交给caller()继续执行)



The name“caller saved”: can be understood in the context of a procedure P having some local data in such a register and calling procedure Q. Since Q is free to alter this register,it is incumbent upon P(the caller) to first save the data before it makes the call.


![image-20231113192710103]({{ "/assets/procedure/image-20231113192710103.png" | absolute_url }})

## 总结

栈帧压栈的顺序是

先看有没有需要保存的被调用者寄存器，有的话压栈，（callee-register）

随后看有没有多余的不满足条件的局部变量，有的话压栈，（Local variables，寄存器用完了存stack）

随后看有没有满足条件的局部变量，有的话压栈，（Local variables，指针，数组存stack）

随后看有没有多余的需要构造的参数，有的话压栈，（arguments大于6个存stack）

随后用call来调用函数并保存返回地址，函数返回后继续运行

运行到最后时，若之前有需要保存的被调用者寄存器，则把值从栈中弹回到对应的寄存器。
