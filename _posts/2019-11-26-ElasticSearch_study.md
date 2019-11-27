---
layout:     post
title:      深入理解ElasticSearch
subtitle:   Mastering ElasticSearch学习笔记
date:       2019-11-26
author:     DCL
header-img: img/code/01.jpg
catalog: true
tags:
    - BigData
---
# 1、ElasticSearch简介 #

## Apache Lucene ##
ElasticSearch的创始人使用Apache Lucene创建索引的同时也用Lucene进行搜索。为什么用Apache Lucene而不上开发自己的全文检索库？Lucene的特点：成熟、高性能、可扩展、轻量级以及强大的功能。Lucene的内核可以创建为独立的Java库文件并且不依赖第三方代码，用户可以使用它提供的各种所见即所得的全文检索功能进行索引和搜索操作。Lucene还有很多扩展功能，如多语言处理、拼写检查、高亮显示等。

Lucene的总体架构：  

- 文档（document）    
- 字段(field)  
- 词项(term)  
- 词条(token)  

Apache Lucene将写入索引的所有信息组织成一种名为倒排索引（inverted index）的结构。该结构是一种将词项映射到文档的数据结构，其工作方式与传统的关系数据库不同。可理解为：倒排索引是面向词项而不上面向文档的。  
索引中还存储了很多其他信息，如词向量（为每个字段创建的小索引，存储该字段中所有的词条）、各字段的原始信息、文档删除标志等。  
每个索引由多个段（segment）组成，每个段指挥被创建一次但会被查询多次。索引期间，段经创建就不会再被修改。  
多个段会在一个叫做段合并（segment merge）的阶段被合并在一起，而且要么强制执行，要么由Lucene的那种机制决定在某个时刻执行，合并后段的数量更少，但是更大。  
段合并非常耗I/O。

文档中的数据转化为倒排索引，查询串转换为搜索的词项，这个过程成为分析（analysis）。文本分析由分析器来执行，而分析器由分词器（tokenizer）、过滤器（filter）和字段映射器（character mapper）组成。  
分词器用来将文本切割成词条。  
Lucene提供了很多现成的过滤器。  
ElasticSearch中有些查询会被分析，而有些则不会。例如，前缀查询（prefix query）不会被分析，而匹配查询（match query）会被分析。  
索引期与检索期的文本分析要采用同样的分析器，只有查询分析出来的词项与索引中词项能匹配上，才会返回预期的文档集。例如，如果在索引期使用了词干还原与小写转换，那么在查询期也应该对查询串做相同的处理，否则查询可能不会返回任何结果。

在Lucene中，一个查询通常被分割为词项与操作符。  
查询中也可以包含布尔操作符，用于连接多个词项，使之构成从句（clause）。AND，OR,NOT,+,-。  
在字段中查询，title:elasticsearch、title:(+elasticsearch+"mastering book")。  
出于性能考虑，通配符不能作为词项的第一个字段出现。  
除通配符之外，Lucene还支持模糊（fuzzy and proximity）查询，通过使用 ~ 字符以及一个紧随其后的整数值。  
使用 ^ 字符并赋以一个浮点数对词项加权（boosting），以提高该词项的重要程度。  
搜索特殊字符，通过反斜杠 \ 进行转义。  

## ElasticSearch ##
基本概念：索引（index）、文档（document）、映射（mapping）、类型（type）、节点（node）、集群（cluster）、分片（shard）、副本（replica）、网关（gateway）。  

当ElasticSearch节点启动时，它使用广播技术（也可配置为单播）来发现同一个集群中的其他节点（这里的关键是配置文件中的集群名称）并与他们连接。  
集群中会有一个节点被选为管理节点（master node）。该节点负责集群的状态管理以及在集群拓扑变化时做出反应，分发索引分片在集群的相应节点上。  
ElasticSearch是基于对等架构的，从用户角度，管理节点并不比其他节点重要，所有操作并不需要经过管理节点处理。  
ElasticSearch在内部也使用Java API进行节点间通信。  

