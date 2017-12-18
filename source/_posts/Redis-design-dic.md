---
title: 《Redis设计与实现》笔记：数据结构与对象 - 字典
date: 2017-12-15 10:42:39
tags:
- redis
categories: "redis"
---
字典，又称符号表（symbol table），关联数组（associative array）或者映射（map），是一种用于保存键值对（key-value pair）的抽象数据结构。
Redis构建了自己的字典实现，并在Redis中应用广泛，比如Redis的数据库就是使用字典来作为底层实现，对数据库的增、删、改、查操作也是构建在对字典的操作之上。
字典还是哈希键的底层实现之一，当一个哈希键包含的键值对比较多，又或者键值对中的元素都是比较长的字符串时，Redis就会使用字典作为哈希键的底层实现。
<!--more-->
### <font color=#0099ff size=5>字典的实现</font>
Redis的字典使用哈希表作为底层实现，一个哈希表里面可以有多个哈希表节点，每个哈希表节点保存了字典中的一个键值对。
#### 1.哈希表
Redis字典所使用的哈希表由dict.h/dictht结构定义：
<div align=center>![dictht](/img/2017/12/15/dictht.png)哈希表结构定义</div>

- table属性是一个数组，数组中的每个元素都是一个指向dict.h/dictEntry结构的指针，每个Entry结构保存着一个键值对。
- size属性记录了哈希表的大小，即table数组的大小。
- used属性则记录了哈希表目前已有节点（键值对）的数量。
- sizemask属性的值总是等于size - 1，这个属性和哈希值一起决定一个键应该被放到table数组的哪个索引上面。
<div align=center>![dictht-example](/img/2017/12/15/dictht-example.png)空哈希表（size为4）示意图</div>

***
#### 2.哈希表节点
哈希表节点使用dictEntry结构表示，每个Entry结构都保存着一个键值对：
<div align=center>![dictEntry](/img/2017/12/15/dictEntry.png)哈希表节点结构定义</div>

- key属性保存着键值对的键
- v属性保存着键值对中的值，值是指针 / unit64_t整数 / int64_t整数
- next属性是指向另一个哈希表节点的指针，这个指针可以**将多个哈希值相同的键值对连接在一起，以此来解决键冲突的问题**。
<div align=center>![dictht-example-1](/img/2017/12/15/dictht-example-1.png)哈希表示意图</div>

***
#### <font color=#FF6A6A>3.字典</font>
Redis中的字典由dict.h/dict结构表示：
<div align=center>![dict](/img/2017/12/15/dict.png)字典结构定义</div>

type属性和privdata属性是针对不同类型的键值对，为创建多态字典而设置的：
- type属性是一个指向dictType结构的指针，每个dictType结构保存了一簇用于操作特定类型键值对的函数，Redis会为用途不同的字典设置不同的类型特定函数。
<div align=center>![dictType](/img/2017/12/15/dictType.png)dictType结构图</div>
- privadata属性则保存了需要传给那些类型特定函数的可选函数。
- ht属性是一个包含两个项的数组，数组中的每个项都是一个dictht哈希表，一般情况下，字典只使用ht[0]哈希表，ht[1]哈希表只会在对ht[0]哈希表进行rehash时使用。
- rehashidx：它记录了rehash目前的进度，如果目前没有在进行rehash，那么它的值为-1。
<div align=center>![dictht-example-2](/img/2017/12/15/dictht-example-2.png)普通状态下的字典</div>

***
### <font color=#0099ff size=5>哈希算法</font>
当要将一个新的键值对添加到字典里面时，程序需要先根据键值对的键计算出哈希值和索引值，然后再根据索引值，将包含新键值对的哈希表节点放到哈希表数组的指定索引上面。
#### Redis计算哈希值和索引值的方法如下：
<div align=center>![calculate-hash](/img/2017/12/15/calculate-hash.png)</div>

举个例子，假如我们要将一个键值对k0和v0添加到字典里面，那么程序会使用语句：
> hash = dict->type->hashFunction(k0);

计算键k0的哈希值。
假设计算得出的哈希值为8，那么程序会继续使用语句：
> index = hash & dict->ht[0].sizemask = 8 & 3 = 0;

计算出键k0的索引值0，这表示**包含键值对k0和v0的节点应该被放置到哈希表数组的索引0位置上**，如下图所示：
<div align=center>![add-key](/img/2017/12/15/add-key.png)</div>

当字典被用作数据库的底层实现，或者哈希值的底层实现时，Redis使用MurmurHash2算法来计算键的哈希值。该算法的优点在于，即使输入的键是有规律的，算法仍能给出一个很好的随机分布性，并且算法的计算速度也非常快。

***
### <font color=#0099ff size=5>解决键冲突</font>
当有两个或以上数量的键被分配到了哈希表数组的同一个索引上面时，我们称这些键发生了冲突（collision）。

Redis的哈希表使用<font color=#FF6A6A>链地址法（separate chaining）</font>来解决键冲突：每个哈希表节点都有一个next指针，多个哈希表节点可以用next指针构成一个单向链表，被分配到同一个索引上的多个节点可以用这个单向链表连接起来，这就解决了键冲突的问题。

