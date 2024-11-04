# 字符设备驱动开发基础实验-Linux驱动开发实验流程

字符设备就是一个一个字节，按照字节流进行读写操作的设备，读写数据是分先后顺序的。比如我们最常见的点灯、按键、IIC、SPI，LCD 等等都是字符设备，这些设备的驱动就叫做字符设备驱动。

<img src=.\fig\1728016201055.png alt=1728016201055 width= "600" height="">

驱动加载成功后会在/dev/目录下升成一个相应的文件，应用程序通过对名为/dev/xxx(xxx为驱动名。)进行相应的操作即可实现对硬件的操作。

应用程序运行在用户空间，驱动属于内核的一部分，运行在内核空间。通过“系统调用”的方法实现从用户空间“陷入”到内核空间。open、close、write 和 read 等这些函数是由 C 库提供的。 

<img src=.\fig\1728016946727.png alt=111 >

比如应用程序中调用了 open 这个函数，那么在驱动程序中也得有一个名为 open 的函数。每一个系统调用，在驱动中都有与之对应的一个驱动函数，在 Linux 内核文件 include/linux/fs.h 中有个叫做 file_operations 的结构体，此结构体就是 Linux 内核驱动操作函数集合。

## 字符设备驱动的开发步骤



- 字符设备驱动框架

  字符设备驱动的编写主要就是驱动对应的open，close，read，write等，其实就是file_operation结构体成员变量的实现

- Linux驱动模块的加载与卸载

  驱动在编译的时候可以选择直接编译到内核里面去，另外一种是不编译到内核中，而是编译成模块.ko文件。我们在调试的时候选择将其编译为模块。测试的时候只需要加载.ko模块就可以了。

  模块有加载和卸载两种操作，我们在编写驱动的时候需要**注册**这两种操作函数，模块的加载和卸载的注册函数如下：

  ```c
  module_init(xxx_init); //注册模块加载函数
  module_exit(xxx_exit); //注册模块卸载函数
  ```

  当使用`insmod`命令加载驱动的时候，`xxx_init` 这个函数就会被调用。`rmmod`命令卸载具体驱动的时候 `xxx_exit`函数就会被调用。
  
  将编译出来的.ko文件放到根文件系统里面。加载驱动会用到加载命令: `insmod`，`modprobe`。移除驱动使用`rmmod`。`insmod`不能解决模块的依赖关系。`modprobe`会分析模块的依赖关系
  
  modprobe 命令主要智能在提供了模块的依赖性分析、错误检查、错误报告等功能，推荐使用 modprobe 命令来加载驱动。modprobe 命令默认会去/lib/modules/<kernel-version>目录中查找模块，比如本书使用的 Linux kernel 的版本号为 4.1.15，因此 modprobe 命令默认会到/lib/modules/4.1.15 这个目录中查找相应的驱动模块，一般自己制作的根文件系统中是不会有这个目录的，所以需要自己手动创建。 
  
  - 将生成的.ko文件拷贝到4.1.15这个目录里面去
  
  - 输入depmod命令。对于一个新的模块使用modprobe加载的时候需要调用一下depmod命令。
  
  - 加载模块
  
  - 驱动模块加载成功以后可以使用`lsmod`查看，`cat /proc/devices`可以查看当前使用的主设备号
  
  - 创建设备节点（后面的教程中可以自动创建节点，本讲我们手动创建）
  
    ```sh
    mknod /dev/chrdevbase c 200 0	# c表示的是字符设备，主设备号是200，次设备号是0
    ```
  
  - ```sh
    ls /dev/ # 查看当前设备节点
    ```
  
  - 运行应用程序
  
    ```sh
    /lib/modules/4.1.15 # ./chrdevbaseAPP /dev/chrdevbase
    chrdevbase_open
    chrdevbase_read
    chrdevbase_write
    chrdevbase_release
    /lib/modules/4.1.15 #
    ```
  
  - 卸载模块使用`rmmod`命令
  
- 驱动代码框架

  ```c
  #include <linux/module.h>
  #include <linux/kernel.h>
  #include <linux/init.h>
  
  /* 加载函数 */
  static int __init chrdevbase_init(void)
  {
  	printk("chrdevbase_init\r\n");
  	return 0;
  }
  
  /* 卸载函数 */
  static void __exit chrdevbase_exit(void)
  {
  	printk("chrdevbase_exit\r\n");
  }
  
  /**
   * 模块入口与出口
   */
  module_init(chrdevbase_init);  /* 注册加载函数 */
  module_exit(chrdevbase_exit);  /* 注册卸载函数 */ 
  
  MODULE_LICENSE("GPL");
  MODULE_AUTHOR("syw");
  ```

- Makefile代码

  ```makefile
  KERNELDIR := /home/syw/linux/IMX6ULL/linux/linux-imx-rel_imx_4.1.15_2.1.0_ga
  CURRENT_PATH := $(shell pwd)
  
  obj-m := chrdevbase.o
  
  build: kernel_module
  
  kernel_module:
      $(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) modules
  
  clean:
      $(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) clean
  ```

- 内核中的显示函数是`printk`，所以在写驱动的时候进行程序调试需要用到该函数
- 用户空间的显示函数是`printf`，在应用程序开发中使用的是printf。

## 注册字符设备和注销字符设备

向系统注册一个字符设备使用函数`register_chrdev`

```c
int register_chrdev(unsigned int major, const char *name,
				  const struct file_operations *fops)
```

注销字符设备`unregister_chrdev`

```c
void unregister_chrdev(unsigned int major, const char *name)
```

其中major为主设备号(Linux中查看设备号的命令`cat /proc/devices`)

name为设备名

定义fops结构体

实现fops结构体中定义的内核端的函数。

### 设备号

设备号分为主设备号和次设备号设备号由`dev_t`的设备类型来表示。（u32）

设备号有32位，其中高12位为主设备号，低20位为低设备号，因此Linux主设备号的范围为0-4095

include/linux/kdev_t.h 中提供了几个关于设备号的操作函数(本质是宏)

```c
#define MINORBITS 20										// 次设备号位数
#define MINORMASK ((1U << MINORBITS) - 1)					// 次设备号掩码

#define MAJOR(dev) ((unsigned int) ((dev) >> MINORBITS))	// 用于从 dev_t 中获取主设备号
#define MINOR(dev) ((unsigned int) ((dev) & MINORMASK))		// 用于从 dev_t 中获取次设备号
#define MKDEV(ma,mi) (((ma) << MINORBITS) | (mi))			// 将给定的主设备号和次设备号的值组合成 dev_t 类型的设备号。
```



## 应用程序开发

```c
fd = open(filename, O_RDWR);
```

打开文件之后后续再对文件进行操作只需要对fd操作即可。`O_RDWR`表示对文件进行读写操作。

应用程序不能直接访问内核数据，必须借助其他函数！

```c
/* 参数 to 表示目的，参数 from 表示源，参数 n 表示要复制的数据长度。如果复制成功，返回值为 0，如果复制失败则返回负数。 */
copy_to_user();
static inline long copy_to_user(void __user *to, const void *from, unsigned long n)
```



源程序: 应用程序可以对驱动读写操作，读的话就是从驱动里面读取字符串，写的话就是向驱动里面写字符串。

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>

/**
 * @brief Linux应用程序编写
 * ./chrdevbase <filename> <1/2> 1表示读，2表示写
 * 
 * @param argc : 输入的参数的个数，包括命令本身
 * @param argv : 输入的参数，指针数组，每个元素都是一个字符串
 * @return int 
 */