ElasticSearch提供四种方式创建索引。  
最简单的方式是使用索引API，它允许用户发送一个文档至特定的索引。curl -XPUT http://localhost:9200/~ -d '{"title":"……","content":"……","tags":["1","2","3"]}'  
第二种或第三种方式允许用户通过bulk API或UDP bulk API来一次性发送多个文档至集群。两者的区别在于网络连接方式，前者使用HTTP协议，后者使用UDP协议，且后者速度快，但是不可靠。  
第四种方式使用插件发送数据，成为河流（river），河流运行在ElasticSearch节点上，能够从外部系统获取数据。  
建索引操作只会发生在主分片上，而不是副本上。

查询并不是一个简单的、单步骤的操作。一般来说，查询分为两个阶段：分散阶段（scatter phase）和合并阶段（gather phase）。分散阶段将query分发到包含相关文档的多个分片中去执行查询，合并阶段则从众多分片中收集返回结果，然后对他们进行合并、排序、后续处理，然后返回给客户端。

# 2、查询DSL进阶 #
## Apache Lucene 默认的评分公式 ##
文档得分依赖多个因子，除了权重和查询本身的结构，还包括匹配的词项数目，词项所在字段，以及用于查询规范化的匹配类型等。  
因子：文档权重（document boost）、字段权重（field boost）、协调因子（coord）、逆文档频率（inverse document frequency）、长度范数（length norm）、词频（term frequency）、查询范数（query norm）。  

TF/IDF公式：  
> score(q,d)=coord(q,d) * query_Boost(q) * [V(q) * V(d) / ∣V(q)∣] * lengthNorm(d) * docBoost(d)

Lucene实际评分公式：
> score(q,d)=coord(q,d) * queryNorm(q) * tinq∑(tf(t in d) * idf(t)2*boost(t) * norm(t,d))

## 查询改写 ##

查询改写操作，就是把费时的原始查询类型实例改写成一个性能更高的查询类型实例。
查询改写的属性：  
scoring_boolean：该选项将每个生成的词项转化为布尔查询中的一个或从句（should clause）。这种处理方法比较耗CPU（因为要计算和保存每个词项的得分），而且有些查询生成的词项太多从而超出了布尔查询的限制，默认为1024个从句。  
sonstand_score_boolean：该选项与前面提到的scoring_boolean类似，但是CPU消耗较小，这是因为该过程并不计算每个从句的得分，而是每个从句得到一个与查询权重相同的常数得分，默认情况下等于1。  
constand_score_filter：通过顺序遍历每个词项来创建一个私有的过滤器，标记跟每个词项相关的所有文档。  
top_terms_N：该选项将每个生成的词项转化为布尔查询中的一个或从句，并保存计算出来的查询得分。  
top_terms_boost_N：该选项与top_terms_N类似，不同之处在于该选项产生的从句类型为常量得分查询，得分为从句的权重。

## 二次评分 ##

ElasticSearch会截取查询返回文档的前N个，并使用预定义的二次评分方法来重新计算它们的得分。  
在rescore对象中的查询对象中，配置参数：  
window_size：窗口大小，该参数默认设置为from和size参数值之和。  
query_weight：查询权重值，默认为1，二次评分查询的得分在与原始查询得分相加之前，将乘以该值。  
rescore_query_weight：二次评分查询的权重值，默认等于1，二次评分查询的得分在与原始查询得分相加之前，将乘以该值。  
resocre_mode：二次评分的模式，默认设置为total。它定义了二次评分中文档得分的计算公式，可用的选项有total、max、min、avg和multiply。  

## 批量操作 ##

批量取（MultiGet）可以通过_mget端点（endpoint）操作，它允许使用一个请求获取多个文档。与实例获取功能类似，文档获取也是实时的，ElasticSearch会返回那些被索引的文档，而不论这些文档可用于搜索还是暂时对查询不可见。  
查询请求发送到_msearch端点。URL中的索引名及类型是可选的，并且会作为剩余输入行的默认参数。剩余行可用于存储搜索类型信息（search_type）以及查询执行的路由或提示信息（preference）。在某些特殊情况下，行中可以包含对象（{}）甚至行本身为空。

## 排序 ##

