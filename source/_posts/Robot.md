---
title: 小白也会用的微信机器人
date: 2018-03-09
tags: python
---
## 天下武功，唯快不破
这一篇文章是讲解如何安装微信自动回复机器人，独乐乐不如众乐乐。
`画风突转，先来一段鸡汤：`并不是每个人都值得你腾出手中的事情秒回，如果有那么一个人，让你情不自禁地秒回了，说明那个人何其重要。
或者说，你被别人秒回，又是多么温暖的存在啊。
他秒回的瞬间，仿佛在说：你在我心中，很重要。
<!--more-->


![秒回](/img/robot/miaohui.jpg)

> 只可惜我只是一个机器人，惊不惊喜，意不意外？

废话不多说，先来看一看效果图：

![举例子图片1](/img/robot/result1.jpeg) 

![举例子图片2](/img/robot/result2.jpeg)

![举例子图片3](/img/robot/result3.jpeg)

看完是不是有点小激动呢？

![激动的搓搓小手](/img/robot/qidai.jpeg)

老司机通道：`会python的直接往下拉--看运行机器人程序`

## 第一步安装python环境
俗话说，人生苦短，我用python。安装python就像安装游戏、QQ一样，去[官网](https://www.python.org/downloads/)下载安装包，只是安装后桌面没有运行图标，而是在我们的计算机上安装了一个运行环境。

看[Windows-python安装教程](https://www.jianshu.com/p/7a0b52075f70)
如果不明白的话，再看看[视频教程](https://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/0014316090478912dab2a3a9e8f4ed49d28854b292f85bb000)。

用苹果电脑的点击观看[苹果电脑python安装教程](https://www.jianshu.com/p/a10402b48a1c)。
如果看不懂，可以去视频网站`搜python安装教学视频`，一边看视频一边依葫芦画瓢，我在这里就不详细说明了。万事开头难，搞不懂的话你可以微信问我呀，哈哈。

安装好以后，在Windows命令行 or Mac的终端检查一下python的version版本

![检查](/img/robot/python.png)


## 安装python包
当你把python环境安装好了以后，接下来你需要安装`python包`（python包是武林高手们打造的神兵利器，从新手村刚刚出来的你再也不用苦苦磨炼，自己去做武器啦）

![超开心耶](/img/robot/happy.jpg)

那么问题来了，你怎么样从大侠手中借到神兵利器？？？
这个时候就需要我们常说的py交易（开玩笑的啦），肯定是用[pip](https://www.jianshu.com/p/cf420956e9c1)
。此外，这个机器人自动回复，需要两个包：一个是requests，另一个是itchat。

在Windows命令行 or Mac的终端 输入以下命令
```python
pip install requests

pip install itchat
```
当用pip借到大佬们的神兵利器后，终于可以开始打怪升级了，美滋滋。

![低头](/img/robot/dalao.png)

## 运行机器人程序

第一步需要点击下载[源代码](https://codeload.github.com/fainyang/Test/zip/master) 下载时存放的`地点`要记住。
下载之后解压文件`Test-master.zip`，其中的`tuling.py`就是我们的微信自动回复机器人小程序 。
在文件夹中找到`tuling.py`,右键=>打开方式用`记事本`打开，其中有两个地方需要修改。

第一个是图灵机器人的`apikey`，首先点击打开[图灵机器人网站](http://www.tuling123.com/)注册一个账号，免费创建一个机器人，得到机器人的`apikey`

![tuling](/img/robot/tuling.png)

第二个地方就是：

```python
itchat.auto_login(enableCmdQR=2) 
```
修改为如下形式：这样是为了弹窗形式二维码比命令行形式的二维码清晰
```python
itchat.auto_login() 
```

至此，我们得到了机器人`apikey`，修改了二维码形式，就可以运行`tuling.py`了。

（最简单的小白法） Mac的终端 输入以下命令：
```python
cd  xxxx/Test-master   #注释：先移动到tuling.py 所在文件
python tuling.py      #释：运行tuling.py
```
![run](/img/robot/run.png)
Windows用户在`Test-master`文件夹打开命令行，方法为：在此文件夹窗口内空白区域`右键单击（需要同时按住Shift）`，从菜单中选择＂在此处打开命令行窗口＂的项。

然后在命令窗口输入
```python
python tuling.py 
```

如果前面python包成功安装，运行将会弹出来一个网页登录的二维码。微信扫码登录后，看到命令窗口的个人信息就意味着大功告成了。
```python
Getting uuid of QR code.
Downloading QR code.
Please scan the QR code to log in.
Please press confirm on your phone.
Loading the contact, this may take a little while.
TERM environment variable not set.
Login successfully as Fain 费洋
Start auto replying. 
```

但是这个机器人有一个缺陷：一旦运行python的电脑断网、休眠、关机。那么自动回复将会失效，因为自动回复的核心是`电脑`运行python调用图灵机器人来进行回复。但是对于小白来讲，这就已经足够~(≧▽≦)~啦啦啦。

## 兽（瘦）人永不为奴，云服务器永不关机
我这周刚买的[腾讯云](https://cloud.tencent.com/act/campus)服务器，学生优惠下来3年360，超级划算的。
![tencent](/img/robot/tencent.png)

我选择的系统是CentOS7.4（自带python2.7），需要安装python3.6--请参考[教程1](http://www.jb51.net/article/133560.htm) 和[教程2](https://cloud.tencent.com/developer/article/1017880) 

安装好python环境以后，也需要安装两个包，一个是`requests`，另一个是`itchat`。
然后是在本地用`scp`把`tuling.py` 上传到云服务器，方法请参考[这里](https://cloud.tencent.com/document/product/213/2133)。
```python
scp 本地文件地址 云服务器登录名@云服务器公网IP/域名:云服务器文件地址
```
上传完成以后，运行试一试？结果你会发现当我们远程ssh关闭的时候，云服务器中的python程序也关闭了。
为了能长时间运行，你需要使用[Linux 技巧](https://www.ibm.com/developerworks/cn/linux/l-cn-nohup/)来续几年。

```python
setsid python tuling.py 
```

这样我们的机器人就能在云服务器一直运行啦，直到天荒地老，海枯石烂？

错！是服务器到期欠费。

![poor](/img/robot/poor.jpeg)



