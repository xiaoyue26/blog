---
title: 快速查看大量gc日志的gui工具
date: 2018-12-11 09:33:53
tags:
- java
- jvm 
categories:
- java
- jvm 

---

之前学习了GC日志的格式，能理解每行日志的大致含义。
然而对于实际生产项目，日志量庞大，逐行看很低效，这个时候借助一下gui工具就很nice了。

经过海淘了谷歌，比较后留下这2个比较好用的工具：
1. GCViewer,
2. http://gceasy.io

被抛弃的工具:
1. GCHisto: 界面好看,功能有点弱;
2. visualgc(jvisualvm的插件): jvisualvm已经1.8,visualgc版本才1.4,完全安装不上，应该已经不维护了。

# GCViewer
https://sourceforge.net/projects/gcviewer/
缺点: 字超小
优点: 功能全

## 查看堆空间的变化
{% img /images/2018-12/gcviewer1.png 800 1200 gcviewer1 %}
view菜单栏可以看到各种颜色的线的含义，例如蓝色线是堆空间的使用。如果全打开的话眼睛会花的，可以逐步打开几个关心的。

像下面这种情况图中黑色柱状较多的时候,可以看出堆空间不足了,gc时间占用较多:
{% img /images/2018-12/danger.png 800 1200 danger %}


## gc回收的各项统计数据
{% img /images/2018-12/gcviewer4.png 800 1200 gcviewer4 %}


如果搭配一个放大镜，`GCViewer`就是一个非常好的gc日志查看工具了。
{% img /images/2018-12/gcviewer.jpg 800 1200 gcviewer %}


# http://gceasy.io
上传日志到网站。
优点：
1. 有诊断功能,自动检测问题;
2. 界面很美观。

## 诊断功能
它能帮忙检测gc日志中反映出的问题:
{% img /images/2018-12/detect.png 800 1200 detect %}
{% img /images/2018-12/gceasy6.png 800 1200 gceasy6 %}

比如能检测到fullgc:
{% img /images/2018-12/fullgc.png 800 1200 fullgc %}
有些还能针对性地给出建议:
{% img /images/2018-12/tips.png 800 1200 tips %}

## 统计信息
它提供了各个角度的统计信息。
新生代、老生代的峰值使用大小:
{% img /images/2018-12/gceasy0.png 800 1200 gceasy0 %}
各种统计图表:
{% img /images/2018-12/gceasy1.png 800 1200 gceasy1 %}
{% img /images/2018-12/gceasy2.png 800 1200 gceasy2 %}
{% img /images/2018-12/gceasy4.png 800 1200 gceasy4 %}
{% img /images/2018-12/gceasy5.png 800 1200 gceasy5 %}
{% img /images/2018-12/gceasy7.png 800 1200 gceasy7 %}