返回文档默认按文档得分降序排序。  
可以通过添加查询的sort部分的missing属性为那些section字段有缺失值的文档定制排序行为。  
基于每个文档的release_dates字段的最小值进行排序。mode参数可以设置为：min，max，avg，sum。
基于多值geo字段的排序。  
基于字段中定义的嵌套对象排序，使用两种情形：使用了显式嵌套映射（在映射中配置type="nested"）的文档以及使用了对象类型的文档。  
也可以使用nested_filter参数，只是该参数仅对嵌套文档有效（即那些被显式标记为嵌套文档的）。利用这个参数，可以在排序前就可已经通过一个过滤器在检索期排除了某些文档，而不是在检索结果文档集中过滤它们。

## 数据更新API ##

从API的角度来看，文档更新可以通过执行发送直送至端点的更新请求来实现，也可以通过在更新请求的url中添加_update参数来更新某个特定的文档，如：/libray/book/1/update。  
upsert属性允许用户在当URL中地址不存在时创建一个文档。  
通过设置ctx.op的值为delete来实现移除整个文档。

## 使用过滤器优化查询 ##

过滤器缓存（filter cache）  
并不是所有过滤器都默认被缓存，以下过滤器默认不缓存：
numeric_range  
script  
geo_bbox  
geo_distance  
geo_distance_range  
geo_polygon  
geo_shape  
and  
or  
not  
ElasticSearch允许用户通过设置_cache和_cache_key属性来开启或关闭过滤器的缓存机制。   
词项查找过滤器，为了使词项查找功能生效，需要先存储_source字段。
在elasticsearch.yml文件中配置属性：indices.cache.filter.terms.size、indices.cache.filter.terms.expire_after_access、indices.cache.filter.terms.expire_after_write。  
 
## ElasticSearch切面机制中的过滤器与作用域 ##

系统只在查询结果之上计算切面结果。如果你在filter对象内部且在query对象外部包含了过滤器，那么这些过滤器将不会对参与切面计算的文档产生影响。  
作用域（scope），它能扩充用于切面计算的文档。  
在切面类型的相同层级上使用切面过滤器（facet filter），能通过使用过滤器减少计算切面时的文档数，就像在查询时使用过滤器那样。  
不需要强制运行第二次查询，这是因为可以使用全局切面作用域（global faceting scope）来达成目的，并具体通过将切面类型的global属性配置为true来实现。  

# 3、底层索引控制 #
## 改变Apache lucene的评分公式 ##
可用的相似度模型：Okapi BM25模型、随机偏离（Divergence from randomness）模型、基于信息的（information based）模型。  
## 相似度模型配置 ##
选择默认的相似度模型；  
配置被选用的相似度模型：配置TF/IDF形似度模型、配置Okapi BM25形似度模型、配置DFR形似度模型、配置IB形似度模型。
## 使用编解码器 ##
可用的倒排表格式：default、pulsing、direct、memory、bloom_default、bloom_pulsing。
## 准实时、提交、更新及事务日志 ##
刷新操作是很耗资源的，因此刷新间隔时间越长，索引速度越快。如果需要长时间高速建索引，并且在建索引结束之前暂不执行查询，那么可以考虑将index.refresh_interval参数值设置为-1，然后在建索引结束以后再将该参数恢复为初始值。  
除了自动的事务日志刷新以外，也可以使用对应的API。也可以使用flush命令对特定的索引进行事务日志刷新。  
事务日志相关配置：index.translog.flush_threshold_period、index.translog.flush_threshold_ops、index.translog.flush_threshold_size、index.translog.disable_flush。  
## 数据处理 ##
输入并不总是进行文本分析。  
索引期更换分词器。  
可以定义default_index分析器和default_search分析器，并将它们作为索引期和检索期的默认分析器。  

## 控制索引合并 ##

段合并（segment merging）。 
 
- 当多个索引段合并为一个的时候，会减少索引段的数量并提高搜索速度。
- 同时也会减少索引的容量（文档数），因为在段合并时会移除被标记为已删除的那些文档。

