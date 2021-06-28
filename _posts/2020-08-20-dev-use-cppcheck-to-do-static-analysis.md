---
layout: post
title: 程式開發 - 使用Cppcheck做靜態程式碼分析
date: 2020-08-20
categories: Programming
---

第一次聽到靜態程式碼分析是當時任職的公司要幫一家知名的外商公司改寫系統程式的時候, 客戶使用靜態程式碼分析軟體檢查我修改的程式碼後, 要求針對這些問題做修改, 其目的就是透過分析靜態的程式碼檔案, 看看程式中是否有存在的漏洞或者是可以優化的地方, 算是程式品質管控的好方法, 但身為一個興趣使然的軟體工程師, 沒錢去購買付費的分析工具, 於是嘗試使用了開源分析工具Cppcheck

[![cppcheck](https://dashboard.snapcraft.io/site_media/appmedia/2018/01/cppcheck-gui.png)](http://cppcheck.sourceforge.net/)

# 安裝

如果是在Ubuntu的環境下開發, 可以直接利用apt來安裝
```sh
$ apt install cppcheck
```

# 使用方式

使用方法也很簡單, 直接指定想要分析的檔案, 或者可以跟後面的範例一樣, 分析當前目錄下所有的程式碼
```sh
$ cppcheck --enable=all --suppress=missingIncludeSystem <c file>
```
* -\-enable=all: 代表檢查所有項目
* -\-suppress=missingIncludeSystem: 不加的話會有找不到一些include檔的訊息, 所以參考網路上的討論加入了這項

# 範例
自己之前寫了一個[message queue](https://github.com/sepfy/message-queue)的小專案, 用cppcheck來檢查一下, 發現還挺多需要改進的地方的, 例如memory leak, unused variable等等
![check](/assets/images/20200820/1.png)

除了cppcheck, 我也有使用過facebook infer, 但是不知道為何infer沒有幫我檢查出以上問題, 所以就決定使用cppcheck了, 希望這個工具能協助我們打造出更優質的程式碼
