title: CCF目录单页版
category: misc
tags:
date: 2017-02-25
---

前一段时间整理论文列表，感觉分成多页的目录用起来不太方便。于是就粘贴复制，把它们合并到一起。
Markdown排版比较麻烦，分别保存为HTML格式和Excel格式。
CCF网站2016年底改版了一次，目前还保留着旧版网站，新旧网站都有目录的内容。

各链接如下：
+ **[我合并后的CCF目录完整列表，HTML链接，2017-02-25](/doc/ccf-all-in-one-2017-02-25.html)** 
+ **[我合并后的CCF目录完整列表，Excel文件](/doc/ccf_all_in_one_2017-02-25.xlsx)**
+ CCF **新** 网站链接：[CCF目录网页版](http://webtest.ccf.org.cn/xspj/gyml/)，[2015版的PDF](http://www.ccf.org.cn/ccf/contentcore/resource/download?ID=32826)
+ CCF **旧** 网站链接：[CCF目录网页版](http://history.ccf.org.cn/sites/ccf/paiming.jsp)，[2015版的PDF](http://history.ccf.org.cn/sites/paiming/2015ccfmulu.pdf) 

<!--more-->

---

# 说明
表格中的类别简写分别是:
+ AI：人工智能
+ 系统：计算机体系结构/并行与分布计算/存储系统
+ 软工：软件工程/系统软件/程序设计语言
+ 数据库：数据库/数据检索/内容检索
+ 安全：网络与信息安全
+ 多媒体：计算机图形学与多媒体
+ 网络：计算机网络
+ 理论：计算机科学理论
+ 其它：交叉/综合/新兴
+ 交互：人机交互与普适计算

表格中是按拼音排序的，排序字段依次是：方向，类型（会议/刊物），级别（A/B/C）。

# CCF目录中存在的问题

## 同一刊物同时被列入不同方向
多个研究方向经常出现交叉，所以这种情况也是可以接受的。
不过软工方向的一些会议，比如OSDI，就没有被重复列入系统方向。
由于列表是按研究方向组织的，不太容易发现重复，如果按会议/期刊排序，就很容易发现了。

+ B类刊物 `DKE: Data and Knowledge Engineering` 分别被列入了 AI 和 数据库 方向
+ C类刊物 `IJIS: International Journal of Intelligent Systems` 分别被列入了 AI 和 数据库 方向
+ C类刊物 `IPL: Information Processing Letters` 分别被列入了 理论 和 数据库 方向
+ B类刊物 `TOMCCAP: ACM Transactions on Multimedia Computing, Communications and Applications` 分别被列入了 网络 和 多媒体 方向
+ A类会议 `HPCA: High-Performance Computer Architecture` （系统方向）的链接是[http://dblp.org/db/conf/cnhpca/] ，这个是国内的同名会议， **我把链接改成了国际的HPCA会议** [http://dblp.org/db/conf/hpca/]
+ C类会议 `CLOUD` [http://dblp.org/db/conf/IEEEcloud/] 2018年分裂为两个 **同名** 但不同人员组织的会议，论文分别由IEEE和Springer发表，不知以后其中一个会不会改名字，**建议以发表在IEEE的为准** 。

## 同一方向中的重复会议
+ 安全方向的 C类会议 `ASIACCS`和 A类会议 `CCS`给出的链接都是 [http://dblp.org/db/conf/ccs/] ，这个 **不是** 错误，因为dblp上确实是在同一页面显示了 `ASIACCS` 和 `CCS`，不过列表中 `ASIACCS`的全称不太恰当
+ 交互方向的 C类会议 `DIS: ACM Conference on Designing Interactive  Systems` 在第2条和第11条重复出现了两次


#TODO NOMS vs. APNOMS

#TODO 一个新的 **期刊** ，SIGMetrics下的 POMACS 
Proceedings of the ACM on Measurement and Analysis of Computing Systems [https://dl.acm.org/citation.cfm?id=J1567]

<img src="/img/ccf_sum.png" alt="会议/刊物分方向汇总">
