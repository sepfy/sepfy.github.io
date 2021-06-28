---
layout: post
title: 用樹莓派學嵌入式Linux (3) - Kernel module用ioctl交換訊息
date: 2020-10-04 00:00:00
categories: rpi
---

前面介紹過sysfs透過讀寫檔案來交換訊息, 但是如果較於複雜的功能與設定, 這樣的做法可能不太方便, 所以接下來要實做ioctl來達到交換資料與操作的功能

### Misc device
首先我們先透過misc_register註冊一個雜項設備(Miscellaneous Devices), 由於這邊將name定義為misc_dev, 當安裝這支驅動程式後, 會產生/dev/misc_dev裝置節點, 我們可以對該檔案做許多操作, 例如讀取寫入等等…, 在miscdevice結構中有個成員fops變數可以讓我們定義驅動程式該如何處理這些操作

```c
static struct miscdevice my_miscdev = {
	.minor      = 11,
	.name       = "misc_dev",
	.fops       = &mydev_fops,
};

static int __init init_modules(void)
{
	int ret;
	ret = misc_register(&my_miscdev);
	if (ret != 0) {
	printk("cannot register miscdev on minor=11 (err=%d)\n",ret);
	}

	return 0;
}

static void __exit exit_modules(void)
{
	misc_deregister(&my_miscdev);
}
```

### File operations
fops對應的是file_operations結構, 包含的操作有open, release, read, write以及ioctl, 我們需要實現每種操作所對應的function

```c
static ssize_t
mydev_read(struct file *filp, char __user *buf, size_t count, loff_t *pos)
{
	char tmp[] = "Kernel says hello";
	printk("[%s][%d]\n", __func__, __LINE__);
	copy_to_user(buf, tmp, sizeof(tmp));
	*pos = 0;
	return 0;
}

static ssize_t
mydev_write(struct file *filp, const char __user *buf,
        size_t count, loff_t *pos)
{
	char tmp[128] = {0};
	printk("[%s][%d]\n", __func__, __LINE__);
	copy_from_user(tmp, buf, sizeof(tmp));
	printk("%s", tmp);
	return count;
}

static int mydev_open(struct inode *inode, struct file *filp)
{
	printk("[%s][%d]\n", __func__, __LINE__);
	return 0;
}

static int
mydev_release(struct inode *inode, struct file *filp)
{
	printk("[%s][%d]\n", __func__, __LINE__);
	return 0;
}

static struct file_operations 
  mydev_fops = {
	.owner = THIS_MODULE,
	.open = mydev_open,
	.release = mydev_release,
	.read = mydev_read,
	.write = mydev_write,
	.unlocked_ioctl = mydev_ioctl,
};
```

後面會看到應用程式如何執行open, read等指令, 而在read/write的function中, 我們使用了copy_to_user與copy_from_user兩個API, 如同字面上的意思, copy_from_user代表將user-space的記憶體複製到kernel-space, 所以在mydev_write的function中會使用它, 因為write的功能代表將資料寫入, 而kernel-space為了要獲取user-space要寫入的資料, 透過copy_from_user將資料從user-space複製過來, 範例是獲取user-space傳遞的字串並用printk顯示出來, 同樣的, copy_to_user是將kernel-space的資料複製到user-space, 所以用在mydev_read之中, 用來將"Kernel says hello"字串傳遞到user-space

### iotctl
而ioctl可以讓我們自行定義我們想要的功能, 在這個範例中, 我們加入了兩項功能IOCTL_MISCDEV_SET與IOCTL_MISCDEV_GET, 可以在mydev_ioctl的function中看到一個switch的條件用來判別user-space的應用程式傳遞下來的命令, 當然這個命令可以自行定義

```c
struct miscdev_data {
    int val;
    char data[64];
};

#define IOCTL_MISCDEV_SET 0x00
#define IOCTL_MISCDEV_GET 0x01

static long mydev_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{

	struct miscdev_data data;
 
	memset(&data, 0, sizeof(data));
	switch (cmd) {
	case IOCTL_MISCDEV_SET:
		printk("[%s][%d]\n", __func__, __LINE__);
		copy_from_user(&data, (int __user *)arg, sizeof(data));
		printk("Set data: miscdev_data.val = %d, miscdev_data.data = %s\n", data.val, data.data); 
	   	break;
	case IOCTL_MISCDEV_GET:
		printk("[%s][%d]\n", __func__, __LINE__);
		sprintf(data.data, "Kernel says hi");
		data.val = 3;
		copy_to_user((int __user *)arg, &data, sizeof(data));
		break;
	default:
		break;
	}
	return 0;
}
```

前面定義了一個結構miscdev_data, 在ioctl這裡會把這個結構當作內容來傳遞

### User-space

接著我們要撰寫一個user-space的程式來測試我們定義好的功能, 範例程式中我們先透過open建立file descriptor, 前面呼叫了read/write各一次, 再呼叫ioctl, 最後將檔案close

```c
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <fcntl.h>
#include <sys/ioctl.h> // for open
#include <unistd.h> // for close

#define IOCTL_MISCDEV_SET 0x00
#define IOCTL_MISCDEV_GET 0x01

struct miscdev_data {
    int val;
    char data[64];
};

int main(void) {

    int fd, ret;
    char buf[128] = {0};
    struct miscdev_data data;

    fd = open("/dev/misc_dev", O_RDWR);
    if(fd < 0)
        perror("open");


    sprintf(buf, "User says hello\n");
    write(fd, buf, 128);

    read(fd, buf, 128);
    printf("Read data frm kenrel: %s\n", buf);

    memset(&data, 0, sizeof(data));
    data.val = 10;
    sprintf(data.data, "User says hi");

    ret = ioctl(fd, IOCTL_MISCDEV_SET, &data);
    if(ret < 0) {
        perror("ioctl set");
    }

    ret = ioctl(fd, IOCTL_MISCDEV_GET, &data);
    if(ret < 0) {
        perror("ioctl get");
    }
    
    printf("Get data: miscdata_data.val = %d, miscdata_data.data = %s\n", data.val, data.data);

    if(fd > 0)
        close(fd);

    return 0;
}
```

執行應用程式可以讓我們更了解它們之間的關聯, 在範例中的Makefile除了編譯ko檔以外, 還會一起編譯user-space的測試程式
```sh
$ cd learn-embedded-linux-with-rpi/03-misc-ioctl
$ make
$ sudo insmod misc_dev.ko
$ sudo ./sample
```

![userspace](/assets/images/20201004/4.png)

執行程式會看到一些訊息, 第一行"Kernel says hello"對應了kernel module中的mydev_read, 當應用程式執行read時, kernel module將字串透過copy_to_user傳遞給user-space. 第二行則是顯示了miscdev_data結構中的成員, 這是ioctl(fd, IOCTL_MISCDEV_GET, &data)這段代碼得到的結果, 使用ioctl並指定IOCTL_MISCDEV_GET這個命令, kernel module同樣透過copy_to_user將資料複製到data變數中, 再來看看kernel產生的訊息
```sh
$ dmesg
```

![dmesg](/assets/images/20201004/5.png)

這次是在write與iotctl(IOCTL_MISCDEV_SET)對應的函式中調用copy_from_user將資料複製後透過printk顯示出來, 除此之外還有open/close對應的mydev_open與mydev_release, 比起sysfs, ioctl提供了更有彈性的方式來定義驅動程式的功能讓應用程式使用

### 完整範例

[Github](https://github.com/sepfy/learn-embedded-linux-with-rpi/tree/master/03-misc-ioctl)
