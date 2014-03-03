---
layout: post
title: "在使用pktgen如何利用网卡多队列功能"
date: 2013-03-15 22:16
comments: true
categories: kernel
---
之前奇怪,为什么pktgen明明有QUEUE_MAP_CPU的flags,为什么使用起来还是单核的效果.今天同事说起来,说自己可以用多核.然后我也就跟着试了下.原来还真可以.只是pktgen的samples并没有提到这个(或许我看的比较老)
<!-- more -->
简单说下原理,涉及到网卡发送队列的一点点内容.当pktgen的脚本设置了QUEUE_MAP_CPU这个flag后,会在构造包的时候,将当前cpu的id绑定到skb的mapqueue里,这样就实现了多队列的效果.我们在insmod的时候,pktgen会根据cpu的个数来创建同等数量的线程,主要还是因为数据发送后的一些工作比较消耗CPU,这样我们绑定后,就将tx_action分配到了本地的cpu上.
原来我们可以通过dev_name@XX的形式来为通一个设备,加同一个设备添加到多个thread上去.比如,我们有一个eth0的多队列网卡,我们可以通过pgset "add_device eth0@1"的形式,将eth0添加到某个thread上去.所以我们就可以用下面的方式将同一个网卡绑定给多个thread.

{% codeblock %}
for ((id=0;processor<$CPUS;id++))
do
PGDEV=/proc/net/pktgen/kpktgend_$id 
pgset "add_device eth0@$processor"
done
{% endcodeblock %}

这样我们就可以利用网卡的队列,利用多核来发包了.双路4核,关闭超线程的cpu用8599发包,差不多速率在12Mpps, 去年这时候的时候跟intel的人聊过,说82599差不多利用4个队列就能完成9M pps的发包速率,今天试了下,还真是,9+Mpps的速率.

之前只改了自己一部分功能方面的代码,今天因为这个问题,看了下源码,里面的代码写的还是不错的.改天可以再去仔细的看下,毕竟以后估计要通过他来完成更多的工作 	