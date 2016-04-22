---
title: 计算Plank分组成绩的python小程序
date: 2016-04-21 21:08:19
categories: python
tags: python
---

##计算Plank分组成绩的小程序
为了复习一下最近的学习成果，又恰巧公司每周二都会组织Plank比赛，所以就想写个小程序来完成比赛成绩的计算。

```
#!/usr/bin/env python
# -*- coding:utf-8 -*-

str1 = "请输入分组总人数:\n"
str2 = "请输入实际出勤人数:\n"
GroupUsers = raw_input(str1)
InputUsers = raw_input(str2)
# 根据规则设定的算法类,最终成绩=(总成绩/出勤人数) * (出勤人数/分组人数)
##这里定义了一个 计算类，分别由计算总成绩、出勤率、平均成绩、最终成绩的方法。
class algorithm:
    def __init__(self,iusers,gusers):
        self.iusers = iusers
        self.gusers = gusers
    # 总成绩  使用while循环来完成参赛队员的成绩录入。
    def tsum(self):
        i = int(1)
        s = float(0)
        while i <= int(self.iusers):
            Perach = raw_input("请输入对人员 %d 成绩: \n" % (i))
            s += float(Perach)
            i += 1
        return s
    #出勤率
    def attendance(self):
        att = float(self.iusers)/float(self.gusers)
        return att
    #平均成绩
    def avg(self):
     #   tsums = algorithm(InputUsers,GroupUsers)
        tsums1 = tsums.tsum()
        avg = tsums1/float(self.iusers)
        return avg
    #最终成绩
    def sumresult(self):
     #   tsums = algorithm(InputUsers,GroupUsers)
        avg = tsums.avg()
        att = tsums.attendance()
        sumresult = avg * att
        return sumresult
#类实例
tsums = algorithm(InputUsers,GroupUsers)
tsum = float(tsums.tsum())
attendance = float(InputUsers)/float(GroupUsers)
avg = tsum / float(InputUsers)
sum = attendance * avg
print  "本队最终成绩: %f \n 总成绩: %f\n 出勤率: %f \n 平均成绩: %f\n" % (sum,tsum,attendance,avg)
```
执行结果如下：
```
请输入分组总人数:
10
请输入实际出勤人数:
5
请输入对人员 1 成绩: 
100
请输入对人员 2 成绩: 
200
请输入对人员 3 成绩: 
300
请输入对人员 4 成绩: 
150
请输入对人员 5 成绩: 
210
本队最终成绩: 96.000000 
 总成绩: 960.000000
 出勤率: 0.500000 
 平均成绩: 192.000000
```
这个小程序只是为了实现计算成绩的功能、复习类的实现等学习小结，没有做更严谨的判断、异常处理等。


