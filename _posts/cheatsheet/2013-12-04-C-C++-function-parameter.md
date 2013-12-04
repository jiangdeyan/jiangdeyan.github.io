---
layout: post
title: C/C++ 函数传参
category: cheatsheet
description: 举例探究形参和实参的联系
---

C/C++的形参一般有三种形式——指针，值，引用，在C中没有引用的概念。
网络上一般的将形参传递分为两种，传值和传址。其实我认为是不准确的。  

严格来说C的函数形参传递只有一种，就是传递**传入变量**的拷贝，即传入参数和函数形参的地址不同。例子：

```C
#include <stdio.h>


int changeto1(int i)
{
        printf("changeto1 i's address is %p\n", &i);
        i = 1;
        return 0;
}

int changeto2(int *i)
{
        printf("changeto2 i's address is %p\n", &i);
        *i = 2;
        return 0;
}
int main(int argc, char** argv)
{
        int t = 0;
        printf("before changeto1 t's address is %p\n",&t);
        changeto1(t);
        int* p = &t;
        printf("before changeto2 p's address is %p\n",&p);
        changeto2(p);
        return 0;
}
```

output:

```C
before changeto1 t's address is 0x7fff75690b8c
changeto1 i's address is 0x7fff75690b5c
before changeto2 p's address is 0x7fff75690b80
changeto2 i's address is 0x7fff75690b58
```

由此得出，C函数的函数传递，只是简单的把传入参数的内容拷贝一下，即形参跟实参是两个地址。

C++有个特殊的形参类型是引用，如果形参是引用的话，那么形参和实参实际是同一个地址。例子：

```C
#include <iostream>
using namespace std;
int changeto2(int &i)
{
        cout<<"changeto2 i's address is "<<&i<<endl;
        i = 2;
        return 0;
}
int main(int argc, char** argv)
{
        int t = 0;
        cout<<"before changeto2 i's address is "<<&t<<endl;
        changeto2(t);
        return 0;
}
```

output:

```C
before changeto2 i's address is 0x7ffffbd779ec
changeto2 i's address is 0x7ffffbd779ec
```

形参和实参的地址是一致的。
