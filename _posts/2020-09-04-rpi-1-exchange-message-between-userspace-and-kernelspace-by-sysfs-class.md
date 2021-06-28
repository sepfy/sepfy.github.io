---
layout: post
title: 用樹莓派學嵌入式Linux (1) - 利用sysfs class與用戶空間交換訊息
date: 2020-09-04
categories: rpi
---

工作上時常接觸Embedded Linux程式開發, 所以藉由Raspberry Pi來記錄一些平常遇到的功能.
![arch](/assets/images/20200904/1.png)

開發驅動程式不外乎是為了控制硬體, 但是在整個產品系統中, 還是由user space的應用程式來控制產品的功能, 例如在網頁介面按下按鈕後裝置的LED要亮燈等等…所以由kernel space處理的功能, 如果能提供介面讓應用程式操作, 才能整合出完整的產品, 而sysfs是其中的一個方法.

###開發環境

最簡單的開發方式, 即是直接在Raspberry Pi OS上開發, 透過Raspberry Pi網站上的教學下載最新的image並燒入至SD後開機即可. 因為我們直接在樹莓派上開發驅動, 需要kernel的header檔才可以編譯, 可以透過以下指令來安裝kernel header.

```sh
sudo apt-get install -y raspberrypi-kernel-headers
```

安裝完後可在/usr/src路徑下找到當前kernel 版本的header.
第一支Kernel module
接著我們來撰寫一支能讓user space讀寫變數的驅動程式, 因為程式碼不多, 所以我將完整的程式碼貼上來

```c
#include <linux/module.h>
#include <linux/device.h>
#include <linux/err.h>

int value = 0;

static ssize_t attr_store(struct device *dev,
		struct device_attribute *attr,
		const char *buf, size_t count)
{

    value = simple_strtoul(buf, NULL, 10);
    printk("value:%d\n", value);
    return count;
}

static ssize_t attr_show(struct device *dev,
		struct device_attribute *attr,
		char *buf)
{
        int val=0;
        val=sprintf(buf, "%d\n",value);
        printk("%d\n", val);
        return val;
}

static struct class *mydev_class;
static struct device *dev;

static DEVICE_ATTR(data, 0644, attr_show, attr_store);

static int __init init_mydev(void)
{
	int ret = 0;
	printk("Init my dev\n");
	mydev_class = class_create(THIS_MODULE, "mydev_class");
	if(IS_ERR(mydev_class)) {
		ret = PTR_ERR(mydev_class);
		printk(KERN_ALERT "Failed to create class.\n");
		return ret;
	}

	dev = device_create(mydev_class, NULL, MKDEV(100,0), NULL, "dev");
	if(IS_ERR(dev)) {
		ret = PTR_ERR(dev);
		printk(KERN_ALERT "Failed to create device.\n");
		return ret;
	}

	ret = device_create_file(dev, &dev_attr_data);
	if(ret < 0) {
		printk(KERN_ALERT "Failed to create attribute file.\n");
		return ret;
	}
	return ret;
}


static void __exit cleanup_mydev(void)
{
	printk("Cleanup my dev\n");
	device_remove_file(dev, &dev_attr_data);
	device_destroy(mydev_class, MKDEV(100, 0));
	class_destroy(mydev_class);
}


module_init(init_mydev);
module_exit(cleanup_mydev);

MODULE_LICENSE("GPL");
```

除了kernel module最基本的init與exit函式外, 我們透過device_create_file在/sys/class下建立一個能在user space讀寫的檔案, attr_store與attr_show是針對該檔案被讀寫時所對應的函式, 範例只是將寫入的值儲存在變數value當中, 讀取時將該變數讀出來而已, 這提供了一個簡單的方式讓我們可以去設定與讀取kernel module的變數.
編譯
基本上Linux driver都是由Makefile來進行建置編譯的, 以下是一個最基本的Makefile, 執行Make指令後會產生出一支mydev.ko的檔案

```makefile
PWD := $(bash pwd) 
KVERSION := $(bash uname -r)
KERNEL_DIR = /usr/src/linux-headers-$(KVERSION)/

MODULE_NAME = mydev
obj-m := $(MODULE_NAME).o

all:
	make -C $(KERNEL_DIR) M=$(PWD) modules
clean:
	make -C $(KERNEL_DIR) M=$(PWD) clean
```

### 安裝

我們可以執行insmod指令將編譯出來的kernel module安裝到kernel中, 再用dmesg查看printk產生出來的訊息

```sh
sudo insmod mydev.ko
dmesg
```

![insmod](/assets/images/20200904/2.png)

使用lsmod可以查看目前加入的kernel module, 如果想移除該module, 執行rmmod
```sh
sudo lsmod | grep mydev
sudo rmmod mydev
```
![rmmod](/assets/images/20200904/3.png)

### 測試
安裝完成後, 在/sys/class路徑下會產生mydev_class的目錄, 我們可以對該目錄底下的檔案做讀寫, 達到與kernel space交換資料的功能
```sh
# Change to root
sudo su
# Write
echo 3 > /sys/class/mydev_class/dev/data
# Read
cat /sys/class/mydev_class/dev/data
```
![test](/assets/images/20200904/4.png)

當然也可以使用其他程式語言, 如python, golang…的檔案讀寫功能, 下一篇我們會實做一個感測器的驅動程式, 讓我們的應用程式獲取資料.

### 完整範例
[Github](https://github.com/sepfy/learn-embedded-linux-with-rpi/tree/master/01-sysfs)