三种可用的合并策略：tiered(默认)、log_byte_size、log_doc。  
索引合并调度器（scheduler）分为两种，并发合并调度器concurrent（默认）、顺序合并调度器serial。  
系统是8核，调度器允许的最大线程数可设置为4。
index.merge.scheduler.type:concurrent（使用并发合并调度器）  
index.merge.scheduler.type:serial（使用顺序合并调度器）  

# 4、分布式索引架构 #  
## 选择合适的分片和副本数 ##
ElasticSearch设计者选择的默认配置（5个分片和1个副本）使数据量增长和多分片搜索结果合并之间达到平衡。  
公式：节点最大数=分片数*(副本数+1)  
## 路由 ##
路由是优化集群的一个很强大的机制。它能让我们根据应用程序的逻辑来部署文档，从而可用用更少的资源构建更快速的查询。  
路由确保了在索引时拥有相同路由值的文档会索引到相同的分片上，但一个给定的分片上可以有很多拥有不同路由值的文档。路由可以限制查询时使用的节点数，但是不能替代过滤功能。这意味着无论一个查询有没有使用路由都应该使用相同的过滤器。  
别名能让我们像使用普通索引那样使用路由。 "alias":"别名"  
允许在一次查询中使用多个路由值。多路由值意味着在一个或多个分片上查询。  
## 调整默认的分片分配行为 ##
两种分片分配器（ShardAllocator）：even_shard、balanced（默认）。  
**even_shard**：这个分配器并排工作在索引级别。这意味着只要分片及其副本在不同的节点上，分配器就认为工作正常，而不关心来自同一索引的不同分片存放到哪里。  
**even_shard、balanced**：
可调整的参数如下所示:
cluster.routing.allocation.balance.shard:默认值为0.45.
cluster.routing.allocation.balance.index:默认值为0.5.
cluster.routing.allocation.balance.primary默认值为0.05.
cluster.routing.allocation.balance.threshold:默认值为1.0.
第一个参数告诉ElasticSearch每个节点都分配数量相近的分片对我们有多么重要。
第二个参数类似，但不是基于所有的分片数，而是基于同一个索引的分片数。
第三个参数告诉ElasticSearch将主分片平均分配到节点有多么重要。
最后，如果所有因子与其权重的乘积的总和大于已定义的阂值，那么这类索引上的分 片就需要重新分配了。而如果出于某些原因，你希望忽略一个或多个因子，只要把它们的权重设为0就可以了。

**裁决者(decider)：**  
ElasticSearch允许同时使用多个裁决者，而且它们会在决策过程中投票。这里有一个规则共识，例如，如果某裁决者投票反对重新分配一个分片的操作，那么该分片就不能移动。裁决者只有固定的十来个，如果想添加新的决策者，只能通过修改ElasticSearch源码。

**SameShardAllocationDecider**  
顾名思义，该裁决者禁止将相同数据的拷贝(分片和其副本)放到相同的节点上。注意属性 cluster.routing.allocation.same_shard.host属性。它控制了ElasticSearch 是否需要考虑分片放到物理机器上的位置。它默认为false，因为许多节点可能运行在同一台运行着多个虚拟机的服务器上。而当设置成true时，这个裁决者会禁止将分片和其副本 放置在同一台物理机器上。

**ShardsLimitAllocationDecider**  
ShardsLimitAllocationDecider确保一个给定索引在某节点上的分片不会超过给定 数量。这个数量是由index.routing.allocation.total_ shards_per_node属性控制的，可以在elasticsearch.yml文件中设置，或者通过索引更新API在线更新。属性的默认值是-1，表明没有限制。

**FiIterAllocationDecider**  
只在我们增加了控制分片分配的参数时才会用到。

**RepIicaAfterPrimaryActiveAllocationDecider**  
这个裁决者使得ElasticSearch仅在主分片都分配好之后才开始分配副本。

**ClusterRebalanceAllocationDecider**  
ClusterRebalanceAllocationDecider允许根据集群的当前状态来改变集群进行重新平衡 的时机。该裁决者可以通过cluster.routing.allocation.allow_rebalance属性来控制，它支持以下这些值:
indices_all_active :这个默认值表明重新平衡仅在集群中所有已存在的分片都分配好后才能进行。
indices_primaries_active:这个设置表明重新平衡只在主分片分配好以后才进行。
always:这个设置表明重新平衡总是可以进行，甚至在主分片和副本还没有分配好 时也可以。

