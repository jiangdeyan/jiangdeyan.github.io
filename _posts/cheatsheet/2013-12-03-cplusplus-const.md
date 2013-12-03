---
layout: post
title:  关于常量折叠
category: cheatsheet
description: C与C++编译器的不同
---


```c
#include <stdio.h>

int main(int argc, char* argv[])
{
        const int i=0;
        int *j = (int *) &i;
        *j=1;
        printf("%p\n",&i);
        printf("%p\n",j);
        printf("%d\n",i);
        printf("%d\n",*j);
return 0;
}
```
gcc
```
output:
0x7fff136880dc
0x7fff136880dc 
1
1
```

```c++
#include <iostream>
using namespace std;

int main(int argc, char* argv[])
{
        const int i=0;
        int *j = (int *) &i;
        *j=1;
        cout<<&i<<endl;
        cout<<j<<endl;
        cout<<i<<endl;
        cout<<*j<<endl;
        return 0;
}
```
g++
```
output:
0x7fff5c89d53c
0x7fff5c89d53c
0
1
```

C++的这种现象叫做常量折叠，因为i和j都指向相同的内存地址，所以输出的前两个结果是相同的，但为啥相同的内存里的结果不相同么？－－这就是常量折叠。

这个"常量折叠"是 就是在编译器进行语法分析的时候，将常量表达式计算求值，并用求得的值来替换表达式，放入常量表。可以算作一种编译优化。

我只是改了这个地址内容,但是i还是0，因为编译器在优化的过程中，会把碰见的const全部以内容替换掉（跟宏似的: #define pi 3.1415,用到pi时就用3.1415代替），这个出现在预编译阶段；但是在运行阶段，它的内存里存的东西确实改变了!!!
