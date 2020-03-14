---
title: 排列组合了解一下(阿里笔试题改编)
date: 2018-03-04
categories: 学习
tags: python
---
## 问题介绍

假设有小费和小王俩好基友，天天形影不离，二人分在一个组。
为了减少迟到的现象，班主任规定如下：
1. 迟到一次，则当事人记一分。（迟到）
2. 小组两人在某一天同时迟到，则小组总分再记一分。（连坐）
3. 一周内如果每天都迟到，则当事人再扣一分。（太皮了）

<!--more-->
  三种情况可累加记分，一周结算一次小组总分。
  肯定有人想说
![gay里gay气](/img/gay.jpeg) 

## 举个例子好了
下图为某一周的出勤情况：从图中看出小费同学比较皮，天天迟到！小王同学有两天和小费同学一起开黑，导致了迟到。
所以总的小组计分为10分！

![出勤情况](/img/combination1.png)

`那么问题来了：如果已知某一周小组的总分情况，列出可能的出勤表`

## 咸鱼采用的暴力破解法
有两个对象，每个对象有5个记录。每次记录的值有两种情况，我们可以暴力枚举出所有的情况。
一共也才2的10次方，1024种情况。对于计算机来说，当然是小菜一碟咯。

所有上面的出勤图就可以简化为0-1数值图：
![数值图](/img/combination2.png)

接下来问题就变成，如何枚举出这1024种情况？
1. 2行5列一共10个值，循环列出0-1111111111二进制数的情况，高位补0。
2. 如上图，2列是独立的，我们只需求出5位的情况，两列互相遍历即可。
3. 用python自带的 [itertools](https://docs.python.org/3/library/itertools.html)库，分分钟枚举完成。

![解释图](/img/combination3.png)

```python
import itertools

def main(start_day,faith_score):
    Calendar=[[6,6,6,6,6],[6,6,6,6,6]] #模拟日历2行5列
    L=list(itertools.product([0,1],repeat=5)) #枚举产生
    leng=len(L)
    for i in range(leng):#两次遍历
        for j in range(leng):
            Calendar[0]=L[i]
            Calendar[1]=L[j]
            if sumcredit(Calendar)==faith_score:#判断情况
                gg=todate(Calendar,start_day)
                print(gg) #输出情况
```

只要有了枚举的所有情况，判断计分则是很容易的事情。

```python
def credit1(L): #迟到一次扣一分
    credit=0
    for i in L[0]:
        if i==1:
            credit+=1
    for j in L[1]:
        if j==1:
            credit+=1
    return credit

def credit2 (L):  #一起迟到，多扣一分
    credit=0
    for i in range(5):
        if L[0][i]==1 and L[1][i]==1:
            credit+=1
    return credit

def credit3 (L): #连续迟到，罪加一等
    credit=0
    if 0 not in L[0]:
        credit+=1
    if 0 not in L[1]:
        credit+=1
    return credit

def sumcredit(L): #累加总分
    c1=credit1(L)
    c2=credit2(L)
    c3=credit3(L)
    return c1+c2+c3
```

## 时间问题，不容小觑
因为时间在`跨月份`是时候有点麻烦，不能用普通的++，但我们可以用`datetime`库呀。
输入一个起始时间，我们要得到那一周的5天。`timedelta()`函数你值得拥有。

```python
from datetime import datetime,timedelta

def get_date(date):       #Y表示四位数的年份(0-9999)
    date_begin=datetime.strptime(date,'%Y%m%d') 
    time=str(datetime.date(date_begin))

    date1=date_begin+timedelta(days=1) #加1s，哈哈
    time1=str(datetime.date(date1))

    date2=date_begin+timedelta(days=2)
    time2=str(datetime.date(date2))

    date3=date_begin+timedelta(days=2)
    time3=str(datetime.date(date3))

    date4=date_begin+timedelta(days=2)
    time4=str(datetime.date(date4))

    return time,time1,time2,time3,time4
```

最后我们在输出 出勤情况的时候，将那个简化后的二维数组List`=>`日历对应起来。

```python
def todate(L,date):
    time,time1,time2,time3,time4=get_date(date)
    record=[time+'-小费',time+'-小王',time1+'-小费',time1+'-小王',time2+'-小费',\
    time2+'-小王',time3+'-小费',time3+'-小王',time4+'-小费',time4+'-小王']
    L1=L[0]+L[1]
    for i in range(len(L1)):
        if L1[i]==1: #1 代表迟到
            record[i]+='(迟到)'
        else: #0代表没迟到
            record[i]+='(正常)'
    return record
```

假设我们输入`date=20180305,score=0`
输出如下结果：
```python
['2018-03-04-小费(正常)', '2018-03-04-小王(正常)',
 '2018-03-05-小费(正常)', '2018-03-05-小王(正常)', 
 '2018-03-06-小费(正常)', '2018-03-06-小王(正常)', 
 '2018-03-06-小费(正常)', '2018-03-06-小王(正常)', 
 '2018-03-06-小费(正常)', '2018-03-06-小王(正常)']
[Finished in 0.2s]
```

假设我们输入`date=20180305,score=1`
输出如下结果：
![结果](/img/combination4.png)

## 最后一点废话
有一篇关于itertools的文章[这段代码很Pythonic | 相见恨晚的 itertools 库](https://mp.weixin.qq.com/s/Rb5aYWA7NYOi1eckGtakuQ) 安利一下。此文章的python[源代码](https://github.com/fainyang/LeetCode_practice/blob/master/score.py)

马上就要毕业了，再也没有当学生那种无忧无虑的感觉了。以前贪玩欠的债，找工作的时候哭出来。
去面试才知道自己有多菜，所幸我还比较`naive`，未来珍惜时间，好好学习。狗住，加油！