注意这些设置在运行时不能更改。

**ConcurrentRebalanceAllocationDecider**  
ConcurrentRebalanceAllocationDecider用于调节重新部署操作，并基于cluster.routing. allocation.cluster_concurrent_rebalance属性。在该属性的帮助下，我们可以设置给定集群上可以并发执行的重新部署操作的数量，默认是2个，意味着在集群上只有不超过2个的分片可以同时移动。把这个值设为-1将没有限制。

**DisabIeAlIocationDecider**  
DisableAllocationDecider是一个可以调整自身行为来满足应用需求的裁决者。有以下配置
cluster.routing.allocation.disable_allocation:这个设置允许我们禁止所有的分配。
cluster.routing.allocaiotn.disable_new_allocation:这个设置允许我们禁止新主分片的分配。
cluster.routing.allocation.disable_replica_allocation:这个设置允许我们禁止副本的分配。
所有这些设置的默认值都是false。它们在你想要完全控制分配时非常有用。例如，当 你想要快速地重新分配和重新启动一些节点时，便可以禁止重新分配。另外请记住尽管你可以在elasticsearch.yml文件里设置前面提到的那些属性，但这里使用更新API会更有意义。

利用该配置可快速重启节点，而不进行分片分配。

**AwarenessAllocationDecider**  
AwarenessAllocationDecider是用来处理意识部署功能的。无论什么时候使用了cluster. routing.allocation.awareness.attributes设置，它都会起作用。

**ThrottlingAllocationDecider**  
ThrottlingAllocationDecider与前面讨论过的ConcurrentRebalanceAllocationDecider类 似，该裁决者允许我们限制分配过程产生的负载。我们可以使用下面的属性控制恢复过程:
cluster.routing.allocation.node_initial_primaries_recoveries:参数值默认为4。它描述了单节点所允许的最初始的主分片恢复操作的数量。
cluster.routing.allocation.node_concurrent_recoveries:参数值默认为2。它定义了单节点上并发恢复操作的数量。

**RebalanceOnlyWhenActiveAllocationDecider  **  
限制重新平衡过程仅在分片组（主分片和它的副本们）内的所有分片都是活动的（active）情况下进行。

**DiskThresholdDecider**  
DiskThresholdDecider在ElasticSearch的0.90.4版本里引人，它允许我们基于服务器上的空余磁盘容量来部署分片。默认它是禁用的，因而必须设置cluster.routing.allocation.disk.threshold_enabled属性为true来启用它。该裁决者允许我们配置一些阈值，以决定何时将分片放到某个节点上以及ElasticSearch应该何时将分片迁移到另一个节点上。
cluster.routing.allocation.disk.watermark.low：属性允许我们在分片分配可用时指定一个 阀值或绝对值。例如，默认值是0.7，这告诉Elasticsearch新分片可以分配到一个磁盘使用率低于70%的节点上。
cluster.routing.allocation.disk.watermark.high：属性允许我们在某分片分配器试图将分片 迁移到另一个节点时指定一个阀值或绝对值:‘默认值是0.85，意味着ElasticSearch会在磁盘空间使用率上升到85%时重新分配分片。
cluster.routing.allocation.disk.watermark.low和cluster.routing.allocation.disk.watermark. high这两个属性都可以设置成百分数(如0.7或0.85 )，或者绝对值(如1 OOOmb )。另外，本节提到的所有属性都可以在 elasticsearch.yml里静态设置，或使用ElasticSearch API动态更新。

## 调整分片分配 ##
在elasticsearch.yml文件里做如下设置:  
index.routing.allocation.total_shards_per_node:4  
这样，单个节点上最多会为同一个索引分配4个分片。  

**更多的分片分配属性：**

