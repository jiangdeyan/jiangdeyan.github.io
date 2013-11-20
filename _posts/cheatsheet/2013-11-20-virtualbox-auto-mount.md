---
layout:     post
title:      VirtualBox实现自动挂载
category: cheatsheet
description:
---

VirtualBox在主机Window虚拟机为Linux时，本身支持把共享空间自动挂载，但是无法指定挂载目录。

通过修改Linux的/etc/fstab文件，可以实现自动加载。该文件存放的是系统中的文件系统信息。

加载格式为  
`<file system>		<mount point>		<type>	<options>	<dump>	<pass>`  
myCode			/home/username/myCode	vboxsf	defaults	0	0 

这样就实现了自动挂载了。