int main(int argc, char *argv[])
{
    int ret = 0;
    int fd =0;
    char *filename;
    char readbuf[100], writebuf[100];
    static char usrdata[] = "usr data!";

    if (argc != 3)
    {
        printf("Error usage!\r\n");
    }
    

    filename = argv[1];

    fd = open(filename, O_RDWR);        // 如果文件打开错误将会返回-1
    if (fd < 0)
    {
        printf("Can't open file %s\r\n", filename);
        return -1;
    }

    /* read */
    if (atoi(argv[2]) == 1)
    {
        ret = read(fd, readbuf, 50);
        if (ret < 0) {
            printf("read file %s failed!\r\n", filename);
        } else {
            printf("APP read data: %s\r\n", readbuf);
        }
    }
    
    
    /* write */
    if (atoi(argv[2]) == 2)
    {
        memcpy(writebuf, usrdata, sizeof(usrdata));
        ret = write(fd, writebuf, 50);
        if (ret < 0)
        {
            printf("write file %s failed!\r\n", filename);
        } else {
            
        }
    }
    
    /* close */
    ret = close(fd);
    if (ret < 0)
    {
        // printf("close file %s failed!\r\n", filename);
    } else {

    }
}
```

```c
/lib/modules/4.1.15 # ./chrdevbaseAPP /dev/chrdevbase 1
chrdevbase_open
chrdevbase_read
read success!
chrdevbase_releasenel data!
// 最后一句是用户态和内核态的输出冲突了
```



# 嵌入式LED驱动开发实验

## 地址映射

- MMU内存管理单元

  完成虚拟空间到物理空间的映射

  内存保护设置存储器的访问权限，设置虚拟存储空间的缓冲特性。

  Linux内核启动的时候会初始化MMU，设置好内存映射，设置好以后CPU访问的都是虚拟地址。

- ioremap, iounmap物理地址和虚拟地址之间的转换

  - ioremap函数

    用于获取指定物理地址空间对应的虚拟地址空间。定义在文件夹arch/arm/include/asm/io.h文件中。

    ```c
    1 #define ioremap(cookie,size) __arm_ioremap((cookie), (size), 
    MT_DEVICE)
    2
    3 void __iomem * __arm_ioremap(phys_addr_t phys_addr, size_t size, 
    unsigned int mtype)
    4 {
    5 return arch_ioremap_caller(phys_addr, size, mtype,
    __builtin_return_address(0));
    6 }
    ```

    ioremap 是个宏，有两个参数：cookie 和 size，真正起作用的是函数__arm_ioremap，此函数有三个参数和一个返回值，这些参数和返回值的含义如下： 

    **phys_addr**：要映射的物理起始地址。 

    **size**：要映射的内存空间大小。 

    **mtype**：ioremap 的类型，可以选择 MT_DEVICE、MT_DEVICE_NONSHARED、 

    MT_DEVICE_CACHED 和 MT_DEVICE_WC，ioremap 函数选择 MT_DEVICE。 

    **返回值：**__iomem 类型的指针，指向映射后的虚拟空间首地址。

  - iounmap函数

    卸载驱动的时候需要使用 iounmap 函数释放掉 ioremap 函数所做的映射

    ```c
    void iounmap (volatile void __iomem *addr)
    ```

    iounmap 只有一个参数 addr，此参数就是要取消映射的虚拟地址空间首地址。

    

- 驱动程序编写

  - 定义相关物理地址
  
  - 定义地址映射
  
  - 利用函数进行地址映射（在入口函数进行，在进行了地址映射之后对相关寄存器进行初始化配置）
  
    ```c
    /* 例如 */
    #define SW_MUX_GPIO1_IO03_BASE (0X020E0068)
    static void __iomem* SW_MUX_GPIO1_IO03;
    SW_MUX_GPIO1_IO03 = ioremap(SW_MUX_GPIO1_IO03_BASE, 4);
    ```
    
    利用IO内存访问函数进行内存的读写操作
    
    ```c
    /* 读操作函数，分别对应8bit, 16bit, 32bit */
    1 u8 readb(const volatile void __iomem *addr)
    2 u16 readw(const volatile void __iomem *addr)
    3 u32 readl(const volatile void __iomem *addr)
    /* 写操作函数，分别对应8bit, 16bit, 32bit */
    1 void writeb(u8 value, volatile void __iomem *addr)
    2 void writew(u16 value, volatile void __iomem *addr)
    3 void writel(u32 value, volatile void __iomem *addr)
    ```
    
    在内核编程中，`__iomem`通常与指针一起使用，用于指示该指针指向的是一个 I/O 内存区域，而不是常规的内存区域。这种限定符的使用有助于提高代码的可读性和安全性，明确指针所指向的内存的特定用途。
  
  - 在驱动的write函数中处理开关灯的操作。
  - 在出口函数中取消地址映射，注销字符设备。
  
  驱动函数如下：
  
  ```c
  #include <linux/types.h>
  #include <linux/delay.h>
  #include <linux/ide.h>
  #include <linux/module.h>
  #include <linux/kernel.h>
  #include <linux/init.h>
  #include <linux/errno.h>
  #include <linux/gpio.h>
  #include <linux/fs.h>
  #include <linux/slab.h>
  #include <asm/mach/map.h>
  #include <linux/uaccess.h>
  #include <linux/io.h>
  
  /* 宏定义 */
  #define LED_MAJOR 	200		// 设备号
  #define LED_NAME 	"led"	// 设备名字
  #define LED_ON		1
  #define LED_OFF		0
  
  // LED相关的物理地址
  #define CCM_CCGR1_BASE				(0x020C406C)
  #define SW_MUX_GPIO1_IO03_BASE		(0X020E0068)
  #define SW_PAD_GPIO1_IO03_BASE		(0X020E02F4)
  #define GPIO1_DR_BASE				(0X0209C000)
  #define GPIO1_GDIR_BASE				(0X0209C004)
  
  // 物理地址的虚拟映射
  static void __iomem *IMX6U_CCM_CCGR1;
  static void __iomem *SW_MUX_GPIO1_IO03;
  static void __iomem *SW_PAD_GPIO1_IO03;
  static void __iomem *GPIO1_DR;
  static void __iomem *GPIO1_GDIR;
  
  /**
   * @brief 
   * 
   * @param sta ： LED_ON, LED_OFF
   */
  static void led_switch(u8 sta) {
  	u32 val = 0;
  	if (sta == LED_ON) {
  		val = readl(GPIO1_DR);
  		val &= ~(1 << 3);
  		writel(val, GPIO1_DR);
  	} else if (sta == LED_OFF)
  	{
  		val = readl(GPIO1_DR);
  		val |= (1 << 3);
  		writel(val, GPIO1_DR);
  	}
  }
  
  /* fop结构体函数实现 */
  static int led_open(struct inode *inode, struct file *filp) {
  	return 0;
  }
  static int led_release(struct inode *inode, struct file *filp) {
  	return 0;
  }
  static ssize_t led_write(struct file *filp, const char __user *buf, size_t count, loff_t *ppos) {
  	u64 retvalue;
  	u8 databuf[1];
  	u8 ledstat;
  
  	retvalue = copy_from_user(databuf, buf, count);
  	if (retvalue < 0) {
  		printk("kernel write failed!\r\n");
  		return -EFAULT;
  	}
  	
  	ledstat = databuf[0];
  	led_switch(ledstat);
  
  	return 0;
  }
  
  /* fop结构体定义-字符设备操作集 */
  static const struct file_operations led_fops = {
  	.owner = THIS_MODULE,
  	.open = led_open,
  	.release = led_release,
  	.write = led_write,
  };
  
  /* 入口函数 */
  static int __init led_init(void) {
  	int ret = 0;
  	int val = 0;
  
  	/* 初始化LED */
  	/* 1. 寄存器映射 */
  	IMX6U_CCM_CCGR1		= ioremap(CCM_CCGR1_BASE, 4);
  	SW_MUX_GPIO1_IO03	= ioremap(SW_MUX_GPIO1_IO03_BASE, 4);
  	SW_PAD_GPIO1_IO03	= ioremap(SW_PAD_GPIO1_IO03_BASE, 4);
  	GPIO1_DR			= ioremap(GPIO1_DR_BASE, 4);
  	GPIO1_GDIR			= ioremap(GPIO1_GDIR_BASE, 4);
  
  	/* 2. 使能时钟 */
  	val = readl(IMX6U_CCM_CCGR1);
  	val &= ~(3 << 26);
  	val |= (3 << 26);
  	writel(val, IMX6U_CCM_CCGR1);
  
  	/* 3. 设置复用 */
  	writel(5, SW_MUX_GPIO1_IO03);
  
  	/* 4. 设置io属性 */
  	writel(0x10B0, SW_PAD_GPIO1_IO03);
  
  	/* 5. 设置为输出 */
  	val = readl(GPIO1_GDIR);
  	val &= ~(1 << 3);
  	val |= (1 << 3);
  	writel(val, GPIO1_GDIR);
  
  	/* 6. 默认关闭LED */
  	val = readl(GPIO1_DR);
  	val |= (1 << 3);
  	writel(val, GPIO1_DR);
  
  	/* 注册字符设备 */
  	ret = register_chrdev(LED_MAJOR, LED_NAME, &led_fops);
  	if (ret < 0) {
  		printk("Register chrdev failed!\r\n");
  		return -EIO;
  	}
  	
  
  	printk("Led init!\r\n");
  	return 0;
  }
  
  /* 出口函数 */
  static void __exit led_exit(void) {
  	/* 注销字符设备 */
  	unregister_chrdev(LED_MAJOR, LED_NAME);
  
  	printk("Led exit!\r\n");
  }
  
  /* 注册模块的入口和出口函数 */
  module_init(led_init);
  module_exit(led_exit);
  
  /* 自由软件许可证，作者 */
  MODULE_LICENSE("GPL");
  MODULE_AUTHOR("syw");
  ```
  
  应用程序测试如下：
  
  ```c
  #include <sys/types.h>
  #include <sys/stat.h>
  #include <fcntl.h>
  #include <stdio.h>
  #include <unistd.h>
  #include <stdlib.h>
  #include <string.h>
  
  int main(int argc, char *argv[])
  {
      int fd, retvalue;
      char *filename;
      unsigned char databuf[1];
  
      if (argc != 3) {
          printf("Error Usage!\r\n");
          return -1;
      }
      
  
      filename = argv[1];
  
      /* 打开文件 */
      fd = open(filename, O_RDWR);
      if (fd < 0) {
          printf("File open failed!\r\n");
          close(fd);
          return -1;
      }
      
      /* 向驱动写数据，控制LED灯的亮灭 */
      databuf[0] = atoi(argv[2]);
      retvalue = write(fd, databuf, sizeof(databuf));
      if(retvalue < 0) {
          printf("LED control failed!\r\n");
          return -1;
      }
      
      /* 关闭文件 */
      retvalue = close(fd);
      if(retvalue < 0) {
          printf("File %s close failed!\r\n", argv[1]);
          return -1;
      }
      return 0;
  }
  ```

# 新的字符设备驱动实验

- 使用register_chrdev注册字符设备浪费了很多次设备号。而且需要我们手动指定主设备号。

- 换用自动注册设备号函数，不用手动指定设备号

  ```c
  // 使用以下函数注册设备号
  // baseminor是次设备号的起始数, count是要申请的次设备号的个数，name是申请的字符设备名称
  int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name);	/* 没有指定设备号 */
  int register_chrdev_region(dev_t from, unsigned count, const char *name); // 已指定
  
  // 释放设备号
  void unregister_chrdev_region(dev_t from, unsigned count);
  ```

- 新的字符设备注册方法`cdev`结构体

  - 编写字符设备驱动之前需要定义一个cdev结构体变量，这个变量就表示一个字符设备

  ```c
  // 结构体定义路径: include/linux/cdev.h
  struct cdev {
  	struct kobject 					kobj;
  	struct module 					*owner;
  	const struct file_operations 	*ops;
  	struct list_head 				list;
  	dev_t 							dev;
  	unsigned int 					count;
  };
  ```

  - 定义完cdev结构体变量后使用`cdev_init`函数进行初始化

    ```c
    void cdev_init(struct cdev *cdev, const struct file_operations *fops);
    ```

  - 初始化完结构体变量后需要添加字符设备。使用`cdev_add`函数

    ```c
    // cdev结构体变量；设备号；设备数量
    int cdev_add(struct cdev *p, dev_t dev, unsigned count);
    ```

  - 删除字符设备,使用函数`cdev_del`

    ```c
    void cdev_del(struct cdev *p);
    ```

- 自动创建设备节点`mdev`

  - `mdev`是`udev`的简化版本。udev 可以检测系统中硬件设备状态，可以根据系统中硬件设备状态来创建或者删除设备文件。比如使用modprobe 命令成功加载驱动模块以后就自动在/dev 目录下创建对应的设备节点文件,使用rmmod 命令卸载驱动模块以后就删除掉/dev 目录下的设备节点文件。使用 busybox 构建根文件系统的时候， busybox 会创建一个 udev 的简化版本—mdev 。

    Linux系统中的热插拔事件由`mdev`管理，在`/etc/init.d/rcS`文件中添加如下语句

    ```sh
    echo /sbin/mdev>/proc/sys/kernel/hotplug
    ```

    将 `/sbin/mdev` 写入 `/proc/sys/kernel/hotplug` 文件的作用是告诉内核，在设备热插拔事件发生时，应该调用 `/sbin/mdev` 这个程序来处理这些事件。

  - 自动创建设备节点的工作是在驱动程序的入口函数中完成的，一般在cdev_add函数后面添加相关代码。

    创建`class`类，使用累创建函数

    ```c
    struct class = class_create(owner, name);
    ```
    卸载驱动的时候需要删除类

    ```c
    void class_destroy(struct class *cls)
    ```

  - 创建好类之后还需要创建设备，使用device_create函数

    ```c
    struct device *device_create(struct class 	*class,
    							 struct device 	*parent,
    							 dev_t 			devt,
    							 void 			*drvdata,
    							 const char 	*fmt, ...)
    ```

    这是一个可变参数函数参数 class 就是设备要创建哪个类下面；参数 parent 是父设备，一般为 NULL，也就是没有父设备；参数 devt 是设备号；参数 drvdata 是设备可能会使用的一些数据，一般为 NULL；参数 fmt 是设备名字，如果设置 fmt=xxx 的话，就会生成/dev/xxx这个设备文件。返回值就是创建好的设备。 

    卸载驱动时需要删除创建的设备。使用函数`device_destroy`

    ```c
    void device_destroy(cls, dev);
    ```

    `class` 是一个抽象的概念，用于表示一组具有相似功能的设备。在我们的例子中，所有的LED设备可以归类为 `led` 类。

    `device` 是Linux设备模型中的一个实体，表示系统中的一个硬件设备。在我们的例子中，每个LED设备都是一个 `device`。

- 设置文件私有数据

  - 将所有的设备属性都定义为一个结构体

    ```c
    /* open 函数 */
    static int test_open(struct inode *inode, struct file *filp)
    {
    	filp->private_data = &testdev; /* 设置私有数据 */
    	return 0;
    }
    ```

    在open函数中设置好私有数据之后，在write，read，close函数中直接读取private_data即可得到设备结构体。

    ```c
    static ssize_t my_read(struct file *file, char __user *buf, size_t count, loff_t *ppos)
    {
        my_data_t *data = (my_data_t *)file->private_data;
    
        // 使用 data 中的数据进行读操作
        // ...
    
        return count;
    }
    ```

    

# 设备树



什么是设备树？将板子信息做成独立的格式，文件扩展名为`.dts`.

dts相当于c源码文件。DTC相当于gcc编译器，将dts编译成dtb

一般.dtsi 文件用于描述 SOC 的内部外设信息，比如 CPU 架构、主频、外设寄存器地址范围，比如 UART、 IIC 等等。 

```sh
make imx6ull-alientek-emmc.dtb
# 编译所有的dtb文件
make dtbs
```

在编译Linux内核的时候arch/arm/boot/dts/Makefile文件中可以指定编译哪些dts文件，如果修改了自己的开发板设备树文件，需要在Makefile文件中加入自己的设备树文件。



## dts基本语法

- dts运行类似C语言的头文件调用。

- 有节点和子节点，固定节点如cpu外设信息在一般子文件中使用.h文件调用

- 使用标签追加子节点信息

  ```c
  / {	/* 根节点，两个文件“/”根节点的内容会合并成一个根节点。 */
  	label: node-name@unit-address {
  		a = "xxx";				/* 属性信息 字符串 */
  		reg = <0>;				/* 属性信息 0*/
  		reg = <0 0x123456 100>;	/* 属性信息 一组值*/
  		compatible = "fsl,imx6ull-gpmi-nand", "fsl, imx6ul-gpmi-nand";
  								/* 属性信息 字符串列表*/
  
  		label: mode-name@unit-address {
  			
  		}	/* 子节点 */
  	};
  }
  
  /* 追加节点信息，一般不会修改固定的dtsi文件内容，我们添加子节点总线如IIC上的设备信息时需要在自己板子的设备树中间中追加设备 */
  &label: {
  
  }
  ```

- 标准属性

  - compatible 属性也叫做“兼容性”属性，这是非常重要的一个属性！ compatible 属性的值是一个字符串列表， compatible 属性用于将设备和驱动绑定起来。字符串列表用于选择设备所要使用的驱动程序， compatible 属性的值格式如下所示： 

    ```
    "manufacturer,model"
    ```

    其中 manufacturer 表示厂商， model 一般是模块对应的驱动名字。比如 imx6ull-alientekemmc.dts 中 sound 节点是 I.MX6U-ALPHA 开发板的音频设备节点， I.MX6U-ALPHA 开发板上的音频芯片采用的欧胜(WOLFSON)出品的 WM8960， sound 节点的 compatible 属性值如下：

    ```
    compatible = "fsl,imx6ul-evk-wm8960","fsl,imx-audio-wm8960";
    ```

    属性值有两个，分别为“fsl,imx6ul-evk-wm8960”和“fsl,imx-audio-wm8960”，其中“fsl”表示厂商是飞思卡尔，“imx6ul-evk-wm8960”和“imx-audio-wm8960”表示驱动模块名字。 sound这个设备首先使用第一个兼容值在 Linux 内核里面查找，看看能不能找到与之匹配的驱动文件，如果没有找到的话就使用第二个兼容值查。一般驱动程序文件都会有一个 OF 匹配表，此 OF 匹配表保存着一些 compatible 值，如果设备节点的 compatible 属性值和 OF 匹配表中的任何一个值相等，那么就表示设备可以使用这个驱动。

    OF 匹配表示例代码

    ```c
    static const struct of_device_id imx_wm8960_dt_ids[] = {
    	{ .compatible = "fsl,imx-audio-wm8960", },
    	{ /* sentinel */ }
    };
    MODULE_DEVICE_TABLE(of, imx_wm8960_dt_ids);
    
    static struct platform_driver imx_wm8960_driver = {
    	.driver = {
    		.name = "imx-wm8960",
    		.pm = &snd_soc_pm_ops,
    		.of_match_table = imx_wm8960_dt_ids,
    	},
    	.probe = imx_wm8960_probe,
    	.remove = imx_wm8960_remove,
    };
    ```

  - 根节点的 compatible 属性可以知道我们所使用的设备，一般第一个值描述了所使用的硬件设备名字，比如这里使用的是“imx6ull-14x14-evk”这个设备，第二个值描述了设备所使用的 SOC，比如这里使用的是“imx6ull”这颗 SOC。 Linux 内核会通过根节点的 compoatible 属性查看是否支持此设备，如果支持的话设备就会启动 Linux 内核。 具体流程见文档

  - model 属性值也是一个字符串，一般 model 属性描述设备模块信息，比如名字什么的，比如 

    ```
    model = "wm8960-audio";
    ```

  - status 属性看名字就知道是和设备状态有关的， status 属性值也是字符串，字符串是设备的状态信息 

    |   “okay”   | 表明设备是可操作的。                                         |
    | :--------: | ------------------------------------------------------------ |
    | “disabled” | 表明设备当前是不可操作的，但是在未来可以变为可操作的，比如热插拔设备插入以后。至于 disabled 的具体含义还要看设备的绑定文档。 |
    |   “fail”   | 表明设备不可操作，设备检测到了一系列的错误，而且设备也不大可能变得可操作。 |
    | “fail-sss” | 含义和“fail”相同，后面的 sss 部分是检测到的错误内容。        |

  - #address-cells 和#size-cells 属性 

    #address-cells:子节点 reg 属性中地址信息所占用的字长(32 位)， #size-cells 子节点 reg 属性中长度信息所占的字长(32 位) 一般 reg 属性都是和地址有关的内容，和地址相关的信息有两种：起始地址和地址长度 

    ```
    reg = <address1 length1 address2 length2 address3 length3……>
    ```

    每个“address length”组合表示一个地址范围，其中 address 是起始地址， length 是地址长度 

    第 8 行，子节点 gpio_spi: gpio_spi@0 的 reg 属性值为 <0>，因为父节点设置了#addresscells = <1>， #size-cells = <0>，因此 addres=0，没有 length 的值，相当于设置了起始地址，而没有设置地址长度。 

  - ranges属性

    ```
    ranges = <child-bus-address parent-bus-address size>;
    ```

    ```c
    soc {
        #address-cells = <1>;
        #size-cells = <1>;
        ranges;
    
        bus@10000000 {
            #address-cells = <2>;
            #size-cells = <1>;
            ranges = <0x0 0x10000000 0x100000>;
    
            device@0 {
                reg = <0x0 0x0 0x1000>;
            };
        };
    };
    /* soc 是系统级芯片的节点，它的 #address-cells = <1>，表示它的地址空间使用单个 32 位值来表示地址。#size-cells = <1>，表示它的地址空间的大小也是一个 32 位值。ranges;，空的 ranges 表示这个节点的地址空间和父级（比如内存、系统地址空间）是直接对应的，没有地址映射。
    
    bus@10000000 是一个总线节点，它的 #address-cells = <2>，表示它的地址空间使用两个 32 位值来表示地址（64 位地址）。ranges = <0x0 0x10000000 0x100000>; 表示这个总线节点的地址空间映射如下：
    0x0: 子节点的起始地址（子总线的地址空间从 0x0 开始）。
    0x10000000: 父节点（soc）的地址空间起始地址，这表示该总线的地址从 soc 的 0x10000000 开始。
    0x100000: 表示映射的大小为 1 MB。
    
    device@0 是连接在总线 bus@10000000 上的一个设备，它的 reg 属性定义为 <0x0 0x0 0x1000>：
    0x0 0x0: 这是设备在子总线上的地址（根据总线的 #address-cells = <2> 定义的 64 位地址）。
    0x1000: 表示设备的地址空间大小为 4 KB。
    
    */
    ```

    

## 创建小型设备树文件

- /
  - cpus
    - cpu0: cpu@0
  - soc
    - ocram
    - aips1
      - ecspi1
    - aips2
      - usbotg1
    - aips3
      - rngb



## 设备树在系统中的体现

- 系统启动以后可以在根文件系统里面看到设备树信息。在/proc/device-tree/目录下存放着设备树信息
- 内核启动的时候会解析设备树，然后在/proc/device-tree/目录下呈现出来。

- 可以看出来哪些是子节点哪些是属性

## 特殊节点

- aliases子节点imx6ull.dtsi 文件中。功能是定义别名

  ```
  aliases {
  	can0 = &flexcan1;
  	can1 = &flexcan2;
  	ethernet0 = &fec1;
  	ethernet1 = &fec2;
  	gpio0 = &gpio1;
  	gpio1 = &gpio2;
  	......
  	spi0 = &ecspi1;
  	spi1 = &ecspi2;
  	spi2 = &ecspi3;
  	spi3 = &ecspi4;
  	usbphy0 = &usbphy1;
  	usbphy1 = &usbphy2;
  };	
  ```

- 类似spi1的别名在Linux识别设备树后会转换成，节点-标号的形式如`spi-1`

- chosen子节点

  主要是为了uboot向Linux内核传递数据，重点是bootargs参数

  一般.dts 文件中 chosen 节点通常为空或者内容很少， imx6ull-alientekemmc.dts 中 chosen 节点内容如下所示： 

  ```
  chosen {
  	stdout-path = &uart1;
  };
  ```

  是当我们进入到/proc/device-tree/chosen 目录里面，会发现多了 bootargs 这个属性。这个bootargs是由uboot传递过来的。bootargs 会作为 Linux 内核的命令行参数， Linux 内核启动的时候会打印出命令行参数(也就是 uboot 传递进来的 bootargs 的值) 

  第 288 行，调用函数 fdt_find_or_add_subnode 从设备树(.dtb)中找到 chosen 节点，如果没有找到的话就会自己创建一个 chosen 节点。第 292 行，读取 uboot 中 bootargs 环境变量的内容。第 294 行，调用函数 fdt_setprop 向 chosen 节点添加 bootargs 属性，并且 bootargs 属性的值就是环境变量 bootargs 的内容。

  一切事情的源头都源于如下命令： 

  bootz 80800000 – 83000000 

## Linux内核解析dtb文件

Linux 内核在启动的时候会解析 DTB 文件，然后在/proc/device-tree 目录下生成相应的设备树节点文件。 

通过内核启动时调用的一系列函数解析dtb文件中的各个节点。

## 绑定文档信息

设备树是用来描述板子上的设备信息的，不同的设备其信息不同，反映到设备树中就是属性不同。那么我们在设备树中添加一个硬件对应的节点的时候从哪里查阅相关的说明呢？在Linux 内核源码中有详细的.txt 文档描述了如何添加节点，这些.txt 文档叫做绑定文档，路径为： Linux 源码目录/Documentation/devicetree/bindings 

比如我们现在要想在 I.MX6ULL 这颗 SOC 的 I2C 下添加一个节点，那么就可以查看Documentation/devicetree/bindings/i2c/i2c-imx.txt，此文档详细的描述了 I.MX 系列的 SOC 如何在设备树中添加 I2C 设备节点 

## 设备树常用OF操作函数

设备树描述了设备的详细信息，这些信息包括数字类型的、字符串类型的、数组类型的，我们在编写驱动的时候需要获取到这些信息。比如设备树使用 reg 属性描述了某个外设的寄存器地址为 0X02005482，长度为 0X400，我们在编写驱动的时候需要获取到 reg 属性 然后初始化外设 。 Linux 内核给我们提供了一系列的函数来获取设备树中的节点或者属性信息，这一系列的函数都有一个统一的前缀“of_”，所以在很多资料里面也被叫做 OF 函数。这些 OF 函数原型都定义在 `include/linux/of.h` 文件中 

### 查找节点

Linux 内核使用 device_node 结构体来描述一个节点，此结构体定义在文件 include/linux/of.h 中 

```c
struct device_node {
	const char *name;
	const char *type;
	phandle phandle;
	const char *full_name;
	struct fwnode_handle fwnode;

