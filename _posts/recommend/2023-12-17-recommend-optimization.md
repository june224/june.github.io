---
title: golang推荐服务的一些优化经验
author: june
date: 2023-12-17
category: recommend
layout: post
---

### 背景

一个常规的推荐接口需要经过召回，粗排，精排，重排等阶段，相同流量体量下，相比常规服务端接口，推荐接口复杂程度更高，面临的挑战也更大。但因为推荐系统对数据延迟性容忍度比较高的特性，所以推荐服务有比较大的优化空间。本文将阐述golang推荐服务经常碰到的性能问题，以及优化思路和经验。

> 若读者对推荐流程不了解，建议先阅读上一篇文章《[ 推荐系统设计简述 ](2023-12-16-recommend-system-overview.html)》

### 问题

推荐是从N个item对象里，下发M个符合用户偏好的item对象，且M远小于N（M数量级为千或万，N数量级一般为个位数，最多也不会超过20）。所以用户和item是一对多的关系。所以对于推荐服务，在item侧，问题很容易被放大，而且这也往往是推荐服务的性能瓶颈。
* 服务计算量大。举个例子，一个推荐请求，往往是从全量的item对象池里召回符合条件的item对象，假设QPS为1k，item对象全量有1w，每次召回对item做1次条件判断，那么1s就会有1k*1w*1=1000w次计算。但单个item往往不会只有1次条件判断。因此在item侧，一个很小的问题很容易被放大。
* 容易产生较多临时对象，从而导致频繁申请释放对象，gc频率和耗时都变高，从而影响服务的性能。比如上面的例子，在对单个item的一次条件判断过程中产生8Bytes的临时对象，那么1s中就产生了约76MB的临时对象内存。
* 容易产生长任务，从而导致服务出现长尾效应。

### 优化三大原则

服务优化总结下来，基本遵循三个原则：减少，降频和复用。
* 减少：去掉已下线业务逻辑，精简请求数据量，减少网络请求。
* 降频：一般都是空间换时间思路，比如数据结构从list改为map结构，检索元素事件复杂度从O(N)变为O(1)。
* 复用：对象复用，避免频繁创建和回收对象。常规做法是使用对象池。

### 推荐服务优化经验

* **缓存预热**。一次推荐请求，无论是在召回，不但需从召回池多路召回item对象，还需请求上千上万item对象的召回画像，还是在精排，需请求几百个item对象的上百个item特征画像，请求量都是巨大的。若不做缓存预热，每次请求都打到召回服务和画像服务，召回服务和画像服务都需承受巨大的压力，特别是QPS高，业务逻辑复杂的推荐场景。又由于推荐服务对数据延迟性容忍度比较高，可根据不同推荐场景，针对不同召回队列的召回结果，以及对画像数据做不同的缓存策略。   
  比如定时预拉取召回量大的或者请求频率高的召回队列及其item画像至本地，甚至召回池不大的推荐场景，可全量预拉取item对象及其画像至本地，再比如根据不同的画像决定是采取LRU还是LFU缓存策略。

* **避免频繁产生过多临时对象**。假若推荐接口QPS为1k，多路召回队列总共召回1w个对象，则1s产生1kw个临时对象。这势必会导致服务gc频繁，成为服务性能瓶颈。因此，对于频繁创建的对象使用对象池，比如召回对象，画像对象，特征对象等使用对象池，对象池大小即为QPS乘于单次请求产生的对象数量。

* **提高并行度**：串行请求改为并行请求，大任务拆分小任务并行处理。将无先后顺序的串行远程调用改为并行请求，则多个请求耗时取决于耗时最长的远程调用。比如，推荐请求的数据准备阶段，ab系统实验数据获取，用户侧画像数据获取和曝光过滤器获取，哪个远程调用先请求都可以，并无先后顺序依赖，即可改为并行请求。  
  除此之外，还可将大任务拆分多个小任务并行处理。举个例子，精排阶段，几百个item对象，一个item特征可能会有上百个特征，一个特征可能会需要执行几个算子，一次请求，特征处理计算量非常大。并且，将几百个item的那么多特征数据塞进一个请求里调用模型服务，网络传输耗时也是不可忽视的。不仅如此，对于模型服务，单个请求推理预测任务计算量过大，也容易造成其它小请求的任务阻塞，从而形成长尾效应。   
  因此，对于精排阶段，可将需进行特征处理和模型打分的item对象大数组拆分成几个小数组，多协程并行处理各自小数组的item对象的特征以及模型预测打分。不但提高cpu资源的利用率，还降低特征传输的网络耗时以及减轻模型服务的长尾效应。

* **空间换时间**。用空间复杂度换取时间复杂度是常见的一种优化思路。比如，某个数据的数据结构由slince改成map，底层存储变复杂了，但查找某个元素时间复杂度由O(n)变为了O(1)。特别是针对item对象的数据，数据结构由list改为map大大降低了时间复杂度。但元素个数较大，也需要考虑map数据结构扩容带来的性能问题。总之，空间复杂度和时间复杂度需根据实际情况来进行取舍。

### 关于k8s的调整

如果服务部署在k8s上，可利用k8s的特性来提高服务系统的稳定性。比如：
* **绑核**。绑核即是pod独占所需的cpu core。绑核对于计算密集型的推荐服务性能提升尤为显著。比如加载深度学习模型的tensorflow serving服务，就以笔者工作所负责的推荐场景为例，绑核之前，tensorflow serving推理预测请求预测P99耗时毛刺尤为严重，最高可达300+ms，绑核之后推理预测请求P99耗时稳定在80+ms，再加上前面所讲的大请求拆分小请求，最终稳定在30ms左右。
* **定时伸缩容**。k8s的hpa伸缩容或许没那么灵敏，对于一些容易有突发流量的推荐服务，或者稳定性要求比较高的推荐服务，可针对业务高峰期和低峰期不同时间段做伸缩容。
* **节点做业务隔离**。不同业务场景节点应该隔离，特别是和一些资源要求比较高的业务，k8s部署的节点区分开来。笔者所在公司，就曾将推荐服务和其它资源要求高的业务服务部署混合在一起部署，推荐服务的pod所需的cpu资源竞争不过，导致cpu被严重节流，从而影响推荐服务的性能。