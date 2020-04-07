
----
layout:     post
title:      Elasticsearch根据term查询id列表
subtitle:   Elasticsearch底层原理
date:       2020-04-06
author:     搬砖侠
header-img: img/es.jpg
catalog: 	 true
tags:
    - ElasticSeach
----

# 前言
Elasticsearch底层数据原理

# 正文
## 传统数据库使用B-tree索引 

二叉树的效率是logN，同时插入新的节点不必移动全部节点，所以采用树形结构存储索引，能够同时兼顾插入和查询的性能。因此在这个基础上，再结合磁盘的读取特性，传统关系型数据库采用b-tree或b+tree这样的数据结构；为了提高查询效率，减少磁盘寻道次数，将多个值作为一个数组连续区间存放，一次寻道读取多个数据，同时也降低了树的高度。

## lucene倒排索引
提出几个问题：
```
1、为什么说Elasticsearch的倒排索引比B-TREE更快呢？
2、ES是从内存中查找数据还是从磁盘中读取数据？
3、ES为什么那么快？
```

Elasticsearch的倒排索引是由*term index、term dictionay*以及*posting list*组成：
![531f994f3992f42711eeec06798d34ab.png](evernotecid://0ABD4F8F-9514-42ED-9A5C-CBEA96656BCB/appyinxiangcom/20212136/ENResource/p517)
现在假设有几条数据如下：
| ID | Name | Age  |  Sex     |
| -- |:------------:| -----:| -----:| 
| 1  | Kate         | 24 | Female
| 2  | John         | 24 | Male
| 3  | Bill         | 29 | Male
根据name创建倒排索引如下：
| Term | Posting List |
| -- |:----:|
| Kate | 1 |
| John | 2 |
| Bill | 3 |


### **Posting List**
ElasticSeach会为设置需要索引的字段创建倒排索引（默认index为true），如上述表格中的数据显示：Kate,John,Bill为term，而1，2，3就是Posting List。Posting List是一个文档ID列表，存储了符合某个分词结果term的所有ID。

*提出一个问题：现在需要term为kate的Posting List，如果数据量很少，通过顺序查找便可快速获得结果，如果数据量很大呢？*

现在看一下Elasticsearch是通过什么方式来提升查询速度的。
### **Term Dictionary**
为了像二分查找一样，ES提出了*Term Dictionary*的概念:将所有的Term进行排序，二分查找term的时间复杂度为logN。

这样看来其实和传统数据库使用的B-Tree类似，那为什么说ES的倒排索引会比B-Tree更快呢？并且ES为了提高查询效率，会在内存中查找Term，并不会像B-tree从磁盘中读取数据，如果term数据量很大，*Term Dictionary*也会很大，全部放在内存中也不现实，所以ES提出了*Term Index*。

### Term Index
*Term Index*就像字典中的索引页一样，它存储了例如A开头的放在第几页。Term Index可以理解为一棵树。
![e4632ac1392b01f7a39d963fddb1a1e0.png](evernotecid://0ABD4F8F-9514-42ED-9A5C-CBEA96656BCB/appyinxiangcom/20212136/ENResource/p518)

*Term Index*并没有存储所有的term，它只是存储了term的前缀和*Term Dictionary*的映射关系，通过Term Index可以快速的定位到存放在磁盘中*Term Dictionary*的某个offset，然后往后顺序查找。再使用一些压缩技术（FST），*Term Index*所占用的内存空间仅有所有term的几十分之一，使得*Term Index*在内存中存储变成了可能。
最终的查询流程图如下所示：
![e4599b618e270df9b64a75eb77bfb326.jpeg](evernotecid://0ABD4F8F-9514-42ED-9A5C-CBEA96656BCB/appyinxiangcom/20212136/ENResource/p519)

*现在回答第一个问题：为什么ES的倒排索引会比B-tree快？*
mysql只有*Term Dictionary*这一层，是以B-tree的方式存储在磁盘中的。检索一个term需要若干次的随机读取操作。而Lucene在*Term Dictionary*的基础上加入了Term Index加速搜索，*Term Index*以树的形式存放在内存中，从*Term Index*中查找到*Term Dictionary*的block位置后，再去磁盘中查找term，这样大大的减少了随机读取的次数。

### Lucene的压缩技巧

字典常用的数据结构：
| 数据结构 | 优缺点 |
| -- |:----:|
|排序列表Array/List|使用二分法查找，不平衡|
|HashMap/TreeMap|性能高、内存消耗大，几乎是原始数据的三倍|
|Skip List|跳跃表，适用于高并发场景。[跳跃表详细介绍](https://www.iteye.com/blog/kenby-1187303)|
|Trie|适用于英文词典，且有大量英文字符串并且没有公共前缀的情况。[Trie详细介绍](http://dongxicheng.org/structure/trietree/)|
|Double Array Trie|适用于中文词典，内存占用小。[双数组Trie](https://blog.csdn.net/zhoubl668/article/details/6957830)|
|Ternary Search Tree|三叉树，兼具占用空间小和查询快的优点。[Ternary Search Tree](http://www.drdobbs.com/database/ternary-search-trees/184410528)|
|FST|一种有限状态转移机，lucene4有开源实现|

##### ***1、Term Index的压缩***
上文已经说过Term Index通过FST压缩存储。
FST有以下优点：
1、占用空间小。通过对词典前缀和后缀的重复利用，压缩了存储空间
2、查询速度快。时间复杂度为len(str)
现演示“cat”“deep”“do”“dog”"dogs"的FST存储
(1)cat
![faaaed78b9352325afaa9c4db03bc67d.png](evernotecid://0ABD4F8F-9514-42ED-9A5C-CBEA96656BCB/appyinxiangcom/20212136/ENResource/p520)
(2)deep
![6e5a2798ae4a9e0733eb2f0f0479a97b.png](evernotecid://0ABD4F8F-9514-42ED-9A5C-CBEA96656BCB/appyinxiangcom/20212136/ENResource/p521)
(3)do
![76fa8c68a813ad0d38865fbaf127b274.png](evernotecid://0ABD4F8F-9514-42ED-9A5C-CBEA96656BCB/appyinxiangcom/20212136/ENResource/p522)
(4)dog
![a8d379120177349e0312a8a2cd5e77a2.png](evernotecid://0ABD4F8F-9514-42ED-9A5C-CBEA96656BCB/appyinxiangcom/20212136/ENResource/p523)
(5)dogs
![0db5c6562b0c46279c1f73ab448d5767.png](evernotecid://0ABD4F8F-9514-42ED-9A5C-CBEA96656BCB/appyinxiangcom/20212136/ENResource/p524)


##### ***2、Term Dictionary的压缩***
Term Dctionary在磁盘中是以block的形式保存的。一个block内部利用公共前缀压缩，比如以AB开头的会把AB去掉进行存储

##### ***3、Posting List的压缩***
如果Posting List包含数百万个ID，Es是如何优化压缩存储的？答案就是Frame Of Reference：
*增量编码压缩，将大数变为小数，按字节存储。*

![a362927d48c9b2fd68d80d51bedcaf6e.png](evernotecid://0ABD4F8F-9514-42ED-9A5C-CBEA96656BCB/appyinxiangcom/20212136/ENResource/p525)

Elasticsearch要求Posting List中的ID是有序的。

***Roaring bitmaps***

假设某个posting list：
[1,3,5,6]
那么用bitmap表示为：
[1,0,1,0,1,1]
使用bitmap表示非常直观，但是bitmap有个缺点就是：随着文档数的增多所占用的空间呈指数增长。
![9482b84c4aa3fb77a959c1ead553037e.png](evernotecid://0ABD4F8F-9514-42ED-9A5C-CBEA96656BCB/appyinxiangcom/20212136/ENResource/p527)
其实压缩的思路很简单，与其保存100个0，占用100bit，还不如保存一次0，然后声明0被重复了100次。


***联合索引***

上面讨论的都是对单个field索引的查询，那么如何对多个field的索引进行联合搜索呢？
比如查找出age为18并且sex为男的文档，对于mysql来说，假如对age和sex都建立的索引，只需要在遍历其中一个条件的时候过滤掉另外一个即可；但是对于Es来说，理论上可以分别查找出age为18和sex为男的posting list，然后做and操作即可。
ES有两种方式来实现联合索引：
- **使用skiplist。同时遍历出age和sex的posting list，然后skip并求交集(Frame Of Reference)**
- **使用bitset。分别求出age和sex的bitset,然后做与运算（Roaring bitmaps）**
1、如果使用skiplist,对最短的posting list中的id，从小到大依次遍历，逐个查找其他的posting list该id是否存在，在查找的过程中可以跳过一些元素，如在查找2的时候，可以跳过红色和蓝色的1。
![eafa46683272ff1b2081edbc8db5469f.jpeg](evernotecid://0ABD4F8F-9514-42ED-9A5C-CBEA96656BCB/appyinxiangcom/20212136/ENResource/p526)

过程如下：
```
Next -> 2
Advance(2) -> 13
Advance(13) -> 13
Already on 13
Advance(13) -> 13 MATCH!!!
Next -> 17
Advance(17) -> 22
Advance(22) -> 98
Advance(98) -> 98
Advance(98) -> 98 MATCH!!!
```

2、如果使用bitset,直接按位与即可得到最终结果。

***结论***
（1）这两种合并使用索引的方式都有其用途，但对于简单的相等条件的过滤缓存成纯内存的bitset还不如直接访问skiplist的方式要快。两种方式性能比较：[Frame Of Reference和Roaring bitmaps](https://www.elastic.co/blog/frame-of-reference-and-roaring-bitmaps)
（2）不需要索引的字段一定要明确定义出来，因为es默认是索引的
（3）选择有规律的ID

***参考***：https://www.infoq.cn/article/database-timestamp-02/?utm_source=infoq&utm_medium=related_content_link&utm_campaign=relatedContent_articles_clk

> 本文首次发布于 [搬砖侠](http://kaakaome.github.io), 作者 [@搬砖侠](http://github.com/kakaome) ,转载请保留原文链接.
