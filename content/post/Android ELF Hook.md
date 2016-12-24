+++
date = "2016-12-24T09:39:25+08:00"
title = "Android ELF Hook"
categories = ["API Hook Network Hijack"]
tags = [
  "ELF",
  "Hook",
  "Android"
  ]

+++

首先写一段C代码，用arm-Linux-androideabi-gcc 编译成可执行文件:

````C
#include <stdio.h>
#include <string.h>

typedef int (*fn_strlen)(const char *);
fn_strlen g_strlen = (fn_strlen)strlen;
int main(const int argc, const char * args[])
{
    const char * helloworld = "hello world!";
    int a = g_strlen(helloworld);
    int b = strlen(helloworld);
    return 0;
}

````

用arm-linux-androideabi-objdump -D 反汇编出所有的段，然后分析我们感兴趣的几个段:

>.text 的\<main\>  
>.plt 的 \<strlen@plt\>  
>.got  
>.rodata  
>.data  

<div align=center>
    <img src="/images/android-elf-hook/figure1.png" width="80%" alt="figure 1"/>
    <img src="/images/android-elf-hook/figure2.png" width="80%" alt="figure 2"/>
</div>


## 1. strlen的函数重定位：

\<main\> 函数反汇编的地址 0x384 中，指令 bl 294 对应了C代码中的strlen(helloworld);  
bl 294 的机器码是 eb ff ff ff c2, 其中 高8位 eb 是条件跳转指令，低24位是相对于当前PC的偏移地址offset   
offset = (目的地址 - 当前PC) >> 2, 而 PC 等于当前指令的地址 + 8  所以  
offset = (0x294 - (0x384+0x8)) >> 2 = (-0xF8 >> 2) = -(0x3E) 
-(0x3E) 取24位补码等于 0xFFFFC2


在\<strlen@plt\>:地址0x294的位置是三条汇编指令，等价于 

```` 
ip = pc + 0 
ip = ip + 0x1000
pc = [ip + 0xd60]
````
也就是将 (0x294 + 0x8) + 0x1000 + 0xd60 = 0x1FFC 指向的内存区的值取出来 赋给PC

地址0x1FFC, 是在<_GLOABLE_OFFSET_TABLE>中的表项，内容是0x268, <__libc_init@plt-0x14>的地址，初始化成0x268这个直，应该就是为了动态链接函数地址时作lazy load用的。

再来看在图三rel.plt表中offset为0x1FFC的项也说明了strlen函数对应的offset地址。

<div align=center>
    <img src="/images/android-elf-hook/figure3.png" width="80%" alt="figure 3"/>
</div>

## 2. g_strlen全局函数变量的重定位：
对应的汇编代码在  <main>:0x368-0x378， 写成伪代码如下：

````
    0x368: r3 = [0x3a0] = 0x1c8c
    0x36c: r3 = pc + 0x1c8c =  0x36c + 0x08+ 0x1c8c = 0x2000
    0x370: r3 = [r3] = [0x2000]
    0x378: call r3 
````
图三.rel.dyn 表中 offset = 0x2000 对应的sym name 为g_strlen. 
图二中最后一行也可以看到对应0x2000 的值是0， 当linker链接后会置为符号strlen的实际地址.

## 3. \.rel\.dyn 与 \.rel\.plt 的区别
.rel.dyn和.rel.plt是REL/RELA，它们是Elf32_Rel类型或者Elf64_Rela的结构体数据  
.rel.dyn节的每个表项对应了除了外部过程调用的符号以外的所有重定位对象，
.rel.plt节的每个表项对应了所有外部过程调用符号的重定位信息。


**elf hook 的原理实际上就是通过.rel.dyn节和.rel.plt节的rel/rela项，找到符号对应的地址，例如本文中0x1ffc和0x2000, 并将其替换。**  
