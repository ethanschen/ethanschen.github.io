+++
date = '2021-06-05T01:07:08+08:00'
draft = false
title = '一致性Hash算法与虚拟节点'
categories = [ "唠唠技术" ]
tags = [ "一致性Hash", "算法" ]
+++
## 缘起

今天我们来说说“一致性Hash”算法，以及虚拟节点。

这并不是一个难理解的概念，希望一篇文章下来，你能完全吃透。

在网站系统发展初期，前辈工程师探索出了数据库这一系统核心组件，

数据的持久化被与系统本身解耦开，独立发展且愈加可靠。

时间往后推移，随着互联网的普及，一个系统需要承载的用户数量指数级增长，

开发者不得不横向扩展服务器，通过负载均衡技术，使用户分散到各个服务器上。

随着服务器的增多，可靠的数据库系统也不堪重负，

开发者不得不将数据库中的数据通过“分库分表”技术，切分到不同的数据库中，

减轻单一数据库系统的压力。

那么问题来了，如何知道我们需要的数据在哪个数据库中？

没错。hash！

正如我们在 HashMap 中做的一样，对参数取 hash 值，再对 hash 值取模，

就可以既均匀切分存储数据，又知道数据在哪个库中。

## 简单hash

举个例子，现在有A，B，C，D共4个库，和参数为1，2，3……9，10共10个数据。

