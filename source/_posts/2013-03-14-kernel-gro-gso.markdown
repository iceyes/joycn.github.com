---
layout: post
title: "kernel中关于gso gro的一些内容"
date: 2013-03-14 22:42
comments: true
categories: kernel
---

之前在lvs利用网卡gro gso功能的时候遇到了一个bug,所以就想去了解下kernel关于gro gso的实现.简单介绍网上有一大堆,这里简单介绍下
<!-- more -->
## offload ##
offload特性都是为了提升网络收/发性能。TSO、UFO和GSO是对应网络发送，在接收方向上对应的是LRO、GRO。

TSO
TSO(TCP Segmentation Offload)，是一种利用网卡对TCP数据包分片，减轻CPU负荷的一种技术，有时也被叫做 LSO (Large segment offload) ，TSO是针对TCP的，UFO是针对UDP的。如果硬件支持 TSO功能，同时也需要硬件支持的TCP校验计算和分散/聚集 (Scatter Gather) 功能。

GSO
GSO(Generic Segmentation Offload)，它比TSO更通用，基本思想就是尽可能的推迟数据分片直至发送到网卡驱动之前，此时会检查网卡是否支持分片功能（如TSO、UFO）, 如果支持直接发送到网卡，如果不支持就进行分片后再发往网卡。这样大数据包只需走一次协议栈，而不是被分割成几个数据包分别走，这就提高了效率。

LRO
LRO(Large Receive Offload)，通过将接收到的多个TCP数据聚合成一个大的数据包，然后传递给网络协议栈处理，以减少上层协议栈处理 开销，提高系统接收TCP数据包的能力。

GRO
GRO(Generic Receive Offload)，基本思想跟LRO类似，克服了LRO的一些缺点，更通用。后续的驱动都使用GRO的接口，而不是LRO。

## 内核的一些实现 ##
其实其他的都还简单,参考下别人的文章也就了解了.之前一直没理解,收到的包,到底是怎么组织到一起的.组织的核心实现,是通过skb_gro_receive函数来完成的.
简单说下原理,gro接收到的所有包,都是通过skb中的frag_list来管理的.当收到第一个包的时候,对新建专门新建一个数据包,来存包第一个数据包的报头信息,然后将新的数据包中的frag_list指向刚刚收到的这个数据包,然后,在后续收到的数据包的时候,就直接将新的数据包连接到frag_list的结尾.这样就通过frag_list的方式将所有的数据包就都串到了一起.
这里就直接copy代码,然后在通过注释的方式来解释了.
{% codeblock lang:c %}
int skb_gro_receive(struct sk_buff **head, struct sk_buff *skb)
{
    struct sk_buff *p = *head;
    struct sk_buff *nskb;
    struct skb_shared_info *skbinfo = skb_shinfo(skb);
    struct skb_shared_info *pinfo = skb_shinfo(p);
    unsigned int headroom;
    unsigned int len = skb_gro_len(skb);
    unsigned int offset = skb_gro_offset(skb);
    unsigned int headlen = skb_headlen(skb);

    if (p->len + len >= 65536)
        return -E2BIG;
/*
当第一次数据包到达的时候,*head指向的是第一个数据包的地址,而对应的frag_list为空,当第二个数据包到的时候,也就是这里处理的第一个数据包到达的时候,frag_list肯定是空的,因为gro不会处理分片的数据包,也正因为如此,所以gro才能使用frag_list.
*/
    if (pinfo->frag_list)
        goto merge;
/*
这里的代码其实也没什么说的,就是说如果skb中headlen,小于报头的长度,就需要从page里的数据拷贝到*head里的page就好了
*/
    else if (headlen <= offset) {
        skb_frag_t *frag;
        skb_frag_t *frag2;
        int i = skbinfo->nr_frags;
        int nr_frags = pinfo->nr_frags + i;

        offset -= headlen;

        if (nr_frags > MAX_SKB_FRAGS)
            return -E2BIG;

        pinfo->nr_frags = nr_frags;
        skbinfo->nr_frags = 0;

        frag = pinfo->frags + nr_frags;
        frag2 = skbinfo->frags + i;
        do {
            *--frag = *--frag2;
        } while (--i);

        frag->page_offset += offset;
        frag->size -= offset;

        skb->truesize -= skb->data_len;
        skb->len -= skb->data_len;
        skb->data_len = 0;

        NAPI_GRO_CB(skb)->free = 1;
        goto done;
    } else if (skb_gro_len(p) != pinfo->gso_size)
        return -E2BIG;
/*
这里就是当frag_list为空的时候,新建一个head头部来存放,数据包头部的信息
*/
    headroom = skb_headroom(p);
    nskb = netdev_alloc_skb(p->dev, headroom + skb_gro_offset(p));
    if (unlikely(!nskb))
        return -ENOMEM;

    __copy_skb_header(nskb, p);
    nskb->mac_len = p->mac_len;

    skb_reserve(nskb, headroom);
    __skb_put(nskb, skb_gro_offset(p));

    skb_set_mac_header(nskb, skb_mac_header(p) - p->data);
    skb_set_network_header(nskb, skb_network_offset(p));
    skb_set_transport_header(nskb, skb_transport_offset(p));

    __skb_pull(p, skb_gro_offset(p));
    memcpy(skb_mac_header(nskb), skb_mac_header(p),
           p->data - skb_mac_header(p));

    *NAPI_GRO_CB(nskb) = *NAPI_GRO_CB(p);
/*
如果frag_list为null,那么,我就把head放到frag_list.然后将新的指针做为head,并把head->pre记录frag_list中的最后一个,用于后续再来包的话,直接添加到pre->next就好了.这样就直接连到了frag_list的最后
*/
    skb_shinfo(nskb)->frag_list = p;
    skb_shinfo(nskb)->gso_size = pinfo->gso_size;
    pinfo->gso_size = 0;
    skb_header_release(p);
    nskb->prev = p;

    nskb->data_len += p->len;
    nskb->truesize += p->len;
    nskb->len += p->len;

    *head = nskb;
    nskb->next = p->next;
    p->next = NULL;

    p = nskb;

merge:
    if (offset > headlen) {
        unsigned int eat = offset - headlen;

        skbinfo->frags[0].page_offset += eat;
        skbinfo->frags[0].size -= eat;
        skb->data_len -= eat;
        skb->len -= eat;
        offset = headlen;
    }

    __skb_pull(skb, offset);

    p->prev->next = skb;
    p->prev = skb;
    skb_header_release(skb);

done:
    NAPI_GRO_CB(p)->count++;
    p->data_len += len;
    p->truesize += len;
    p->len += len;

    NAPI_GRO_CB(skb)->same_flow = 1;
    return 0;
}
{% endcodeblock %}

其他地方的逻辑,代码很容易看明白的.

不过问题似乎不是出在这里,后面在把gso的内容给补充下

临了了抱怨句,白天看代码效率真低,半天没能去静下心来看这里的代码,晚上看了一会就明白里面的意思了.