	struct	property *properties;
	struct	property *deadprops;	/* removed properties */
	struct	device_node *parent;
	struct	device_node *child;
	struct	device_node *sibling;
	struct	kobject kobj;
	unsigned long _flags;
	void	*data;
#if defined(CONFIG_SPARC)
	const char *path_component_name;
	unsigned int unique_id;
	struct of_irq_controller *irq_trans;
#endif
};
```



#### of_find_node_by_name函数

```c
/**
* from：开始查找的节点，如果为 NULL 表示从根节点开始查找整个设备树。
* name：要查找的节点名字。
*
* 返回值： 找到的节点，如果为 NULL 表示查找失败。
*/
struct device_node *of_find_node_by_name(struct device_node *from,const char *name);
```

#### of_find_node_by_type

```c
/**
* from：开始查找的节点，如果为 NULL 表示从根节点开始查找整个设备树。
* type：要查找的节点对应的 type 字符串，也就是 device_type 属性值。
*
* 返回值： 找到的节点，如果为 NULL 表示查找失败。
*/
struct device_node *of_find_node_by_type(struct device_node *from, const char *type)
```

#### of_find_compatible_node

```c
/**
* from：开始查找的节点，如果为 NULL 表示从根节点开始查找整个设备树。
* type：要查找的节点对应的 type 字符串，也就是 device_type 属性值。可以为 NULL，表示忽略掉
* device_type 属性。
* compatible： 要查找的节点所对应的 compatible 属性列表。

* 返回值： 找到的节点，如果为 NULL 表示查找失败。
*/
struct device_node *of_find_compatible_node(struct device_node *from,const char *type,
											const char *compatible)
