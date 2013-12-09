---
layout: post
title:  浅谈C语言的数据存储
category: cheatsheet
description:  C语言数据存储基础知识
---

[浅谈C语言的数据存储（一）][1]  
[浅谈C语言的数据存储（二）][2]  
[C语言中全局变量、局部变量、静态全局变量、静态局部变量的区别][3]  


C程序大致来讲可以分为四个数据区：常量区，静态区，堆区，栈区。  

###常量区    
只包含字符串常量(赋值给数组除外)，const全局变量。是read-only的。  
字符常量，整型，浮点型常量不包含在常量区，这些都以立即数的形式复制。  
const的局部变量不在常量区。    
因此`char * p = "abc"；`是成立的，会把常量区的字符串头指针复制给`p`。  
当字符串赋值给数组时，该字符串常量是存在在栈中的。比如`char q[] = "adc"`，`q`和`"adc"`的都是存在在栈中。 

其他情况下指针必须使用calloc函数、&、数组形式进行初始化。  


###静态区   

全局变量，静态全局变量，局部变量，静态局部变量。  

除了局部变量，其余都存在在静态区。

###栈区   

局部变量，数组字符串

###堆区   


[1]:http://www.embedu.org/Column/Column540.htm
[2]:http://www.embedu.org/Column/Column558.htm
[3]:http://blog.csdn.net/tigerjibo/article/details/7425580
