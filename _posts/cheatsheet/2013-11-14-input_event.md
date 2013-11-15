---
layout:     post
title:      Linux（Android） input_event结构体
category: cheatsheet
description: 
---

Linux input_event struct定义在Linux/include/uapi/linux/input.h 
 
```c
    struct input_event {
         struct timeval time; 
         __u16 type;
         __u16 code;
         __s32 value;
    };
```
在Android中，标志性的type中有EV_KEY，EV_REL，EV_ABS。  

各种KEY事件是EV_KEY。Sensor的事件是EV_REL。Touch事件是EV_ABS。  
EV_KEY中CODE表示何种KEY，value=0表示抬起，value=1表示按下。  
EV_ABS中CODE表示哪个轴，value表示数值。  

在Android中可以使用getevent获取input_event事件的值，其输出形式为：
 
```c
printf("%04x %04x %08x", event.type, event.code, event.value);
```
参考[Android 中input event的分析][1]  

getevent -l 可以将type和code转换成对应的信息。  



[1]:http://blog.csdn.net/learnrose/article/details/6236890
