---
layout: post
title: "ethtool with fdir"
date: 2013-12-06 20:09
comments: true
categories: nic
---

之前比较奇怪,既然intel的82599以及提供了对flow derector等功能的编程接口,为什么没有能直接进行配置的工具.然后自己再网上搜了下,想想,这东西,通用点的话,肯定就通过ethtool来配置了.
<!-- more -->
网上搜了下,还真有.新版的ethtool增加了新的参数 -U -u来对网卡的这些特性进行查看和配置.具体的配置可以直接通过man来进行查看.so easy.

ps:今天一个coding了几年glibc的码农,说了句,"不想什么都用写代码的方式来解决".资深码农都有这思想了,想想,工作到现在,最大的收获就是,拿来主义..