- cluster.routing.allocation.allow_rebalance:这个属性允许我们基于集群中所有分片的 状态来控制执行再平衡(rebanlance)的时机。可选的值包括:always、indices_primaries_active 、indices_all_active （默认）。
- cluster.routing.allocation.cluster concurrent_rebanlance:这个属性控制我们的集群内有多少分片可以并发参与再平衡处理，默认值为2。
- cluster.routing.allocation.node_initial_primaries_recoveries:这个属性指定了每个节 点可以并发恢复的主分片数量。基于主分片恢复通常比较快，即使把属性值设置得较大，也不会给节点本身带来过多的压力。这个属性的默认值为4.
- cluster.routing.allocation.node_concurrent_recoveries:这个属性指定了每个节点允 许的最大并发恢复分片数，默认值是2。
- cluster.routing.allocation.disable_new_allocaiton:默认值是false。该属性用来控制是否禁止为新创建的索引分配新分片(包括主分片和副本分片)。
- cluster.routing.allocation.disable_allocation:默认值是false。该属性用来控制是否禁 止针对已创建的主分片和副本分片的分配。请注意，将一个副本分片提升为主分片(如果主分片不存在)的行为不算分配。
- cluster.routing.allocation.disable_replica_allocation:这个属性默认是false。当它设 为true时，将会禁止将副本分片分配到节点。
前面所有的属性既可以在elasticsearch.yml文件里设置，也可以通过更新API来设置。然而在实践中，通常只使用更新API来配置一些属性，如- cluster.routing.allocation.disable_new_allocation、cluster.routing.allocation.disable_allocation、cluster.routing.allocation. disable_replica_allocation。

**应用我们的知识**
 
性能测试开源工具：

- ApacheJMeter
- ActionGenerator
- ElasticSearch paramedic的插件
- ElasticSearch BigDesk的插件 
 
避免单个节点上运行多个ElasticSearch实例  
将node.max_local_storage_nodes属性值设为1，是为了避免单个节点上运行多个ElasticSearch实例。

发现
在发现模块的配置中，我们仅仅设置了一个属性，即设置discovery.zen.minimum_master_nodes的属性值为3。它能指定组成集群所需的且可以成为主节点的最小节点数，取值至少是集群节点数的一半加1.

记录慢查询
协同ElasticSearch工作时，有件事情非常重要:记录那些执行时间大于等于某个阈值的查询。请注意，这个日志记录的不是查询的全部执行时间，而是查询在每个分片上执行的时间，这意味着日志记录的只是执行时间的一部分。

在例子里，我们想使用INFO级日志记录执行超过500毫秒的查询和执行超过1秒的实时读取操作。对于调试级日志，这里分别设置为100毫秒和200毫秒。下面是这部分内容的配置片段:

index.search.slowlog.threshold.query.info:500ms
index.search.slowlog.threshold.query.debug:100ms
index.search.slowlog.threshold.fetch.info:1s
index.search.slowlog.threshold.fetch.debug:200ms
记录垃圾回收器的工作情况
最后，由于我们在没有监控的情况下就开始了，因而想看看垃圾回收器表现如何。例如，想要搞清楚是否以及何时垃圾回收器消耗了太多的时间。为了实现这个目的，我们在elasticsearch.yml文件里加人下面几行内容:

monitor.jvm.gc.ParNew.info:700ms
monitor‘jvm.gc.ParNew.debug:400ms
monitor.jvm.gc.ConcurrentMarkSweep.info:5s
monitor.jvm.gc.ConcurrentMarkSweep.debug:2s
通过使用INFO级日志，当并发标记清除(concurrent mark sweep)执行等于或超过5秒时，以及新生代收集(younger generation collection)执行超过700毫秒时，ElasticSearch会记录垃圾回收器工作超时的信息。同时也加上了DEBUG级日志，用来应对我们想要调试或者修复问题的情况。

内存设置
通常的建议是不要使Java虚拟机堆的大小超过可用内存总量的50%。
在前面所展示的配置中，我们设置了ElasticSearch记录垃圾回收器的信息，然而为了长期监控，你可能还需要使用类似SPM ( http://sematext.com/spm/index.html)或Munin(http://munin-monitoring.org/)的监控工具。