```

#### of_find_matching_node_and_match

```c
/**
* from：开始查找的节点，如果为 NULL 表示从根节点开始查找整个设备树。
* matches： of_device_id 匹配表，也就是在此匹配表里面查找节点。
* match： 找到的匹配的 of_device_id。

* 返回值： 找到的节点，如果为 NULL 表示查找失败。
*/
struct device_node *of_find_matching_node_and_match(struct device_node *from,
													const struct of_device_id *matches,
													const struct of_device_id **match)
```

#### of_find_node_by_path

```c
/**
* path：带有全路径的节点名，可以使用节点的别名，比如“/backlight”就是 backlight 这个节点的全路径。
* 
* 返回值： 找到的节点，如果为 NULL 表示查找失败
*/

inline struct device_node *of_find_node_by_path(const char *path)
```



### 查找父/子节点

#### of_get_parent

```c
/* 用于获取指定节点的父节点 */
struct device_node *of_get_parent(const struct device_node *node)
```

#### of_get_next_child

```c
/**
* 迭代的方式查找子节点
* node：父节点。
* prev：前一个子节点，也就是从哪一个子节点开始迭代的查找下一个子节点。可以设置为
* NULL，表示从第一个子节点开始。
* 
* 返回值： 找到的下一个子节点。
*/
struct device_node *of_get_next_child(const struct device_node *node,
											struct device_node *prev)
```



### 提取属性值

节点的属性信息里面保存了驱动所需要的内容，因此对于属性值的提取非常重要， Linux 内核中使用结构体 property 表示属性，此结构体同样定义在文件 include/linux/of.h 中 

```c
struct property {
	char	*name;
	int	length;
	void	*value;
	struct property *next;
	unsigned long _flags;
	unsigned int unique_id;
	struct bin_attribute attr;
};
```



#### of_find_property

```c
/**
* of_find_property 函数用于查找指定的属性 
* np: 设备节点
* name：属性名字。
* lenp：如果函数成功找到属性，它会将属性的长度存储在这个指针所指向的整数中。如果不需要获取属性的长度，可以* 将这个参数设置为 NULL。
* 
* 返回值： 找到的属性。
*/
property *of_find_property(const struct device_node *np,
									const char *name,
									int *lenp)
```

#### of_property_count_elems_of_size

```c
/**
* of_property_count_elems_of_size 函数用于获取属性中元素的数量，比如 reg 属性值是一个
* 数组，那么使用此函数可以获取到这个数组的大小
* proname： 需要统计元素数量的属性名字。
* elem_size：元素长度。这是一个整数，表示属性中每个元素的大小（以字节为单位）。
* 返回值： 得到的属性元素数量。
*/
int of_property_count_elems_of_size(const struct device_node *np,
									const char *propname,
                                    int elem_size)
```

#### of_property_read_u32_index

```c
int of_property_read_u32_index(const struct device_node *np,
								const char *propname,
								u32 index,
								u32 *out_value)
const struct device_node *np:

/*
const char *propname:表示要查找的属性的名称。

u32 index:表示要读取的元素在属性数组中的索引位置。函数会从这个索引位置读取对应的 32 位整数值。

u32 *out_value: 这是一个指向无符号 32 位整数的指针，用于存储读取到的值。函数会将读取到的 32 位整数值存储在这个指针所指向的变量中。

如果函数成功读取到指定索引位置的 32 位整数值，则返回 0；如果属性不存在、索引超出范围或读取失败，则返回负的错误码。

 -EINVAL 表示属性不存在， -ENODATA 表示没有
要读取的数据， -EOVERFLOW 表示属性值列表太小。
*/
```

#### of_property_read_u8_array 

of_property_read_u8_array 

of_property_read_u16_array 

of_property_read_u32_array 

of_property_read_u64_array 

这 4 个函数分别是读取属性中 u8、 u16、 u32 和 u64 类型的数组数据，比如大多数的 reg 属性都是数组数据，可以使用这 4 个函数一次读取出 reg 属性中的所有数据。 

```c
/*
out_value：读取到的数组值，分别为 u8、 u16、 u32 和 u64。
sz： 要读取的数组元素数量。函数会从这个属性中读取指定数量的 8 位整数，并存储在 out_values 数组中。
*/
int of_property_read_u8_array(const struct device_node *np,
								const char *propname,
								u8 *out_values,
								size_t sz)
```

#### of_property_read_u8  

of_property_read_u16  

of_property_read_u32 

of_property_read_u64

```c
// 用于读取这种只有一个整形值的属性，分别用于读取 u8、 u16、 u32 和 u64 类型属性值
int of_property_read_u8(const struct device_node *np,
						const char *propname,
						u8 *out_value)
```

返回值： 0，读取成功，负值，读取失败， -EINVAL 表示属性不存在， -ENODATA 表示没有要读取的数据， -EOVERFLOW 表示属性值列表太小。 



#### of_property_read_string 

用于读取属性中字符串值 

```c
int of_property_read_string(struct device_node *np,
							const char *propname,
							const char **out_string)
```

#### int of_n_addr_cells(struct device_node *np) 

用于获取#address-cells 属性值 

#### int of_n_size_cells(struct device_node *np) 

用于获取#size-cells 属性值 

### 其他常用的OF函数

#### of_device_is_compatible

用于查看节点的 compatible 属性是否有包含 compat 指定的字符串，也就是检查设备节点的兼容性 

```c
int of_device_is_compatible(const struct device_node *device,const char *compat)
```

device：设备节点。

compat：要查看的字符串。

返回值： 0，节点的 compatible 属性中不包含 compat 指定的字符串； 正数，节点的 compatible属性中包含 compat 指定的字符串。

#### of_get_address

用于获取地址相关属性，主要是“reg”或者“assigned-addresses”属性值 

```c
const __be32 *of_get_address(struct device_node *dev,
								int index,
								u64 *size,
								unsigned int *flags)
```

用这个函数获取reg信息，假如

```c
reg = <0x10000000 0x1000 0x0>, <0x20000000 0x2000 0x1>;
```

那么索引index就表示获取第几个`<>`中的地址信息，从0开始。

`flags`是标志信息，对应着reg的`<>`中的第三个数据。常用的标志信息有IORESOURCE_IO、 IORESOURCE_MEM 等 

#### of_address_to_resourse

IIC、 SPI、 GPIO 等这些外设都有对应的寄存器，这些寄存器其实就是一组内存空间， Linux内核使用 resource 结构体来描述一段内存空间 “resource”翻译出来就是“资源”，因此用 resource结构体描述的都是设备资源信息， resource 结构体定义在文件 include/linux/ioport.h 中 

```c
/* include/linux/ioport.h */
struct resource {
	resource_size_t start;
	resource_size_t end;
	const char *name;
	unsigned long flags;
	struct resource *parent, *sibling, *child;
};
```

对于 32 位的 SOC 来说， resource_size_t 是 u32 类型的。其中 start 表示开始地址， end 表示结束地址， name 是这个资源的名字， flags 是资源标志位，一般表示资源类型，可选的资源标志定义在文件 include/linux/ioport.h 中。大家一般最常见的资源标志就是 IORESOURCE_MEM 、 IORESOURCE_REG 和IORESOURCE_IRQ 等 

```c
int of_address_to_resource(struct device_node *dev,
							int index,
							struct resource *r)
```

index		：		地址资源标号。
r				：		得到的 resource 类型的资源值。
返回值	  ： 		0，成功；负值，失败。

#### of_iomap

of_iomap 函数用于直接内存映射，以前我们会通过 ioremap 函数来完成物理地址到虚拟地址的映射，采用设备树以后就可以直接通过 of_iomap 函数来获取内存地址所对应的虚拟地址，不需要使用 ioremap 函数了。当然了，你也可以使用 ioremap 函数来完成物理地址到虚拟地址的内存映射，只是在采用设备树以后，大部分的驱动都使用 of_iomap 函数了。 of_iomap 函数本质上也是将 reg 属性中地址信息转换为虚拟地址，如果 reg 属性有多段的话，可以通过 index 参数指定要完成内存映射的是哪一段， 

```c
void __iomem *of_iomap(struct device_node *np,int index)
```

返回值： 经过内存映射后的虚拟内存首地址，如果为 NULL 的话表示内存映射失败。 

# 设备树下的LED驱动

在设备树根目录下存放控制LED的相关寄存器信息

```c
alphaled {
		#address-cells = <1>;
		#aize-cells = <1>;

		compatible = "alientek,led";
		status = "okay";
		reg = <	0x020C406C 0x04
				0X020E0068 0x04
				0X020E02F4 0x04
				0X0209C000 0x04
				0X0209C004 0x04 >;
	};
