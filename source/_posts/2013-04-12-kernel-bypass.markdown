---
layout: post
title: "kernel_bypass"
date: 2013-04-12 08:47
comments: true
categories: kernel networking
---

现在10Gbe网卡现在已经比较通用了,目前40Gbe的网卡也已经出来一段时间了.随着网卡的不断发展,系统网络栈的设计,越来越成为数据包处理的瓶颈.这不仅有系统的问题,也跟特定的应用有关,毕竟有的应用用不到太多内核中相关的过于复杂的设计.现在kernel_bypass也变得越来越流行.主流的kernel_bypass的方案有下面这么几个:
<!-- more -->

- [ntop.org DNA](http://www.ntop.org/products/pf_ring/dna/).
- [netmap.](http://info.iet.unipi.it/~luigi/netmap/)
- [Intel DPDK.](http://www.intel.com/content/www/us/en/intelligent-systems/intel-technology/packet-processing-is-enhanced-with-software-from-intel-dpdk.html)
- Myricom [Sniffer10G](https://www.myricom.com/software/sniffer10g.html) and [DBL](https://www.myricom.com/software/dbl.html).
- SolarFlare [OpenOnload](http://www.openonload.org/).
- [Napatech](http://www.napatech.com/products/network_adapters.html).

Myricom和Napatech都要用到自己特定的硬件,DNA又需要license.Napatech又需要DNA,又需要硬件.netmap是完全开源的,intel DPDK现在也开源了(网址就是www.dpdk.org).之前看过netmap的介绍,自己也简单试了下,效果没有想象的那么好,而且大部分也修改的驱动也是intel网卡的驱动.既然DPDK已经开源了,就先了解下DPDK方面的东西了.

## 总结下DPDK是如何去实现80Mpps的目标的 ##
正常模式下存在的问题

- 对于大量的网卡中断,系统已经跟不上中断的速度.
- 对于linux进程切换的消耗.
- CPU的速度与PCIe速度之间的差距
- cache与访存的问题
- 数据共享的问题
- 页表的频繁更新

对于大量中断的问题,linux原本已经做了一部分优化,就是napi,按理说应该已经可以很大的减少硬件中断带来的性能损耗.但是,并没有减少硬件中断的次数.在napi的逻辑里,在网卡中断的处理逻辑是,检查该网卡是否已经有需要处理的数据包,如果有的话,不需要再做后续的处理,napi会在处理之前的数据包的时候,把现在的数据包也一起处理了,从而减少了硬中断的工作量.但是,中断的频率并没有改变.DPDK的方法其实,就是取消中断,靠系统,去主动的polling网卡,减少中断的开销.这样,虽然在网卡闲的时候,额外功会比较多,但是在网卡忙的时候,有效功就很高了.

对于进程切换的问题,绑定逻辑进程到固定的cpu,减少进程间的切换带来的消耗.由于处理数据包大部分的工作都交给用户态进程来做.所以,kernel<->userspace之间的切换也少了很多.而且还减少了,进程在不通cpu上切换,带来的numa相关的问题

对于PCIe速度的差异,以及访问内存的差异.跟第一点也有一定的关系,其实通过polling,一次取数据的时候,尽可能多的取数据,减少访存的次数.对于访问内存的差异,可以通过pre-fetch的方式预取,PCIe的话,可以直接通过DDIO将数据包放到L3cache中.

数据共享的问题,其实还是说,逻辑放到了用户态,用户可以自己去实现逻辑,通过合理的方式,减少为了保证数据一致性,使用同步机制带来的性能损耗

页表的更新..这个,不太熟,大概还是因为说,intel通过内核huge page的方式,尽可能的减少页表的更新.后面了解下这里

