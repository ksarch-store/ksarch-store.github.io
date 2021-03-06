---
layout: post
title: Memcache 访问热点导致服务雪崩的一个 case
tags: Memcache, 陷阱
category: memcache
author: Shen Jiale
email: shenjiale@baidu.com
---

去年年底，某产品线发生了一起严重的丢失用户流量的事故，就这个 case 来谈谈 Memcache 使用不当的问题。

他们的使用方法是这样的，在站点的主页上每次请求会首先请求一个单热点 key，value 大概在 250 KB 左右。

![Get 请求热点 key](/images/2015-03-05-001.svg)

以千兆网卡的容量计算，热点机器网卡容量极限为 400 QPS。若请求失败（cache 失效、访问超时等），PHP 会根据业务逻辑，再请求大约 600 个 cache 数据。

![请求 600 个 cache 数据](/images/2015-03-05-002.svg)

然后重新构造首页的数据块。

![重新构造](/images/2015-03-05-003.svg)

在当天下午 14 ~ 15 点左右，用户流量有自然增长，超过了 400 QPS，于是那台热点的机器单机网卡打满，大批量的首页请求获取热点 cache 失败。
PHP 业务为了重新构造数据块，另外请求 600 个 key，导致所有的 cache 机器请求都上涨，网卡占用上涨。同时，由于请求 600 多次 cache 需要耗时过长，产生了很多的长耗时请求，这些请求占用 dbproxy 连接不释放，导致 DB 的连接数也打满。

此时，其他请求大量失败，因此对于 memcache 调用减少，但是还是有相当量的首页请求仍然在请求单热点，导致单热点网卡依然处于打满情况，其他机器网卡有所下降。此时产品线无法提供正常服务，处于挂站状态。

那我们如何防止这样的事故发生呢？


1. 明确使用场景，防止滥用

    首先要确定一个需求是不是适合用 cache。大多数场景下，cache 里存储的都是几百个字节的小数据（如帖子列表、用户信息、图片元信息等），复杂的结构数据序列化之后一般也不会超过 2 KB。

    250 KB 的大数据块，如果是图片，应该塞到图片存储系统；如果是整个网页，那么展示时“实时渲染 VS 直接从 cache 取”这两种方案，还需商榷。

2. 不人为制造访问热点

    Memcache 的访问本身就具有一定的热点（比如某些书看的人多、一段时间内的热门话题等），在实际工程中，这些热点也是需要尽量被平均的。

    然而在这个 case 中，人为制造了热点，即，对同一个 key 的访问在每一个请求的关键路径上，这是一定需要避免的。

3. 实例拆分

    多个业务（如主页和文章列表等）使用同一个 Memcache 实例，某一个业务流量飙升（正常增长或隐藏的 bug 导致流量异常）就会导致其它所有的业务访问受影响，一挂挂全站。

    这时需要把比较重要的服务依赖的 cache 拆分成单独的实例，尽量减少互相影响，提升可用性。


做到这些，也只是构建一个健壮的业务的沧海一隅，后面的博客里会继续谈谈 Memcache 使用时其它的坑。