```

驱动程序中使用相关of函数读取reg节点的值并进行地址映射(ioremap)。

还可以使用of_iomap函数进行寄存器内存映射。

```c
IMX6U_CCM_CCGR1		= of_iomap(dtsled.nd, 0);
SW_MUX_GPIO1_IO03	= of_iomap(dtsled.nd, 1);
SW_PAD_GPIO1_IO03	= of_iomap(dtsled.nd, 2);
GPIO1_DR			= of_iomap(dtsled.nd, 3);
GPIO1_GDIR			= of_iomap(dtsled.nd, 4);
```

```c
#include <linux/types.h>
#include <linux/delay.h>
#include <linux/ide.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/errno.h>
#include <linux/gpio.h>
#include <linux/fs.h>
#include <linux/slab.h>
#include <linux/uaccess.h>
#include <linux/io.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/of.h>
#include <linux/of_address.h>
#include <linux/irq.h>

#include <asm/mach/map.h>
#include <asm/uaccess.h>
#include <asm/io.h>

/* 使用设备树编写LED驱动程序 */
#define DTSLED_COUNT    1
#define DTSLED_NAME     "dtsled"
#define LED_OFF         0
#define LED_ON          1

// 物理地址的虚拟映射
static void __iomem *IMX6U_CCM_CCGR1;
static void __iomem *SW_MUX_GPIO1_IO03;
static void __iomem *SW_PAD_GPIO1_IO03;
static void __iomem *GPIO1_DR;
static void __iomem *GPIO1_GDIR;

struct dtsled_dev {
    dev_t devid;
    int major;
    int minor;
    struct cdev cdev;
    struct class *class;
    struct device *device;
    struct device_node *nd;
};
struct dtsled_dev dtsled;

/**
 * @brief 开关灯函数
 * 
 * @param sta ： LED_ON, LED_OFF
 */
static void led_switch(u8 sta) {
	u32 val = 0;
	if (sta == LED_ON) {
		val = readl(GPIO1_DR);
		val &= ~(1 << 3);
		writel(val, GPIO1_DR);
	} else if (sta == LED_OFF)
	{
		val = readl(GPIO1_DR);
		val |= (1 << 3);
		writel(val, GPIO1_DR);
	}
}

/* fops函数定义区 */
static int dtsled_open(struct inode *inode, struct file *filp) {
    filp->private_data = &dtsled;
    return 0;
}

static int dtsled_release(struct inode *inode, struct file *filp) {
    return 0;
}

static ssize_t dtsled_read(struct file *filp, char __user *buf, size_t cnt, loff_t *offt) {
    return 0;
}

static ssize_t dtsled_write(struct file *filp, const char __user *buf, size_t cnt, loff_t *offt) {
    u64 retvalue;
	u8 databuf[1];
	u8 ledstat;

    retvalue = copy_from_user(databuf, buf, cnt);
    if (retvalue < 0) {
		printk("kernel write failed!\r\n");
		return -EFAULT;
	}

    ledstat = databuf[0];
	led_switch(ledstat);

    return 0;
}

const static struct file_operations dtsled_fops = {
    .owner = THIS_MODULE,
    .open = dtsled_open,
    .release = dtsled_release,
    .read = dtsled_read,
    .write = dtsled_write,
};

static int __init dtsled_init(void) {
    int ret;

    /* LED驱动初始化 */
    u32 val = 0;
    const char *str;
    u32 regdata[10];
    u8 i = 0;

	/* 1. 寄存器映射，初始化LED相关寄存器 */
    /* 读取设备树属性内容 */
    dtsled.nd = of_find_node_by_path("/alphaled");
    if(dtsled.nd == NULL) {
        ret = -EINVAL;
        goto fail_find;
    }
    ret = of_property_read_string(dtsled.nd, "status", &str);
    if(ret < 0) {
        goto fail_rs;
    } else {
        printk("status = %s\r\n", str);
    }

    ret = of_property_read_string(dtsled.nd, "compatible", &str);
    if(ret < 0) {
        goto fail_rs;
    } else {
        printk("compatible = %s\r\n", str);
    }

    ret = of_property_read_u32_array(dtsled.nd, "reg", regdata, 10);
    if(ret < 0) {
        goto fail_rs;
    } else {
        for(i = 0; i < 10; i++) {
            printk("regdata[%d] = %#X\r\n", i, regdata[i]);
        }
    }

	IMX6U_CCM_CCGR1		= ioremap(regdata[0], regdata[1]);
	SW_MUX_GPIO1_IO03	= ioremap(regdata[2], regdata[3]);
	SW_PAD_GPIO1_IO03	= ioremap(regdata[4], regdata[5]);
	GPIO1_DR			= ioremap(regdata[6], regdata[7]);
	GPIO1_GDIR			= ioremap(regdata[8], regdata[9]);

	/* 2. 使能时钟 */
	val = readl(IMX6U_CCM_CCGR1);
	val &= ~(3 << 26);
	val |= (3 << 26);
	writel(val, IMX6U_CCM_CCGR1);

	/* 3. 设置复用 */
	writel(5, SW_MUX_GPIO1_IO03);

	/* 4. 设置io属性 */
	writel(0x10B0, SW_PAD_GPIO1_IO03);

	/* 5. 设置为输出 */
	val = readl(GPIO1_GDIR);
	val &= ~(1 << 3);
	val |= (1 << 3);
	writel(val, GPIO1_GDIR);

	/* 6. 默认关闭LED */
	val = readl(GPIO1_DR);
	val |= (1 << 3);
	writel(val, GPIO1_DR);

    /* 注册设备号 */
    dtsled.major = 0;
    if(dtsled.major) {
        dtsled.devid = MKDEV(dtsled.major, 0);
        ret = register_chrdev_region(dtsled.devid, DTSLED_COUNT, DTSLED_NAME);
    } else {
        ret = alloc_chrdev_region(&dtsled.devid, 0, DTSLED_COUNT, DTSLED_NAME);
    }
    if (ret < 0) {
        goto fail_devid;
    }

    /* 注册设备 */
    dtsled.cdev.owner = THIS_MODULE;
    cdev_init(&dtsled.cdev, &dtsled_fops);
    ret = cdev_add(&dtsled.cdev, dtsled.devid, DTSLED_COUNT);
    if(ret < 0) {
        goto fail_cdev;
    }

    /* 注册类设备 */
    dtsled.class = class_create(THIS_MODULE, DTSLED_NAME);
    if(IS_ERR(dtsled.class)) {
        ret = PTR_ERR(dtsled.class);
        goto fail_class;
    }
    dtsled.device = device_create(dtsled.class, NULL, dtsled.devid, NULL, DTSLED_NAME);
    if(IS_ERR(dtsled.device)) {
        ret = PTR_ERR(dtsled.device);
        goto fail_device;
    }

    return 0;

fail_rs:

fail_find:
    device_destroy(dtsled.class, dtsled.devid);

fail_device:
    class_destroy(dtsled.class);

fail_class:
    cdev_del(&dtsled.cdev);

fail_cdev:
    unregister_chrdev_region(dtsled.devid, 1);

fail_devid:

    
    return ret;
}

static void __exit dtsled_exit(void) {
    /* 取消寄存器映射 */
	iounmap(IMX6U_CCM_CCGR1);
	iounmap(SW_MUX_GPIO1_IO03);
	iounmap(SW_PAD_GPIO1_IO03);
	iounmap(GPIO1_DR);
	iounmap(GPIO1_GDIR);
    device_destroy(dtsled.class, dtsled.devid);
    class_destroy(dtsled.class);
    cdev_del(&dtsled.cdev);
    unregister_chrdev_region(dtsled.devid, 1);
}