![简单Hash](https://res.cloudinary.com/dbsadrsxp/image/upload/v1736077122/2021-06-05-%E7%AE%80%E5%8D%95Hash_qmbf9a.webp)

我们简化hash算法为乘以1，

即 (1*1)%4=1，参数为1的数据落在A库中。

即 (2*1)%4=2，参数为2的数据落在B库中。

即 (3*1)%4=3，参数为3的数据落在C库中。

即 (4*1)%4=0，参数为4的数据落在D库中。

……

![简单Hash图2](https://res.cloudinary.com/dbsadrsxp/image/upload/v1736077373/2021-06-05-%E7%AE%80%E5%8D%95Hash%E5%9B%BE2_wgcdar.webp)

嗯！我很满意，一切井然有序。

当系统需要参数为2的数据时，只需要通过定位算法 (2*1)%4=2 便知道数据存在B库中。

当系统需要参数为9的数据时，只需要通过定位算法 (9*1)%4=1 便知道数据存在A库中。

可好景不长，正当系统稳定健康运行的时候，

B库不知道出现什么问题，失去了连接，系统中只剩下A，C，D共3个库。

这可真糟糕，但是我们还不知道会发生什么事情，对系统的影响有多大。

让我们先把目光聚集到局部上面来分析一下，

参数为2,6和10的数据存在B库上，影响应该是最大的，

如果现在系统需要参数为2的数据，那么它会通过定位算法  (2*1)%3=2 找到C库上。

![简单Hash图3](https://res.cloudinary.com/dbsadrsxp/image/upload/v1736077285/2021-06-05-%E7%AE%80%E5%8D%95Hash%E5%9B%BE3_iemdmd.webp)

但很遗憾，C上并没有存储参数为2的数据。

值得庆幸的是，原来数据是有副本的，失去连接的B库数据并没有丢失，而是在一个更大的主库中，

只要给系统一些时间，主库中对应B库的数据，就会根据定位算法被重新分配到A，C，D库中。

分配如下，

即 (2*1)%3=2，参数为2的数据落在C库中。

即 (6*1)%3=0，参数为6的数据落在D库中。

即 (10*1)%3=1，参数为10的数据落在A库中。

![简单Hash图4](https://res.cloudinary.com/dbsadrsxp/image/upload/v1736077435/2021-06-05-%E7%AE%80%E5%8D%95Hash%E5%9B%BE4_lufbom.webp)

好了，现在原来B库中的数据被重新分配到A，C，D库。

当系统需要取数据的时候，只需要通过参数根据定位算法，

到对应的库中读取即可。

如果现在你以为万事大吉，那可就太早了。

别忘了我们刚刚是聚焦到局部来分析状况的。

当我们目光拉远时，我们发现，

不止是参数为2,6,10的数据被重新分配，

几乎所有的数据都被重新分配了！

因为，

 (1*1)%3=1，参数为1的数据落在A库中。

 (2*1)%3=2，参数为2的数据落在C库中。

 (3*1)%3=0，参数为3的数据落在D库中。

……

![简单Hash图5](https://res.cloudinary.com/dbsadrsxp/image/upload/v1736077525/2021-06-05-%E7%AE%80%E5%8D%95Hash%E5%9B%BE5_g7cgue.webp)

真是糟糕透顶！

原本是想节约提高性能，结果凭空需要浪费这么多计算资源在重新分配数据上。

该怎么办呢？

诶，对，一致性Hash算法要大显身手了！

## 一致性Hash算法

一致性Hash算法通过一个 Hash 环，巧妙的让影响降到很低。

假设存在一个 Hash 环，它一圈的范围是 0 到 2^32-1，

先对数据库A，B，C和D的标识做 hash 运算，即上面的定位算法，

确定4个库在 Hash 环上的位置，

再通过定位算法将得出参数为1，2，3……9，10共10个数据在 Hash环上的位置，

这边我们不展示过程，只展示结果。

![一致性Hash](https://res.cloudinary.com/dbsadrsxp/image/upload/v1736077577/2021-06-05-%E4%B8%80%E8%87%B4%E6%80%A7Hash_mmbfui.webp)

一致性Hash算法规定我们将数据存在定位到的 Hash 环上位置顺时针遇到的第一个节点（数据库）。

即，

参数为1,4和8的数据存在数据库A中，

参数为5的数据存在数据库B中，

参数为3和7的数据存在数据库C中，

参数为2,6,9和10的数据存在数据库D中。

此时，其实我们已经不惧怕数据库宕机的情况了，

假设我们的数据库B又一次失去连接。

会发生什么情况呢？

![一执行Hash图2](https://res.cloudinary.com/dbsadrsxp/image/upload/v1736077641/2021-06-05-%E4%B8%80%E8%87%B4%E6%80%A7Hash%E5%9B%BE2_bckrcm.webp)

影响十分有限，只有数据库A和B在 Hash 环上的位置之间的数据，才受到了影响。

如图，参数为5的数据，需要重新从主库中复制到数据库C中，以保证系统需要参数为5的数据时可以顺利读取到。

而对于其他参数值的数据，并没有受到影响。

以上描述的是节点减少的情况，实际上在节点增加的时候，

一致性Hash算法依然可以保持大部分节点的稳定，不需要改变。

在这里我不做赘述，但你参考上面，独立思考一下。

一致性 Hash 算法如果只是到这里，实际上还引入了一个新的问题——数据倾斜。

即我们假设数据落在 Hash 环上每个位置的概率是一致的，

但实际上，每个节点覆盖的 Hash 环上的大小并不相等，

甚至可能有几倍的差距。

例如，在上面A，C，D库的基础上，我们添加了一个节点——数据库E。

它的位置如图所示。

![一执行Hash图3](https://res.cloudinary.com/dbsadrsxp/image/upload/v1736077713/2021-06-05-%E4%B8%80%E8%87%B4%E6%80%A7Hash%E5%9B%BE3_cm8ar0.webp)

因为它（数据库E）与上一个节点（数据库D）距离太近，导致没有任何一个数据落到它上面，

而正是与他相近特别近的上一个节点（数据库D），却存储了4个数据，

这就是数据倾斜，有些节点承载了很重的任务，有些节点却悠闲悠闲。

## 虚拟节点

为了解决数据倾斜的问题，我们还需要加入虚拟节点这一策略。

即，将每个数据库都通过定位算法生成几个在 Hash 环上的位置，

每个位置都承担上面节点的功能，

区别在于原来每个数据库对应一个节点，

现在每个数据库会对应若干个节点，

这就是虚拟节点。

![虚拟节点](https://res.cloudinary.com/dbsadrsxp/image/upload/v1736077771/2021-06-05-%E8%99%9A%E6%8B%9F%E8%8A%82%E7%82%B9_pmdari.webp)

为了避免图过于混乱，这边我标出 E 数据库的 3 个虚拟节点，

可以从图中看出，现在

E#1有 0 个数据，

E#2有 1 个数据（参数为5的数据），

E#3有 2 个数据（参数为6和9的数据）。

而实际上，不管数据定位后归属与E#1、E#2还是E#3，

在实际的数据存储和读取时，都是在数据库 E。

不难发现，在添加了虚拟节点的策略之后，

数据倾斜的情况得到了改善。

这就是完整的一致性Hash算法与虚拟节点啦！

记住，在实际应用中，3个虚拟节点是不够的，你需要更多的虚拟节点，以保证节点的负载更加均衡。
