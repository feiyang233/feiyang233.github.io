---
title: 时间求差值timestampdiff
date: 2018-02-23
tags: python
categories: 学习
---

## 问题背景介绍

因为Prof想知道每个building里的average-dwell时间，NUS-WiFi数据库提供了current-time，
first-time。思路就是根据两者时间的差值来估计stay in building的时间。

<!--more-->

![数据时间格式](/img/time1.png)


## 人生苦短，首选python

第一想法当然是用python，用python包：`time`, `datetime`.
代码如下：

```python
import time
import datetime

def difftime(begintime,endtime):
	date1=time.strptime(begintime,'%Y-%m-%d %H:%M:%S') 
	date2=time.strptime(endtime,'%Y-%m-%d %H:%M:%S')
	#y两位数的年份表示（00-99）
	#Y四位数的年份表示（000-9999）
	date3=datetime.datetime(
		date1[0],date1[1],date1[2],date1[3],date1[4],date1[5]) 
	date4=datetime.datetime(
		date2[0],date2[1],date2[2],date2[3],date2[4],date2[5])
	minutes=(date4-date3).seconds/60

	return minutes
```

结果如下图：

![结果展示](/img/time3.png)

## `MySQL` 表示不服

上学期我用python做的统计工作，例如每分钟有多少device，或者每个device在building里停留了多少分钟.
用python至少得写30+行代码去读写csv文件，在MySQL就是一行代码的事情，相见恨晚。
`无知限制了我的生产力！`
MySQL的数据类型里有时间戳格式timestamp。也有专门计算时间差值的`timestampdiff`函数。
TIMESTAMPDIFF(unit,begin,end);返回end-begin的时间差值。

```mysql
select timestampdiff(minute,firstime,curtime)
```

结果如下图：

![结果展示](/img/time2.png)

## INSERT遇到问题了

我想把timestampdiff返回的值插入在表的最后，先创建一个字段（列），再用insert插入，结果遇到了问题。

```mysql
insert into test (diff)  select timestampdiff(minute,firstime,curtime)  from test;
```

![插入问题](/img/issue.png)

如图所示，结果是插在了diff列，但直接插在表的后面，没有覆盖原有的值。经过思考后，我尝试了 update的方法。

```mysql
update test set diff=timestampdiff(minute,firstime,curtime);
```

终于成功了！

![成功了](/img/time4.png)

## 咸鱼的叹息

python定义了一个函数来计算，用MySQL就是一句话的事情。光找这句话我就花了好久，还得好好学习。
还是在项目中体会得多，光看书是不行的，纸上得来终觉浅。

很惭愧，连一点微小的工作都没有做，打扰了。





