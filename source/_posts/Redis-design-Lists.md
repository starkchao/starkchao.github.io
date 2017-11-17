---
title: 《Redis设计与实现》笔记：数据结构与对象 - 链表
date: 2017-11-17 15:01:54
tags:
- redis
categories: "redis"
---
Redis使用的C语言并没有内置链表这种数据结构，因此Redis构建了自己的链表实现。
<!--more-->

### <font color=#0099ff size=5>简介</font>
链表在Redis中的应用非常广泛，比如**列表键的底层实现之一就是链表**：当一个列表键包含了数量比较多的元素，又或者列表中包含的元素都是比较长的字符串时，Redis就会使用链表作为列表键的底层实现。
除了列表键之外，发布与订阅、慢查询、监视器等功能也用到了链表，Redis服务器本身还使用链表来保存多个客户端的状态信息，以及使用链表来构建客户端输出缓冲区。

### <font color=#0099ff size=5>链表和链表节点的实现</font>
每个**链表节点**使用一个adlist.h/listNode结构来表示：<div align=center>![listnode](/img/2017/11/17/listnode.png)</div>

Redis使用adlist.h/list来对链表进行操作：<div align=center>![list](/img/2017/11/17/list.png)</div>

list结构：
- head：表头指针
- tail：表尾指针
- len：链表长度
- dup：用于复制链表节点所保存的值
- free：用于释放链表节点所保存的值
- match：用于对比链表节点所保存的值和另一个输入值是否相等

由list结构和三个listNode结构组成的链表如下图所示：<div align=center>![list+listnode](/img/2017/11/17/list+listnode.png)</div>

<font color=#FF6A6A>**敲黑板画重点了**</font>
Redis链表特点：
- 双端：链表每个节点都带有prev和next指针，**获取某个节点的前置节点和后置节点的复杂度都是O(1)**
- 无环：表头节点的prev指针和表尾节点的next指针都指向NULL
- 带表头指针、表尾指针、链表长度：通过list结构，**程序获取链表的表头节点、表尾节点、链表中节点数量的复杂度均为O(1)**
- 多态：链表节点使用void *指针来保存节点值，并且可以通过list结构的dup、free、match三个属性为节点值设置类型特定函数，所以**链表可以用于保存各种不同类型的值**。