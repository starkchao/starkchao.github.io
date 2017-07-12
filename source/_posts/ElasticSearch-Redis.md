---
title: ElasticSearch + Redis的一个应用实例
date: 2017-07-10 11:40:28
tags:
- es
- redis
categories: "es"
---
##### <font size=4>应用实例</font>
***
###### <font size=3>使用场景</font>
在ElasticSearch中的device索引里共有9688085条记录，每条记录包含了一些信息，如用户id，设备guid，设备型号以及用户经纬度。现在希望根据这些数据当中的经纬度值，计算出用户分布的统计结果。当然在对这些数据进行处理之前，肯定是要进行筛选的，比如去除一些没有获取到经纬度的记录等，最后剩余大约700W+的数据。数据筛选的部分暂且略过。
<!--more-->
###### <font size=3>存在问题</font>
1. 700W+的数据存储在ES当中，<font color=red>**如何遍历这些数据**</font>？
2. 对于每一条记录当中的经纬度值，需要调用地理位置信息接口获取该经纬度对应的地址，并按照省市划分，如何在数据量不断增大的同时，能够保证这样的数据结构，并实现<font color=red>**快速的存、取**</font>？
3. 最终形成的统计结果，怎样实现可视化？

###### <font size=3>解决方案</font>
- **Q1**：<font color=green>**采用ES的scroll翻页查询**</font>。
ES支持**两种翻页**方式：
（1）第一种是类似于sql中limit，在ES中是from/size。这种方式在数据量不大的情况下是可以使用的，但是在深度分页的情况下效率极低。它的原理是返回size条数据，然后从from处截断。所以当我要查询from=10000，size=10010的数据时，会查询前10010条数据，但是只截取10000-10010条数据返回。这就造成了极大的浪费。
（2）第二种是类似于sql中的cursor，在ES中是scroll查询。scroll查询会在前一次的查询的基础上继续查询。在查询时添加一个scroll参数，用来指定游标查询的过期时间。**这个时间会在每次查询时刷新，所以这个时间只需要足够处理当前批次的结果就可以了，而不是处理查询结果的所有文档的时间**。对于大批量的数据，需要用scroll查询来获取数据。
- **Q2**：<font color=green>**使用Redis做存储和查询**</font>。
Redis作为一个内存中的数据结构存储系统，可以被用作数据库、缓存和消息中间件。丰富的数据结构类型（如 字符串（strings）， 散列（hashes）， 列表（lists）， 集合（sets）， 有序集合（sorted sets）），对这些类型的原子操作，持久化以及主从复制等功能Redis都能够提供。
由于数据量会迅速增大，高效率的查询和更新是必须满足的，redis作为内存型的数据库就是以速度著称。另外省：市：数量 这样的数据结构，可以使用redis中的哈希键（key-field-value）来实现。
- **Q3**：统计结果写入日志文件。
对于最后的统计结果，可以通过遍历redis中的哈希键来获取field和value，将结果记录到日志文件中。

##### <font size=4>代码实现</font>
***
###### <font size=3> 1 - ES和Redis配置</font>
![es-redis-config](/img/2017/07/10/es-redis-config.png)
> Redis配置主要是host，port和pwd。
> ES配置中要说明的是type_name，由于在对应的index下有多个type，所以我在这里指定了我要查询的type。

###### <font size=3>2 - 构造方法初始化redis</font>
![construct](/img/2017/07/10/construct.png)

###### <font size=3>3 - 初始化查询，获取scroll_id </font><font color=red>[★]</font>
![init-query](/img/2017/07/10/init-query.png)
> 在ES中使用游标查询，需要在普通的查询基础上，添加scroll参数，值代表游标过期时间，如（scroll=1m）。
> 在查询返回的结果中，得到名为'_scroll_id'的值，作为下一次查询的游标id。

###### <font size=3>4 - 初始化查询后，使用scroll_id继续查询</font><font color=red>[★]</font>
![normal-query](/img/2017/07/10/normal-query.png)
> 在得到游标id以后，就不需要继续指明查询条件了。

###### <font size=3>5 - 使用curl发送请求到es</font>
![curl](/img/2017/07/10/curl.png)
> 这里的就是一个常规的curl请求方式。需要注意的是对于传入的data参数的分解，这样处理之后可以支持批量查询，分解的原因参考ES的批量查询。

###### <font size=3>6 - 计算统计分布</font><font color=red>[★]</font>
![count-distribution](/img/2017/07/10/count-distribution.png)
> 计算分布时调用了根据GPS信息获取地理位置信息的接口，返回结果里会包含详细的省市县以及街道等信息。

###### <font size=3>7 - 结果写入日志文件</font>
![result-to-log](/img/2017/07/10/result-to-log.png)

###### <font size=3>8 - 释放redis中的数据</font>
![clear-redis](/img/2017/07/10/clear-redis.png)
> 由于在往Redis中存储时使用了哈希键，而键使用了省级名称，所以可以按照省级名称为key的方式，释放Redis当中的数据。

###### <font size=3>9 - 最终的run方法</font>
![run](/img/2017/07/10/run.png)
> 综合上边的一系列方法后得到的run方法。

<font color=red>**注**</font>：在使用redis的hGetAll方法时，遇到了段错误（segmentation fault）。当时php版本是7.0.9，在使用php 7.0.16运行脚本后解决。



