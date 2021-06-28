---
layout: post
title: 用樹莓派學嵌入式Linux (2) - 用GPIO控制蜂鳴器
date: 2020-10-04 00:00:00
categories: rpi
---

### 前言

GPIO使用的場景很多, 例如LED燈號控制, 電源開關等等…在嵌入式系統中是一個常見的硬體介面, 網路上搜尋GPIO也有許多介紹, 而最簡單的使用方式大概就是透過/sys/class/gpio的檔案讀寫來控制, 這篇範例我們會用到一個蜂鳴器(Buzzer)與按鈕(Button), 並透過按鈕來控制蜂鳴器, 所以我們開發的驅動程式需要去偵測按下按鈕這個事件(interrupt), 然後利用GPIO來啟動或關閉蜂鳴器, 最後結合前面的教學, 建立一個sysfs檔案來控制蜂鳴器

### 準備
我使用的按鈕與蜂鳴器對應的GPIO Pin為21與23, 可以依照自己的方式修改想使用的GPIO Pin, 只要修改範例程式中BUZZER與BUTTON的define即可, 除了GPIO以外, 還有各一條的接地線, 可以稍微對照一下Raspberry Pi的Pin header

![gpio](/assets/images/20201004/1.png)

我的硬體線路大概長得像這樣

![hw](/assets/images/20201004/2.jpeg)

![hw](/assets/images/20201004/3.jpeg)

另外為了使用Button, 我們還需要將Button佔用的GPIO設為Pull-up, 編輯Raspberry Pi的檔案/boot/config.txt, 加入下面的內容後重新啟動
```sh
gpio=21=pu
```

### GPIO
範例程式如下

```c
#define BUZZER 23
#define BUTTON 21

static short int button_irq = 0;
static unsigned long flags = 0;

int buzzer_trigger = 0;
u32 trigger_jiffies;

static irqreturn_t button_isr(int irq, void *data)
{
	local_irq_save(flags);
	printk("[%s][%d]\n", __func__, __LINE__);
	if((jiffies - trigger_jiffies) > 10)
	{
		trigger_jiffies = jiffies;
		buzzer_trigger = buzzer_trigger ? 0 : 1;
		gpio_set_value(BUZZER, buzzer_trigger);
	}  
	local_irq_restore(flags);
	return IRQ_HANDLED;
}


int init_module(void)
{
	printk("[%s][%d]\n", __func__, __LINE__);
	trigger_jiffies = jiffies;

	if(!gpio_is_valid(BUZZER))
		return -1;
	if(gpio_request(BUZZER, "BUZZER") < 0)
		return -1;

	gpio_direction_output(BUZZER, 0 );

	if(!gpio_is_valid(BUTTON))
		return -1;
	if(gpio_request(BUTTON, "BUTTON") < 0)
		return -1;

	button_irq = gpio_to_irq(BUTTON);
	if(request_irq(button_irq, button_isr ,IRQF_TRIGGER_RISING, NULL, NULL))
		return -1;

	return 0;
}


void cleanup_module(void)
{
	gpio_set_value(BUZZER, 0);
	gpio_free(BUZZER);
	free_irq(button_irq, NULL);
	gpio_free(BUTTON);
}
```

Kernel module的進入點init_module會初始化我們要使用的GPIO Pin(21與23)
* gpio_direction_output將蜂鳴器的direction設為out與狀態low, 可以想成是此module輸出0或1的值來控制蜂鳴器

* 註冊了一個interrupt service routine(isr), 可以想成當硬體的Button按下後會發出一個訊號, 而我們註冊了一個function來處理當訊號發生時所對應的行為

* 當按下按鈕後會進入到到button_isr這個function裡面, buzzer_trigger儲存當下蜂鳴器的狀態, 我們希望如果蜂鳴器靜止的時候按下按鈕要讓它工作, 所以當該值為0時, 將它設為1, 反之設為0, 我們可以透過gpio_set_value設定GPIO23的值來關閉或啟動蜂鳴器

* 除此之外還用到了jiffies來記錄時間差, 讓每次按下按鈕之間有一個時間等待, 這篇先不對jiffies做介紹


所以如果硬體線路正確的情況下, 可以嘗試載入這個kernel module, 按一下按鈕, 蜂鳴器就會啟動, 聽到逼~~~的聲音, 如果再按下按鈕, 蜂鳴器就會停止聲響

### 透過sysfs控制

除了透過實體按鈕來開關蜂鳴器, 我們也可以透過軟體主動來啟動, 在前面我們提到過user-space利用sysfs來讀寫kernel module的變數, 所以我們可以透過簡單的指令來啟動或暫停蜂鳴器, 複製前一份範例程式, 將attr_store寫入的變數改為buzzer_trigger, 並呼叫gpio_set_value來設定GPIO的狀態即可

```c
static ssize_t attr_store(struct device *dev,
		struct device_attribute *attr,
		const char *buf, size_t count)
{

	printk("[%s][%d]\n", __func__, __LINE__);
	buzzer_trigger = simple_strtoul(buf, NULL, 10);
	gpio_set_value(BUZZER, buzzer_trigger);
	return count;
}

static ssize_t attr_show(struct device *dev,
		struct device_attribute *attr,
		char *buf)
{
    int val = 0;
    printk("[%s][%d]\n", __func__, __LINE__);
    val = sprintf(buf, "%d\n", buzzer_trigger);
    return val;
}
```

將GPIO Pin設為High, 啟用蜂鳴器
```sh
echo 1 > /sys/class/mydev_class/dev/data
```

將GPIO Pin設為Low, 關閉蜂鳴器
```sh
echo 0 > /sys/class/mydev_class/dev/data
```

### 完整範例
[Github](https://github.com/sepfy/learn-embedded-linux-with-rpi/tree/master/02-gpio-interrupt)

### 參考資料

<https://www.raspberrypi.org/documentation/configuration/config-txt/gpio.md>
<http://blog.ittraining.com.tw/2015/05/raspberry-pi-b-pi2-linux-gpio-button.html>
