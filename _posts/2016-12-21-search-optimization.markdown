---
layout: post
title:  "Search Optimization"
date:   2016-12-21 01:00:00
author: JIE YU
categories: elasticsearch
---

## Summary

#### [● 做擅长的事    ](#1)
#### [● 利用优秀算法](#2)
#### [● 空间换时间](#3)
#### [● 精简数据结构](#4)
#### [● es集群相关](#5)
#### [● 系统架构](#6)
#### [● 监控](#7)

<br />

## <a name="0">0. 写在前面</a>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;本文主要是最近对公司搜索业务进行优化的总结，再加上自己对搜索的理解，所做的一个简单记录。
<br /><br />

## <a name="1">1. 做擅长的事</a>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我们的application适合做什么，搜索、计算？显然不是。与elasticsearch相比，我们更适合做业务，我们更懂业务。而es比较擅长索引、计算。所以我们的优化思路就是让application做业务逻辑，把索引、计算交给elasticsearch。

<br />

## <a name="2">2. 利用优秀算法</a>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;优化前，我们的很多计算逻辑都是在application中完成的，如距离、价格的计算等。但大量的计算会使应用的效率低下，接口的QPS很低。在研究了elasticsearch的geohash之后，我们决定对距离和价格计算做一个优化。首先，我们把elasticsearch升级到了2.x版本，它支持geo索引，官网有关于geohash的相关介绍（https://www.elastic.co/guide/en/elasticsearch/guide/2.x/geohashes.html），这里不展开。我主要介绍一下受geohash启发，对价格区间搜索做的一个优化。
很多电商网站都有价格区间搜索的功能（如下图），根据价格区间搜索需要的产品。
-----------这里有张图--------------
常规的也是我们之前使用的方法是传入价格区间参数，使用es的range对docs进行过滤：
```
  "range" : {
    "price.price" : {
      "from" : 150.0,
      "to" : 300.0,
      "include_lower" : true,
      "include_upper" : true
    }
  }
 ```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这样做的缺点就是，每次搜索都会有大量的计算，百万数据情况下仅es就会耗时200ms甚至更长时间，在多个range的情况下更糟糕。针对这种情况，我们对document做了一下改造，在document里新增一个属性priceTag，把价格区间抽象成若干priceTag，比如[0，150)、[150，300)、[300，450)、[450，+∞），这四个区间就可以对应四个priceTag，即1、2、3、4。这样，在index是就可以给每个价格映射一个tag值，搜索的时候同样将传入的区间值映射成tag值，利用es的terms去做精确匹配。举个例子，有个商品价格为193元，按上述方法，它的priceTag应该是2，假如搜索的区间是[50, 450], 那么它对应的tag值应该是一个数组，即[1，2，3]，结构如下：
``` 
  "must": {
    "terms": {
      "price.priceTag": [1, 2, 3]
    }
  }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;很明显，这种场景下terms要比range效率高很多，实际测试优化后的方案在es中返回数据只要20ms左右的时间，速度提升10倍。
<br /> <br />

## <a name="3">3. 空间换时间</a>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;空间换时间的思路就是把时效性较弱的数据预先放到缓存或者把一部分计算逻辑提前做好，做到数据即取即用，主要体现在三个方面：首先第二条中将价格区间映射成tag的做法也是一个例子，把原本实时计算的方式转变成先计算，再存储，数据直接可用的方式；然后就是对于商品的一些描述性的、不太容易变动的属性信息，转变成可直接使用的数据结构做本地缓存，然后定期更新，这样在取用时就基本不用消耗时间；再有就是将搜索结果或者热数据做一次缓存（redis等），这个缓存可以根据业务需求加上相应的过期时间，这样做能大大提高热门数据的搜索效率；最后就是可以适当的做一些客户端缓存，更进一步优化用户体验。
<br /> <br />

## <a name="4">4. 精简数据结构</a>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;好的设计一定是精简的，document结构一定要足够简单，只保留参与索引的必要的字段。有些时效性不高的field可以做一些预处理，尽量减少搜索时的计算。另外，复杂的数据结构在json（或xml）解析时非常耗时，也会影响数据传输效率，所以必须精简数据结构。
<br /> <br />

## <a name="5">5. es集群相关</a>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用es，除了相应要快，还要保证它的高可用。下面讨论一些常见的集群相关的问题。由于网络或者个别机器负载过大（data和master在一个node上的情况），有可能会导致脑裂的情况。建议设置discovery.zen.minimum_master_nodes这个参数为N/2+1，其中N为最小候选节点数，比如有三个候选节点（即node.master为true的节点），那么这个参数就可以设置成2，这个数字代表在候选节点中至少有两个节点“同意”，某一结点才能成为master。另外一个参数是discovery.zen.ping_timeout，等待ping相应的超时时间，默认是3秒，在网络情况不太好或者节点负载较高响应缓慢的情况下，可以适当调大这个参数，这样可以避免节点间的“误会”。当然还有一些其他参数可以调优，在官方文档都有介绍（https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-zen.html）。强烈建议多看官方文档，写的非常好。
<br /> <br />

## <a name="6">6. 系统架构方面的考虑</a>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;前面提到的都是相对底层的搜索优化，从底层数据到用户其实还有很长一段距离。在很多系统架构中，都会有一些看似必要的中间层，做业务分发、数据封装之类的工作，但从用户获取数据的整个历程中，这些中间层其实并不是那么重要，但却会消耗一部分响应时间（主要是数据的解析和转换），也会严重降低搜索的QPS，所以，有必要对系统架构做一些精简，尽可能减少一些中间层。
<br /> <br />

## <a name="7">7. 监控</a>

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;这里的监控包含三个方面：应用、中间件和用户数据。应用的监控可以使用zipkin、cat等比较优秀的开源工具，及时发现搜索过程中的各种问题，尽快提出优化方案；中间件的监控主要是针对elasticsearch、redis等的监控，可以使用zabbix等工具对重要的端口进行监控，对于elasticsearch（2.x版本）可以使用官方推荐的Marvel、Reporting对集群进行监控，当然5.x版本可以使用X-Pack，它是一个非常强大的工具，用官方的话“Security, alerting, monitoring, reporting, and Graph in one pack”；用户数据对任何一家公司都是非常重要的，这里只简单提一下用户数据对搜索的重要性，当用户达到一定基数，就可以通过应用日志、页面埋点等方式获取大量用户数据，分析这些数据可以了解用户的搜索行为，了解到这些就可以对反过来对搜索优化提供方向，比如对个别产品增加搜索权重等等。
<br /><br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;总的来说，监控就是为了让系统稳定，让用户体验更好，做好监控也是搜索优化的一个很重要的部分。