module_init(dtsled_init);
module_exit(dtsled_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("wsy");
```

# pinctrl和GPIO子系统实验

## pinctrl

主要用来设置pin的复用和电气属性

1. 获取设备树中的pin信息
2. 根据获取到的pin信息来设置复用
3. 根据pin信息来设置电气属性

对于我们使用者来讲，只需要在设备树里面设置好某个 pin 的相关属性即可，其他的初始化工作均由 pinctrl 子系统来完成，pinctrl 子系统源码目录为 **drivers/pinctrl**。

pinctrl的使用流程：

1. **imx6ull.dtsi**文件中有一个**iomuxc**节点。**imx6ull-alientek-emmc.dts**中有追加内容**&iomuxc**，里面有各种外设使用到的节点的复用信息。我们需要在这里创建自己的子节点。

2. 在子节点中创建复用和电气属性

   /iomuxc/imx6ul-evk/pinctrl_xxx: xxx {

   ​	fsl,pins = <MX6ULxxx		0xxxxx>;

   }

   ```c
   MX6UL_PAD_UART1_RTS_B__GPIO1_IO19 0x17059
   // 0x17059 就是 conf_reg 寄存器值
   ```

   宏定义的位置**arch/arm/boot/dts/imx6ul-pinfunc.h **.imx6ull-pinfunc.h会引用imx6ul-pinfunc.h这个文件

   ```c
   #define MX6UL_PAD_UART1_RTS_B__GPIO1_IO19 0x0090 0x031C 0x0000 0x5 0x0
   // 五个值的含义分别为<mux_reg conf_reg input_reg mux_mode input_val>
   // 分别表示 
   // 复用寄存器的偏址 
   // 电气配置寄存器的偏址 
   // input寄存器偏址，有些pin没有input就不需要设置 
   // 复用寄存器的值，0x5表示复用为GPIO1_IO19 
   // input_reg寄存器的值，这里无效
   ```

### pinctrl驱动程序详解

- 首先根据节点**iomuxc**的**compatible**属性来搜索对应的驱动文件。

- drivers/pinctrl/freescale/pinctrl-imx6ul.c这个文件中的程序会根据上面属性中的内容匹配。of_device_id类型的结构体里面保存着这个驱动文件的兼容值。

- 当设备和驱动程序成功匹配之后，平台驱动platform_driver类型结构体中的probe成员变量所对应的函数就会执行。

- probe成员变量对应的函数会解析iomuxc中的pin信息并且想Linux注册pinctrl控制器。

  随后imx_pinctrl_parse_groups函数中会调用pinctrl_register函数

  Linux内核程序有面向对象的编程思想，pinctrl_dev类型的结构体是一个类，使用pinctrl_register函数注册一个pinctrl控制器随后返回一个该类型的结构体指针。

- 上一条的pinctrl_register函数中有一个参数为pctldesc为pinctrl_desc 结构体指针，此参数就是要注册的PIN控制器，PIN 控制器用于配置 SOC 的 PIN 复用功能和电气特性。

  pinctrl_desc 结构体中有**pctlops**，**pmxops**，**confops**，这三个ops提供了很多操作函数，通过这些操作函数就可以完成对某一个PIN的配置。

### 在设备树中添加pinctrl节点

/iomuxc/imx6ul-evk/pinctrl_xxx: xxx {

​	fsl,pins = <MX6ULxxx		0xxxxx>;

}

## GPIO子系统

初始化GPIO并提供相应的API函数

gpio设计流程

1. 在自己板子的dts目录下添加设备节点

   ```
   &usdhc1 {
   	pinctrl-names = "default", "state_100mhz", "state_200mhz";
   	pinctrl-0 = <&pinctrl_usdhc1>;
   	pinctrl-1 = <&pinctrl_usdhc1_100mhz>;
   	pinctrl-2 = <&pinctrl_usdhc1_200mhz>;
   	/* pinctrl-3 = <&pinctrl_hog_1>; */
   	cd-gpios = <&gpio1 19 GPIO_ACTIVE_LOW>;
   	keep-power-in-suspend;
   	enable-sdio-wakeup;
   	vmmc-supply = <&reg_sd1_vmmc>;
   	status = "okay";
   };
   ```

   pinctrl-names用于定义引脚控制的状态名称，与下面的pinctrl-n对应，用于分辨不同设备状态下的pinctrl控制器。

   ### gpio驱动程序简介

   drivers/gpio/gpio-mxc.c芯片的GPIO驱动文件。有of_device_id 匹配表。

   gpio-mxc.c 所在的目录为 drivers/gpio，打开这个目录可以看到很多芯片的 gpio 驱动文件， “gpiolib”开始的文件是 gpio 驱动的核心文件。

   gpio-mxc.c有平台驱动结构体。运行probe函数。

   ​	第 406 行，定义一个结构体指针 port，结构体类型为 mxc_gpio_port。gpio-mxc.c 的重点工作就是维护              	mxc_gpio_port。mxc_gpio_port 的 bgc 成员变量很重要，因为稍后的重点就是初始化 bgc(bgpio_chip类	型结构体)。

   ​	第 411 行调用 mxc_gpio_get_hw 函数获取 gpio 的硬件相关数据，其实就是 gpio 的寄存器组
   
   ```c
   static struct mxc_gpio_hwdata imx35_gpio_hwdata = {
   	.dr_reg = 0x00,
   	.gdir_reg = 0x04,
   	.psr_reg = 0x08,
   	.icr1_reg = 0x0c,
   	.icr2_reg = 0x10,
   	.imr_reg = 0x14,
   	.isr_reg = 0x18,
   	.edge_sel_reg = 0x1c,
   	.low_level = 0x00,
   	.high_level = 0x01,
   	.rise_edge = 0x02,
   	.fall_edge = 0x03,
};
   ```

   ​	第 417 行，调用函数platform_get_resource 获取设备树中内存资源信息，也就是 reg 属性值。前面说了 reg 属性指定了 GPIO1 控制器的寄存器基地址为 0X0209C000，在配合前面已经得到的 mxc_gpio_hwdata，这样 Linux 内核就可以访问 gpio1 的所有寄存器了。第 418 行，调用 devm_ioremap_resource 函数进行内存映射，得到 0x0209C000 在 Linux 内核中的虚拟地址。

   ​	...后续见文档

   ​	第 450~453 行，bgpio_init 函数第一个参数为 bgc，是 bgpio_chip 结构体指针。bgpio_chip结构体有个 gc 成员变量，gc 是个 gpio_chip 结构体类型的变量。gpio_chip 结构体是抽象出来的GPIO 控制器可以看出，gpio_chip 大量的成员都是函数，这些函数就是 GPIO 操作函数。bgpio_init 函数主要任务就是初始化 bgc->gc 。

   ​	继续回到mxc_gpio_probe 函数，第461行调用函数gpiochip_add向Linux内核注册gpio_chip，
   
   ​	也就是 port->bgc.gc。注册完成以后我们就可以在驱动中使用 gpiolib 提供的各个 API 函数。

# gpioled实验

```c
/* 获取设备节点 */
    gpioled.nd = of_find_node_by_path("/gpioled");
    if(gpioled.nd == NULL) {
        printk("find_node fail!\r\n");
        ret = -EINVAL;
        goto fail_node;
    }

    /* 获取设备树中的GPIO属性，得到LED所使用的GPIO编号 */
    gpioled.led_gpio = of_get_named_gpio(gpioled.nd, "led-gpio", 0);
    if(gpioled.led_gpio < 0) {
        printk("get gpio failed!\r\n");
        ret = -EINVAL;
        goto fail_getgpio;
    }
    printk("gpio number : %d\r\n", gpioled.led_gpio);

    /* 申请GPIO */
    ret = gpio_request(gpioled.led_gpio, "led_gpio");
    if(ret) {
        printk("gpio request fail!\r\n");
        goto fail_gpiorequest;
    }

    /* 设置输出高电平默认关闭led */
    ret = gpio_direction_output(gpioled.led_gpio, 0);
    if (ret < 0)
    {
        printk("gpio output failed!\r\n");
        goto fail_gpioout;
    }



/* LED开关函数 */
static void led_switch(u8 sta) {
    if(sta == LED_ON) {
        gpio_set_value(gpioled.led_gpio, 0);
    } else if (sta == LED_OFF) {
        gpio_set_value(gpioled.led_gpio, 1);
    }
}


static ssize_t led_write(struct file *f, const char __user *buf,	size_t count, loff_t *off) {
    u8 databuf[1];
    int ret;
    ret = copy_from_user(databuf, buf, count);
    if(ret < 0) {
        printk("kernel write failed!\r\n");
    }
    led_switch(databuf[0]);
    return 0;
}
```

设备树

```c
&iomuxc {
	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_hog_1>;
	imx6ul-evk {
		pinctrl_gpioled: ledgrp {
			fsl,pins = <
				MX6UL_PAD_GPIO1_IO03__GPIO1_IO03	0x10b0
			>;
		};
       
/{
    gpioled {
		compatible = "alientek,gpioled";	
		pinctrl-names = "default";
		pinctrl-0 = <&pinctrl_gpioled>;
		led-gpio = <&gpio1 3 GPIO_ACTIVE_LOW>;
		status = "okay";
	};
}
```

# 蜂鸣器实验

程序结构与LED实验相同

设备树：

```sh
beep {
		compatible = "alientek,beep";	
		pinctrl-names = "default";
		pinctrl-0 = <&pinctrl_beep>;
		beep-gpio = <&gpio5 1 GPIO_ACTIVE_LOW>;
		status = "okay";
	};
};

pinctrl_beep: beepgrp {
    fsl,pins = <
    	MX6ULL_PAD_SNVS_TAMPER1__GPIO5_IO01	0x10b0
    >;
};
```



驱动程序如下：

```c
#include <linux/types.h>
#include <linux/delay.h>
#include <linux/ide.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/errno.h>
#include <linux/gpio.h>
#include <linux/fs.h>
#include <linux/slab.h>
#include <linux/uaccess.h>
#include <linux/io.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/of.h>
#include <linux/of_address.h>
#include <linux/irq.h>
#include <linux/of_gpio.h>

#include <asm/mach/map.h>
#include <asm/uaccess.h>
#include <asm/io.h>

/* 宏定义 */
#define MODULE_NAME     "beep"
#define MODULE_CNT      1
#define BEEP_ON         1
#define BEEP_OFF        0

struct beep_dev {
    dev_t               devid;
    int                 major;
    int                 minor;
    struct cdev         cdev;
    struct class        *class;
    struct device       *device;
    struct device_node  *nd;
    int                 beep_gpio;
};

struct beep_dev beep;

/* beep开关函数 */
static void beep_switch(u8 sta) {
    if(sta == BEEP_ON) {
        gpio_set_value(beep.beep_gpio, 0);
    } else if (sta == BEEP_OFF) {
        gpio_set_value(beep.beep_gpio, 1);
    }
}

static int beep_open(struct inode *inode, struct file *file) {
    file->private_data = &beep;
    return 0;
}

static int beep_release(struct inode *inode, struct file *file) {
    return 0;
}

static ssize_t beep_read(struct file *f, char __user *buf, size_t count, loff_t *off) {
    return 0;
}

static ssize_t beep_write(struct file *f, const char __user *buf,	size_t count, loff_t *off) {
    u8 databuf[1];
    int ret;
    ret = copy_from_user(databuf, buf, count);
    if(ret < 0) {
        printk("kernel write failed!\r\n");
    }
    beep_switch(databuf[0]);
    return 0;
}

static const struct file_operations beep_fops = {
    .owner      =       THIS_MODULE,
    .open       =       beep_open,
    .release    =       beep_release,
    .read       =       beep_read,
    .write      =       beep_write,
};

/* 出入口函数 */
static int __init beep_init(void) {
    int ret;

    /* 设备号 */
    beep.major = 0;
    if(beep.major) {
        ret = register_chrdev_region(beep.devid, MODULE_CNT, MODULE_NAME);
    } else {
        ret = alloc_chrdev_region(&beep.devid, 0, MODULE_CNT, MODULE_NAME);
    }
    if(ret < 0) {
        printk("fail chrdev!\r\n");
        goto fail_chrdev;
    }

    /* 注册设备 */
    beep.cdev.owner = THIS_MODULE;
    cdev_init(&beep.cdev, &beep_fops);
    ret = cdev_add(&beep.cdev, beep.devid, MODULE_CNT);
    if(ret < 0) {
        printk("cdev fail!\r\n");
        goto fail_cdev;
    }

    /* class */
    beep.class = class_create(THIS_MODULE, MODULE_NAME);
    if(IS_ERR(beep.class)) {
        ret = PTR_ERR(beep.class);
        printk("Fail class!\r\n");
        goto fail_class;
    }

    /* device */
    beep.device = device_create(beep.class, NULL, beep.devid, NULL, MODULE_NAME);
    if(IS_ERR(beep.device)) {
        ret = PTR_ERR(beep.device);
        printk("Fail device!\r\n");
        goto fail_device;
    }

    /* 获取设备节点 */
    beep.nd = of_find_node_by_path("/beep");
    if(beep.nd == NULL) {
        printk("find_node fail!\r\n");
        ret = -EINVAL;
        goto fail_node;
    }

    /* 获取GPIO */
    beep.beep_gpio = of_get_named_gpio(beep.nd, "beep-gpio", 0);
    if(beep.beep_gpio < 0) {
        printk("get gpio failed!\r\n");
        ret = -EINVAL;
        goto fail_node;
    }
    printk("gpio number : %d\r\n", beep.beep_gpio);

    /* 申请GPIO */
    ret = gpio_request(beep.beep_gpio, "led_gpio");
    if(ret < 0) {
        printk("gpio request fail!\r\n");
        goto fail_node;
    }

    /* 设置输出高电平默认关闭beep */
    ret = gpio_direction_output(beep.beep_gpio, 1);
    if (ret < 0)
    {
        printk("gpio output failed!\r\n");
        goto fail_gpioout;
    }


    return 0;
fail_gpioout:
    gpio_free(beep.beep_gpio);

fail_node:
    device_destroy(beep.class, beep.devid);

fail_device:
    class_destroy(beep.class);

fail_class:
    cdev_del(&beep.cdev);

fail_cdev:
    unregister_chrdev_region(beep.devid, MODULE_CNT);

fail_chrdev:

    return ret;
}

static void __exit beep_exit(void) {
    gpio_free(beep.beep_gpio);
    device_destroy(beep.class, beep.devid);
    class_destroy(beep.class);
    cdev_del(&beep.cdev);
    unregister_chrdev_region(beep.devid, MODULE_CNT);
}

module_init(beep_init);
module_exit(beep_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("wsy");
```

# linux的并发与竞争

在驱动开发中要注意对共享资源的保护。

临界区的概念：所谓的临界区就是共享数据段，对于临界区必须保证每次只有一个线程访问，也就是保证临界区是原子访问的。一般全局变量，设备结构体这些是要保护的。

## 原子操作

- 定义atomic_t类型的变量，定义变量的时候赋值atomic_t b = ATOMIC_INIT(0); 
- 给原子变量写入值void atomic_set(atomic_t *v, int i) 
- 原子变量自增void atomic_inc(atomic_t *v); 原子变量自减void atomic_dec(atomic_t *v);
- 原子变量自减如果结果为0就返回真。int atomic_dec_and_test(atomic_t *v);

测试程序：通过原子变量的值来检测LED有没有被别的程序使用

```c
// 定义原子变量 设备结构体
struct gpioled_dev {
 	...
    atomic_t lock;       // 标志LED驱动是否被应用使用
};
// 给原子变量赋值 init函数
atomic_set(&gpioled.lock, 1);
// 通过原子变量检测 open函数
if(!atomic_dec_and_test(&gpioled.lock)) {
    atomic_inc(&gpioled.lock);
    return -EBUSY;
}
// 释放原子变量 release函数
atomic_inc(&gpioled.lock);
```

## 自旋锁

原子操作只能对整型变量或者位进行操作。当一个线程想要获取共享资源的时候首先要获取相应的锁，当当前的锁被另一个线程持有的时候，当前线程便不能访问此资源，处于**等待状态**。

自旋锁的缺点：等待的线程会一直处于自旋状态，浪费处理器的时间，降低系统性能。所以自旋锁适合短时间轻量级加锁。

- 定义一个自旋锁变量spinlock_t	lock; 定义并初始化自旋变量DEFINE_SPINLOCK(spinlock_t lock) 
- 初始化自旋锁int spin_lock_init(spinlock *lock);
- 获取自旋锁void spin_lock(spinlock *lock);
- 释放自旋锁void spin_unlock(spinlock *lock);

int spin_trylock(spinlock_t *lock) 尝试获取指定的自旋锁，如果没有获取到就返回 0                 

int spin_is_locked(spinlock_t *lock) 检查指定的自旋锁是否被获取，如果没有被获取就返回非 0，否则返回 0。 

注意：被自旋锁保护的临界区一定不能调用任何能够引起休眠或者阻塞的API函数，否则可能会导致死锁现象的发生。自旋锁会自动禁止抢占，如果持有锁的线程A在持有锁期间进入了休眠状态，那么线程A会**自动放弃CPU使用权**，线程B开始运行，如果线程B也想运行锁，但是无法获取，B不执行结束，A也无法继续执行，死锁现象发生了。

线程A运行过程中，中断也想访问共享资源的情况。首先可以肯定的是，中断里面也可以使用自旋锁，但是在中断里面使用自旋锁的时候一定要禁止本地中断，也就是禁止本CPU中断(多核)，否则可能导致死锁现象的发生。线程 A 先运行，并且获取到了 lock 这个锁，当线程 A 运行 functionA 函数的时候中断发生了，中断抢走了 CPU 使用权。右边的中断服务函数也要获取 lock 这个锁，但是这个锁被线程 A 占有着，中断就会一直自旋，等待锁有效。最好的解决方法就是获取锁之前关闭本地中断。

不能递归地申请自旋锁。

- void spin_lock_irqsave(spinlock_t *lock, unsigned long flags) 保存中断状态，禁止本地中断，并获取自旋锁。
- void spin_unlock_irqrestore(spinlock_t *lock, unsigned long flags) 将中断状态恢复到以前的状态，并且激活本地中断，释放自旋锁。 

程序编写：

```c
/* 使用自旋锁实现对设备的互斥访问。互斥访问主要是靠定义的标志量实现，自旋锁的作用主要是对表质量值的改变做一个保护 */
// 定义设备结构体
struct gpioled_dev {
	...
    int dev_sta;
    spinlock_t lock;
};

// init
spin_lock_init(&gpioled.lock);

// open
spin_lock_irqsave(&gpioled.lock, flags);
if(gpioled.dev_sta) {
    spin_unlock_irqrestore(&gpioled.lock, flags);
    return -EBUSY;
}
gpioled.dev_sta--;                          /* 打开时标志位减一 */
spin_unlock_irqrestore(&gpioled.lock, flags);

// release
spin_lock_irqsave(&gpioled.lock, flags);    /* 释放时标志位加一 */
gpioled.dev_sta++;
spin_unlock_irqrestore(&gpioled.lock, flags);
```



## 信号量

要使用信号量必须添加<linux/semaphore.h>头文件。 

信号量可以指定允许几个线程访问。

信号量不能用于中断中，因为信号量会引起休眠而中断不能休眠。

```c
struct semaphore sem; /* 定义信号量 */
sema_init(&sem, 1); /* 初始化信号量 */
down(&sem); /* 申请信号量 */
/* 临界区 */
up(&sem); /* 释放信号量 */


int down_trylock(struct semaphore *sem); // 尝试获取信号量，如果能获取到信号量就获取，并且返回 0。如果不能就返回非 0，并且不会进入休眠。
```

## 互斥体

互斥体同n=1时的信号量

```c
struct mutex lock; /* 定义一个互斥体 */
mutex_init(&lock); /* 初始化互斥体 */

mutex_lock(&lock); /* 上锁 */
/* 临界区 */
mutex_unlock(&lock); /* 解锁 */
```

# Linux系统定时器使用

基本概念

- 定时器：一个数据结构，包含了定时器的到期时间，回调函数以及其他相关信息。
- jiffies: Linux内核中的一个全局变量，表示系统启动以来的时钟滴答数，每个滴答对应一个固定的时间间隔。
- 定时器队列*：内核维护的一个队列，用于管理已经注册的所有定时器。
  - `add_timer()` 函数将定时器添加到内核的定时器队列中。

内核定时器的一般使用流程

```c
struct timer_list timer;	/* 定义定时器 */

/* 定时器回调函数 */
void function(unsigned long arg) {
    /* 定时器处理代码 */
    
    /* 如果需要定时器周期运行的话就使用mod_timer
     * 函数重新设置定时值并启动定时器
     */
    mod_timer(&dev->timertest, jiffies + msecs_to_jiffies(2000));
}

/* 初始化函数 */
void init(void) {
	init_timer(&timer);
    
    timer.function = function;
    timer.expire = jiffies + msecs_to_jiffiecs(2000);	/* 超时时间2秒 */
    timer.data = (unsigned long)&dev				/* 将设备结构体作为参数 */
   	add_timer(&timer);								/* 启动定时器 */
}

/* 退出函数 */
void exit(void) {
    del_timer(&timer);
    /* 或者使用: del_timer_sync函数，该函数不能用于中断 */
}
```

## Linux内核短延时函数

```c
void ndelay(unsigned long nsecs);
void udelay(unsigned long usecs);
void mdelay(unsigned long msecs);
```

## 写驱动实现LED灯闪烁

初始化定时器

回调函数中重新设置定时周期

```c
// 全部代码
#include <...>

#define     DEV_NAME        "ledtimer"
#define     DEV_CNT         1
#define     LED_ON          1
#define     LED_OFF         0

struct ledtimer_dev {
    dev_t devid;
    int major;
    int minor;
    struct cdev cdev;
    struct class *class;
    struct device *device;

    struct device_node *nd;
    int led_gpio;
    struct timer_list timer;        // 定义系统定时器
    int leddelay;
    int ledsta;
};

struct ledtimer_dev ledtimer;

/* LED开关函数 */
static void led_switch(u8 sta) {
    if(sta == LED_ON) {
        gpio_set_value(ledtimer.led_gpio, 0);
    } else if (sta == LED_OFF) {
        gpio_set_value(ledtimer.led_gpio, 1);
    }
}

/* 定时器回调函数 */
void function(unsigned long arg) {
    /* 定时器处理代码 */
    static u8 sta = 1;
    sta = !sta;
    led_switch(sta);
    /* 如果需要定时器周期运行的话就使用mod_timer
     * 函数重新设置定时值并启动定时器
     */
    mod_timer(&ledtimer.timer, jiffies + msecs_to_jiffies(ledtimer.leddelay));
}

static int ledtimer_open(struct inode *inode, struct file *file) {
    file->private_data = &ledtimer;
    return 0;
}
...

struct file_operations ledtimer_fops = {
    .owner      =       THIS_MODULE,
    .open       =       ledtimer_open,
    .release    =       ledtimer_release,
    .read       =       ledtimer_read,
    .write      =       ledtimer_write,
};

static int __init ledtimer_init(void) {
    int ret;

    ...

    /* 获取LED对应的GPIO */
    ledtimer.led_gpio = of_get_named_gpio(ledtimer.nd, "led-gpio", 0);
    if(ledtimer.led_gpio < 0) {
        printk("get gpio failed!\r\n");
        ret = -EINVAL;
        goto fail_getgpio;
    }
    printk("gpio number : %d\r\n", ledtimer.led_gpio);
    /* 申请GPIO */
    ret = gpio_request(ledtimer.led_gpio, "led_gpio");
    if(ret) {
        printk("gpio request fail!\r\n");
        goto fail_gpiorequest;
    }
    /* 设置输出高电平默认关闭LED */
    ret = gpio_direction_output(ledtimer.led_gpio, 1);
    if (ret < 0) {
        printk("gpio output failed!\r\n");
        goto fail_gpioout;
    }

    /* 初始化定时器 */
    ledtimer.leddelay = 500;                                                /* 延时时间设置为500ms */
    init_timer(&ledtimer.timer);
    ledtimer.timer.function = function;
    ledtimer.timer.expires = jiffies + msecs_to_jiffies(ledtimer.leddelay);	/* 超时时间500ms，这个参数用于设置定时器的到期时间 */
    ledtimer.timer.data = (unsigned long)&ledtimer; 		                /* 将设备结构体作为参数, 回调函数的参数 */
    add_timer(&ledtimer.timer);								                /* 启动定时器 */
    return 0;

fail_gpioout:
  	...
    return ret;
}

static void __exit ledtimer_exit(void) {
    del_timer(&ledtimer.timer);
    gpio_free(ledtimer.led_gpio);                       // 释放ledGpio
    device_destroy(ledtimer.class, ledtimer.devid);     // 释放设备
    class_destroy(ledtimer.class);                      // 释放类
    cdev_del(&ledtimer.cdev);                           // 删除cdev
    unregister_chrdev_region(ledtimer.devid, DEV_CNT);  // 删除设备号
}

module_init(ledtimer_init);
...
```

## ioctl控制器

ioctl也属于fops结构体中的函数，应用程序通过系统调用来运行该函数内容

控制开关闪烁周期，CMD命令的定义

使用自旋锁0设置周期值保护

这个例程实现：关闭打开定时器设置定时周期



- CMD命令规则

  在编写ioctl之前通常需要定义一些控制命令通过以下宏定义

  ```c
  _IO(type, nr)					// 不涉及数据传递的命令
  _IOR(type, nr, data_type)		// 从驱动程序读取数据
  _IOW(type, nr, data_type)		// 将数据写入驱动程序
  _IOWR(type, nr, data_type): 	// 双向数据传递
  /* type	: 类型码，通常使用一个ASCII字符表示设备的类型，比如LED使用'L'
   * nr	: 命令号，用于唯一表示该命令的编号。例如编号0可能代表“关闭LED”，1代表“打开LED”。
   * data_type : 需要传递的数据指针 _IOW('L', 2, &dalay)
   */
  
  // 驱动中读取数据
  copy_from_user(&to_delay, (int *)arg, sizeof(int));
  ```

程序代码：

```c
struct ledtimer_dev {
    dev_t devid;
    int major;
    int minor;
    struct cdev cdev;
    struct class *class;
    struct device *device;

    struct device_node *nd;
    int led_gpio;
    struct timer_list timer;        // 定义系统定时器
    int leddelay;
    int ledsta;
    spinlock_t lock;
};

 spin_lock_init(&ledtimer.lock);

static long ledtimer_ioctl(struct file *file, unsigned int cmd, unsigned long arg) {
    struct ledtimer_dev *dev = file->private_data;
    unsigned long flags;
    int delay;

    switch(cmd) {
        case CLOSE_CMD:             /* 关闭定时器 */
            printk("Timer turned off\r\n");
            del_timer_sync(&dev->timer);
            break;

        case OPEN_CMD:              /* 开启定时器 */
            printk("Timer turned on\r\n");
            mod_timer(&dev->timer, jiffies + msecs_to_jiffies(dev->leddelay));
            break;

        case SETPERIOD_CMD:         /* 设置定时器参数 */
            if(copy_from_user(&delay, (int *)arg, sizeof(int))) {
                printk("copy_from_user failed!\r\n");
                return -EFAULT;
            }
            printk("LED blink delay set to %d ms\r\n", delay);

            spin_lock_irqsave(&dev->lock, flags);
            dev->leddelay = delay;
            spin_unlock_irqrestore(&dev->lock, flags);
            
            mod_timer(&dev->timer, jiffies + msecs_to_jiffies(dev->leddelay));
            
            break;
        default:
            break;
    }
    return 0;
}

struct file_operations ledtimer_fops = {
    .owner      =       THIS_MODULE,
    .open       =       ledtimer_open,
    .release    =       ledtimer_release,
    .read       =       ledtimer_read,
    .write      =       ledtimer_write,
    .unlocked_ioctl =   ledtimer_ioctl,
};

// 应用程序
while (1) {
    printf("Please input cmd(0:CLOSE_CMD; 1:OPEN_CMD 2: SETPERIOD_CMD): ");
    ret = scanf("%d", &cmd);
    if (ret < 0) {
        gets(str);
    }

    if (cmd == 0)
        ioctl(fd, CLOSE_CMD);
    else if (cmd == 1) 
        ioctl(fd, OPEN_CMD);
    else if(cmd == 2) {
        printf("Please input delay value(ms): ");
        ret = scanf("%d", &delay);
        if (ret < 0) {
            gets(str);
        }
        ioctl(fd, SETPERIOD_CMD, &delay);
    } else {
        break;
    }
}
```

# 中断

## 上半部 下半部

中断处理分为**上半部**和**下半部**

- 上半部就是中断处理函数，用于处理比较快，不会占用很长时间的处理
- 下半部用于处理比较耗时的程序，这样上半部就可以快进快出

实现下半部的方法

Linux实现一个驱动的方法流程一般是：定义结构体，初始化（注册函数之类），打开，关闭

## 软中断

- 软中断（Softirq）是在硬中断处理程序（硬中断上下文）执行完成后的一种延迟执行机制。软中断通常由内核分配，并通过特定的机制来触发和执行。在内核代码中，软中断的典型应用包括网络协议栈的数据包处理、块设备的数据传输等。用于**时间非敏感型**的任务。
- 注册软中断**open_softirq(int nr, void (*handler)(struct softirq_action *))**
- **local_bh_enable()**重新启用本地的软中断处理。

- **local_bh_disable()**禁用本地软中断，防止在当前执行路径中被其他软中断处理程序打断。通常在执行关键代码段时使用，避免软中断中断正在执行的任务。

- 触发软中断使用的函数**raise_softirq**
- **内核会在中断退出或定时器中调用 `do_softirq` 函数来检查并执行已触发的软中断。**

- 硬中断和软中断一起作用的示例

  ```c
  #include <linux/module.h>
  #include <linux/interrupt.h>
  #include <linux/kernel.h>
  #include <linux/init.h>
  #include <linux/gpio.h>
  
  #define IRQ_LINE 1                 // 模拟的硬件中断号，通常用真实设备 IRQ 号
  #define DEVICE_NAME "my_device"    // 设备名
  
  // 软中断标识符
  static struct softirq_action my_softirq_action;
  
  // 中断处理函数
  static irqreturn_t irq_handler(int irq, void *dev_id) {
      printk(KERN_INFO "Interrupt handler: IRQ %d triggered\n", irq);
  
      // 触发软中断
      raise_softirq(MY_SOFTIRQ);
      return IRQ_HANDLED; // 表示中断已经被处理
  }
  
  // 软中断处理函数
  static void my_softirq_handler(struct softirq_action *action) {
      printk(KERN_INFO "Softirq handler: executing delayed task\n");
      // 延迟处理逻辑，如数据处理等
  }
  
  // 初始化模块
  static int __init my_module_init(void) {
      int result;
  
      // 注册软中断
      open_softirq(MY_SOFTIRQ, my_softirq_handler);
      printk(KERN_INFO "Softirq registered.\n");
  
      // 注册硬中断
      result = request_irq(IRQ_LINE, irq_handler, IRQF_SHARED, DEVICE_NAME, &my_softirq_action);
      if (result) {
          printk(KERN_ERR "Failed to request IRQ\n");
          return result;
      }
  
      printk(KERN_INFO "Interrupt and softirq module loaded.\n");
      return 0;
  }
  
  // 退出模块
  static void __exit my_module_exit(void)
  {
      // 释放硬中断
      free_irq(IRQ_LINE, &my_softirq_action);
      printk(KERN_INFO "Interrupt released.\n");
  
      printk(KERN_INFO "Softirq module unloaded.\n");
  }
  
  module_init(my_module_init);
  module_exit(my_module_exit);
  MODULE_LICENSE("GPL");
  ```

  ## tasklet

- tasklet是利用软中断来实现的另外一种下半部机制，区别于软中断。

  ```c
  /* 定义 taselet */
  struct tasklet_struct testtasklet;
  /* tasklet 处理函数 */
  void testtasklet_func(unsigned long data)
  {
  /* tasklet 具体处理内容 */
  }
  /* 中断处理函数 */
  irqreturn_t test_handler(int irq, void *dev_id)
  {
  ......
  /* 调度 tasklet */
  tasklet_schedule(&testtasklet);
  ......
  }
  /* 驱动入口函数 */
  static int __init xxxx_init(void)
  {
  ......
  /* 初始化 tasklet */
  tasklet_init(&testtasklet, testtasklet_func, data);
  /* 注册中断处理函数 */
  request_irq(xxx_irq, test_handler, 0, "xxx", &xxx_dev);
  ......
  }
  ```

## 工作队列

见文档

## 设备树中的中断信息

- 中断控制器

  ```sh
  intc: interrupt-controller@00a01000 {
  	compatible = "arm,cortex-a7-gic";
  	#interrupt-cells = <3>;
  	interrupt-controller;
  	reg = <0x00a01000 0x1000>,
  			<0x00a02000 0x100>;
  };
  ```

  - interrupt-controller;表示当前节点是一个中断控制器
  - #interrupt-cells = <3>;表示设备中的**interrupts**属性中所描绘的中断信息有几个cell，一个cell32位整型，对于ARM的GIC来说有3个cells， 分别是**中断类型，中断号，标志**。
  - **中断类型**， 0 表示 SPI 中断， 1 表示 PPI 中断 
  - **中断号**，对于 SPI 中断来说中断号的范围为 0~987，对于 PPI 中断来说中断号的范围为 0~15。 
  - **标志**： bit[3:0]表示中断触发类型，为 1 的时候表示上升沿触发，为 2 的时候表示下降沿触发，为 4 的时候表示高电平触发，为 8 的时候表示低电平触发。 bit[15:8]为 PPI 中断的 CPU 掩码。

  gpio也可以作为中断节点

  ```sh
  gpio5: gpio@020ac000 {
  	compatible = "fsl,imx6ul-gpio", "fsl,imx35-gpio";
  	reg = <0x020ac000 0x4000>;
  	interrupts = <GIC_SPI 74 IRQ_TYPE_LEVEL_HIGH>, // 定义了该GPIO控制器支持的中断资源
  				<GIC_SPI 75 IRQ_TYPE_LEVEL_HIGH>;
  	gpio-controller;
  	#gpio-cells = <2>;
  	interrupt-controller;
  	#interrupt-cells = <2>;
   };
  # 第 4 行， interrupts 描述中断源信息GPIO5 一共用了 2 个中断号，一个是 74，一个是 75。其中 74 对应 GPIO5_IO00~GPIO5_IO15 这低 16 个 IO， 75 对应 GPIO5_IO16~GPIOI5_IO31 这高 16 位 IO。
  ```

- 设备节点中使用某个中断控制器

  ```c
  fxls8471@1e {
  	compatible = "fsl,fxls8471";
  	reg = <0x1e>;
  	position = <0>;
  	interrupt-parent = <&gpio5>; !!!指定中断控制器/父中断
  	interrupts = <0 8>; // 中断号和触发方式
  };
  ```

  

## 获取中断号

**unsigned int irq_of_parse_and_map(struct device_node int  *dev,index)** 

如果使用 GPIO 的话，可以使用 gpio_to_irq 函数来获取 gpio 对应的中断号， 

**int gpio_to_irq(unsigned int gpio)** 

返回值： GPIO 对应的中断号。 



## 编写程序实现按键消抖

- 中断服务函数用于消抖 进入中断后使用定时器实现10ms的延时

 

- 定时器回调函数 两重确定，中断设置为边沿触发，按下时设置keyvalue为0x01，松开时设置keyvalue |= 0x80，同时一次按下松开的动作之后给releasekey置1，这样一次按键动作后将同时满足keyvalue=0x81和releasekey=1。
- 在内核read函数中判断是否完成了一次按键动作，然后将按键值发送给应用程序。
- 在key_init函数中主要实现key-gpio的申请，中断的申请，定时器的创建。

### 修改设备树

在key节点下添加中断相关属性

```c
key {
		compatible = "alientek,key";	
		pinctrl-names = "default";
		pinctrl-0 = <&pinctrl_key>;
		key-gpio = <&gpio1 18 GPIO_ACTIVE_LOW>;
    	interrupt-parent = <&gpio1>;
    	interrupts = <18 IRQ_TYPE_EDGE_BOTH>;	/* 双边沿触发的中断 */
		status = "okay";
	};
```





































































# 这个专栏用于记录开发过程中Linux服务器和开发板Linux的一些设置



- modprobe 命令主要智能在提供了模块的依赖性分析、错误检查、错误报告等功能，推荐使用 modprobe 命令来加载驱动。modprobe 命令默认会去/lib/modules/<kernel-version>目录中查找模块，比如本书使用的 Linux kernel 的版本号为 4.1.15，因此 modprobe 命令默认会到/lib/modules/4.1.15 这个目录中查找相应的驱动模块，一般自己制作的根文件系统中是不会有这个目录的，所以需要自己手动创建。 
- tftp不能下载zImage文件的情况错误排查
  1. 开发板的IP地址有没有与其他IP地址重复
  2. Linux服务器的IP地址是否与其他IP地址重复