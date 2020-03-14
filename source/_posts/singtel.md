---
title: singtel 路由器踩坑经历
date: 2019-07-27 10:00:55
tags: network
categories: 生活
---
2019-07-26 搬新家后贫困山区终于通网了。然而当我打算修改路由器的名称(SSID)和密码的时候，却发现被坑了。
<!--more-->
# 背景介绍
singtel 光纤宽带，有一个光模转换器，还有一个 WiFi 路由器。如下图所示：  
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20190727100707.jpg)  
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20190727100704.jpg)  
# 踩坑过程
当家里有网后，我想修改一下 WiFi 名称和密码，谷歌一下，[答案如下](https://www.singtel.com/personal/support/broadband/routers-ont/arcadyan-ac-plus-guide/change-wireless-settings)。
Step 1  
Visit http://192.168.1.254 to view your router configuration page.  

Step 2  
Scroll down to Device Info and Internet Connection to find the information.   

Step 3  
By default, you will see the 2.4GHz and 5GHz wireless settings, Band Steering and WPS feature..  

Step 4  
To apply any changes made to your Wireless settings, click on the Apply button or cancel to disregard the changes.  
  
我按照以上说明操作一下，结果碰到的却是如下的画面：  
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20190727101742.png)  
当时就懵逼了，账号和密码都不知道。结果看到右上角有一个型号 hg8244h, 谷歌一下 hg8244h default password. 结果如下  
The default network address is 192.168.1.254 and so within a browser connect to http://192.168.1.254. The root user login's default password is admin and should be changed.

登录进去之后，画面如下:   
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20190727100710.png)  
我仔仔细细找了一圈，都没有发现修改 SSID 的地方。当我点开 Lan devices, 列表里发现了一个 Singtel-ACPlus, 我才意识到我登录到光模装换器的控制台了。在这里我看到了 Singtel-ACPlus 的 mac 地址。我在自己的苹果电脑命令行查询一下：
```bash
arp -a
```
得到的网关 192.168.1.254 mac 地址与 Singtel-ACPlus 的 mac 地址不符合

我的电脑连接的是 WiFi 路由器，结果网关却是光模路由器。其中必有蹊跷！
# 解决的办法
先将光模路由器和 WiFi 路由器之间的网线断开，（一定要）再重启WiFi 路由器。等待一段时间连上 WiFi, 再次访问 http://192.168.1.254. 这次终于大功告成！  
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20190727100713.png)  
猜测根本原因是：两个路由器之间是桥接模（也许有错）。 WiFi 路由器连上光模路由器了，都没有自己的 IP。 真是卑微，就像我这种菜菜子，不配拥有姓名。
