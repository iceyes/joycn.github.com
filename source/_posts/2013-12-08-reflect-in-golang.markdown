---
layout: post
title: "reflect in golang"
date: 2013-12-08 20:17
comments: true
categories: golang
---

golang中的reflect主要用途就是从interface类型中得到原本的数据类型,以及原本变量的内容.对于interface来,本质上就是就是一个(value, type)的组合.value是复制过来的变量名,type是底层的数据类型.说起来数据类型.type myint int与int虽然底层的数据类型是一样的,比如我们可以通过var i myint; i = 1对i进行赋值,我们却不能通过 var i myint; j := 1; i = j的方式来赋值.编译会报错.使用TypeOf的时候也会分别返回myint和int,但是如果对返回的Type类型再使用Kind()方法,那么都会返回int类型.

对于reflect主要用到的两个方法是TypeOf()和ValueOf(),分别获取interface对应的type信息和value信息.如果要修改interface对应的struct的内容,需要将interface的指针而不是值传递给ValueOf方法