举个例子，假设程序要将键值对k2和v2添加到哈希表里面，并且计算得出k2的索引值为2，那么键k1和k2将产生冲突，而解决冲突的办法就是使用next指针将键k2和k1所在的节点连接起来，如下图所示：
<div align=center>![add-key-1](/img/2017/12/15/add-key-1.png)</div>

因为dictEntry节点组成的链表没有指向链表表尾的指针，所以为了速度考虑，<font color=#FF6A6A>**程序总是将新节点添加到链表的表头位置（复杂度为O(1)）**</font>，排在其他已有节点的前面。

***
### <font color=#0099ff size=5>rehash</font>
随着哈希表保存的键值对的增多或减少，为了让哈希表的负载因子（load factor）维持在一个合理的范围之内，当哈希表保存的键值对数量太多或太少时，程序需要对哈希表的大小进行相应的扩展或者收缩。

扩展和收缩哈希表的工作可以通过执行rehash（重新散列）操作来完成，Redis对字典的哈希表执行rehash的步骤如下：
- 为字典的ht[1]哈希表分配空间，这个哈希表的空间大小取决于要执行的操作，以及ht[0]当前包含的键值对数量（也即是ht[0].used属性的值）：
	- 如果执行的是扩展操作，那么ht[1]的大小为第一个大于等于ht[0].used * 2的2 ^ n （2的n次方幂）
	- 如果执行的是收缩操作，那么ht[1]的大小为第一个大于等于ht[0].used的2 ^ n
- 将保存在ht[0]中的所有键值对rehash到ht[1]上面：rehash指的是重新计算键的哈希值和索引值，然后将键值对放置到ht[1]哈希表的指定位置上。
- 当ht[0]包含的所有键值对都迁移到了ht[1]之后（ht[0]变为空表），释放ht[0]，将ht[1]设置为ht[0]，并在ht[1]新创建一个空白哈希表，为下一次rehash做准备。

下边几张图展示了对哈希表的扩展操作，将哈希表的大小从原来的4扩大到现在的8：
<div align=center>![rehash-1](/img/2017/12/15/rehash-1.png)执行rehash之前的字典</div>
<div align=center>↓</div>
<div align=center>![rehash-2](/img/2017/12/15/rehash-2.png)为字典的ht[1]哈希表分配空间</div>
<div align=center>↓</div>
<div align=center>![rehash-3](/img/2017/12/15/rehash-3.png)迁移ht[0]的键值对到ht[1]</div>
<div align=center>↓</div>
<div align=center>![rehash-4](/img/2017/12/15/rehash-4.png)rehash完成</div>

***
### <font color=#0099ff size=5>渐进式rehash</font>
扩展或收缩哈希表需要将ht[0]里面的所有键值对rehash到ht[1]里面，但是，这个rehash动作并不是一次性、集中式地完成的，而是分多次、渐进式地完成的。

为了避免rehash对服务器性能造成影响，服务器不是一次性将ht[0]里面的所有键值对全部rehash到ht[1]，而是分多次、渐进式地将ht[0]里面的键值对慢慢地rehash到ht[1]。

下面几张图展示了哈希表渐进式rehash的步骤（注意字典的rehashidx属性是如何变化的）：
<div align=center>![rehash-5](/img/2017/12/15/rehash-5.png)准备开始rehash</div>
<div align=center>↓</div>
<div align=center>![rehash-6](/img/2017/12/15/rehash-6.png)rehash索引0上的键值对</div>
<div align=center>↓</div>
<div align=center>![rehash-7](/img/2017/12/15/rehash-7.png)rehash索引1上的键值对</div>
<div align=center>↓</div>
<div align=center>![rehash-8](/img/2017/12/15/rehash-8.png)rehash索引2上的键值对</div>
<div align=center>↓</div>
<div align=center>![rehash-9](/img/2017/12/15/rehash-9.png)rehash索引3上的键值对</div>
<div align=center>↓</div>
<div align=center>![rehash-10](/img/2017/12/15/rehash-10.png)rehash执行完毕</div>

#### <font color=#FF6A6A>渐进式rehash执行期间的哈希表操作</font>
在渐进式rehash的过程中，字典会同时使用ht[0]和ht[1]两个哈希表，所以在渐进式rehash进行期间，字典的删除(delete)、查找(find)、更新(update)等操作会在两个哈希表上进行：比如说，要在字典里面查找一个键的话，程序会先在ht[0]里面进行查找，如果没找到的话，就会继续到ht[1]里面进行查找，诸如此类。

另外，在渐进式rehash执行期间，**新添加到字典的键值对一律会被保存到ht[1]里面，而ht[0]则不再进行任何添加操作**：这一措施保证了ht[0]包含的键值对数量会只减不增，并随着rehash操作的执行而最终变成空表。

***
### <font color=#0099ff size=5>总结</font>

- 字典被广泛用于实现Redis的各种功能，其中包括**数据库**和**哈希键**
- **Redis中的字典使用哈希表作为底层实现**，每个字典带有两个哈希表，一个用于平时使用，另一个仅在进行rehash时使用
- 当字典被用作数据库的底层实现，或者哈希键的底层实现时，Redis使用MurmurHash2算法来计算键的哈希值
- 哈希表使用链地址法来解决键冲突，被分配到同一个索引上的多个键值对，会连接成一个单向链表
- 哈希表的rehash过程是渐进式完成的