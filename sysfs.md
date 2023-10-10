# 								Sysfs文件系统

## sysfs简介

`sysfs`是一个伪文件系统。不代表真实的物理设备，在linux内核中，`sysfs`文件系统将长期存在于内存中。`sysfs`用于对具体的内核对象（例如物理设备）进行建模，并提供一种将设备和设备驱动程序关联起来的方法。使用

```
ls -l /sys
```

`sysfs`文件系统中导出了block、bus、class、dev、devices、[firmware](https://so.csdn.net/so/search?q=firmware&spm=1001.2101.3001.7020)、fs、kernel、module、power内核对象（不同的内核配置，可能导出的内核对象有所差异）。

## sysfs的运行机制原理

sysfs 提供一种机制，使得可以显式的描述内核对象、对象属性及对象间关系。

- 一组接口，针对内核，用于将设备映射到文件系统中。
- 一组接口，针对用户程序，用于读取或操作这些设备。描述了内核中的 sysfs 要素及其在用户空间的表现，如下图所示：

![img](https://img-blog.csdnimg.cn/20200801012027747.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzNDg3MDQ0,size_16,color_FFFFFF,t_70)

## sysfs文件系统在linux内核中的挂载

在start_kernel()函数中将调用vfs_caches_init(totalram_pages);函数进行虚拟文件系统的初始化，此函数中将调用mnt_init()函数，在mnt_init()函数中将调用sysfs_init()对sysfs文件系统进行挂载。如下图片：


![在这里插入图片描述](https://img-blog.csdnimg.cn/c829f647414a4c769594649b80790ef7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaXJpY3poYW8=,size_20,color_FFFFFF,t_70,g_se,x_16)

## linux内核将内核对象集添加到`sysfs`文件系统

linux内核中调用了`kset_create_and_add()`内核函数创建内核对象集，从而将内核对象集添加到sysfs文件系统下，本部分来看看该函数。

该函数有以下重要的函数调用关系：

![在这里插入图片描述](https://img-blog.csdnimg.cn/b3d89fd8a1424948920f601b1774db04.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAaXJpY3poYW8=,size_20,color_FFFFFF,t_70,g_se,x_16)

