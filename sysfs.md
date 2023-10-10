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



## Sys文件系统的初始化过程

![在这里插入图片描述](https://img-blog.csdnimg.cn/c0102a8edcdb47f6a647f5bef12691d5.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5YaF5qC456yU6K6w,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)



## 如何添加Sysfs节点（Kobject 和 kset）

### 1、使用 attribute 例子

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210501134246546.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzNDg3MDQ0,size_16,color_FFFFFF,t_70)

使用 API：

- kobject_init
- kobject_add

```c


```



### 2、使用 device_attribute 例子

使用API:

- ​	kobject_create_and_add
- ​	sysfs_create_group

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/kobject.h>
#include <linux/fs.h>
#include <linux/sysfs.h>
#include <linux/string.h>
#include <linux/slab.h>

static struct kobject *foo_kobj;

struct foo_attr{
	struct kobj_attribute attr;
	int value;
};

static struct foo_attr foo_value;
static struct foo_attr foo_notify;

static struct attribute *foo_attrs[] = {
	&foo_value.attr.attr,
	&foo_notify.attr.attr,
	NULL
};

static struct attribute_group foo_group = {
	.attrs = foo_attrs,
};

static ssize_t foo_show(struct kobject *kobj, struct kobj_attribute *attr, 
		char *buf)
{
	struct foo_attr *foo = container_of(attr, struct foo_attr, attr);
	return scnprintf(buf, PAGE_SIZE, "%d\n", foo->value);
}

static ssize_t foo_store(struct kobject *kobj, struct kobj_attribute *attr, 
		const char *buf, size_t len)
{
	struct foo_attr *foo = container_of(attr, struct foo_attr, attr);

	sscanf(buf, "%d", &foo->value);
	sysfs_notify(foo_kobj, NULL, "foo_notify");
	return len;
}

static struct foo_attr foo_value = {
	.attr = __ATTR(foo_value, 0644,	foo_show, foo_store),
	.value = 0,
};

static struct foo_attr foo_notify = {
	.attr = __ATTR(foo_notify, 0644, foo_show, foo_store),
	.value = 0,
};

static int __init foo_init(void)
{
	int ret = 0;

	printk("%s\n", __func__);

	foo_kobj = kobject_create_and_add("foo", NULL);
	if (!foo_kobj) {
		printk("%s: kobject_create_and_add() failed\n", __func__);
		return -1;
	}

	ret = sysfs_create_group(foo_kobj, &foo_group);
	if (ret) 
		printk("%s: sysfs_create_group() failed. ret=%d\n", __func__, ret);

	return ret;	/* 0=success */
}

static void __exit foo_exit(void)
{
	if (foo_kobj) {
		kobject_put(foo_kobj);
	}

	printk("%s\n", __func__);
}

module_init(foo_init);
module_exit(foo_exit);
MODULE_LICENSE("GPL");

```

### 3、platform_device_register

在注册平台设备时使用 device_attribute 结构注册设备属性

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/kobject.h>
#include <linux/fs.h>
#include <linux/sysfs.h>
#include <linux/string.h>
#include <linux/slab.h>
#include <linux/platform_device.h>                                              

struct foo_attr{
	struct device_attribute attr;
	int value;
};

static struct foo_attr foo_value;
static struct foo_attr foo_notify;

static struct attribute *foo_attrs[] = {
	&foo_value.attr.attr,
	&foo_notify.attr.attr,
	NULL
};

ATTRIBUTE_GROUPS(foo);

static ssize_t foo_show(struct device *dev, struct device_attribute *attr, 
		char *buf)
{
	struct foo_attr *foo = container_of(attr, struct foo_attr, attr);
	return scnprintf(buf, PAGE_SIZE, "%d\n", foo->value);
}

static ssize_t foo_store(struct device *dev, struct device_attribute *attr, 
		const char *buf, size_t len)
{
	struct foo_attr *foo = container_of(attr, struct foo_attr, attr);

	sscanf(buf, "%d", &foo->value);
	sysfs_notify(&dev->kobj, NULL, "foo_notify");
	return len;
}

static struct foo_attr foo_value = {
	.attr = __ATTR(foo_value, 0644, foo_show, foo_store),
	.value = 0,
};

static struct foo_attr foo_notify = {
	.attr = __ATTR(foo_notify, 0644, foo_show, foo_store),
	.value = 0,
};

static struct platform_device foo_device = {                                   
	.name = "foo",
	.id = -1,
	.dev.groups = foo_groups,
};                                                                              

static int __init foo_init(void)
{
	int ret = 0;

	printk("%s\n", __func__);

	ret = platform_device_register(&foo_device);
	if (ret < 0) {
		printk("%s: platform_device_register() failed. ret=%d\n",
				__func__, ret);
		ret = -1;
	}

	return ret;	/* 0=success */
}

static void __exit foo_exit(void)
{
	platform_device_unregister(&foo_device);

	printk("%s\n", __func__);
}

module_init(foo_init);
module_exit(foo_exit);
MODULE_LICENSE("GPL");
```

