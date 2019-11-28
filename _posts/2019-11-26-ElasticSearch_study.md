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

# 5、管理ElasticSearch #
## 选择正确的目录实现-存储模块 ##
存储模块是一个在配置集群时容易被忽视的模块，然而它非常重要。该模块允许用户控制索引的存储方式，例如，可以持久化存储(存储在磁盘上)或非持久化存储(存储在内存中)。  
ElasticSearch提供了4种可用的存储类型。通过index.store.type属性进行设置：  
**简单文件系统存储（simplefs）**  
最简单的目录类的实现，它使用一个随机读写文件(Java RandomAccessFile)进行文件操作，并与Apache Lucene的SimpleFSDirectory类对应。对于简单的应用来说它是够用的，但瓶颈主要是在多线程访问上，即性能非常差。  
**新I/O文件系统存储（niofs）**  
该存储类型使用基于java.nio.FileChannel 的目录实现，与Apache Lucene的NIOFSDirectory类对应。该实现允许多个线程在不降低性能的前提下访问同一个文件。如果想使用这个存储类型，需要将index.store.type属性设为niofs  
**MMap文件系统存储（mmapfs）**  
该存储类型使用了Apache Lucene的MMapDirectory 实现。它使用mmap系统命令来读取和随机写人文件，并从进程的虚拟地址空间的可用部分中分出与被映射文件大小相同的空间，用于装载被映射文件。它没有任何锁机制，因此非常适合多线程访问。  
需要注意的是，MM ap文件系统存储在64位环境下工作最佳，对于32位系统，只有你确信索引足够小，且虚拟地址空间足够大时才可以使用。  
**内存存储（memory）**  
内存存储是唯一一个不是基于Apache Lucene目录的存储类型，它允许我们把全部索引都保存在内存中，因此文件并没有存储在硬盘上。  
当使用内存存储类型（memory）时，我们也能在一定程度上控制缓存，这一点非常重要。注意以下设置都是节点级别的:  
cache.memory.direct:定义内存存储是否应该被分配到Java虚拟机堆内存之外，默认为true。一般情况下应该保持为默认值，从而能避免堆内存过载。  
cache.memory.small_buffer_size:定义小缓冲区的大小，默认值是1KB。小缓冲区是用来容纳段(segment )信息和已删除文档信息的内部内存结构。  
cache.memory.large_buffer_size:定义大缓冲区的大小，默认值是1MB。大缓冲区是用来容纳除段信息和已删除文档信息外的索引文件的内部内存结构。  
cache.mernory.small_cache_size:定义小缓存的大小，默认值是10MB。小缓存是用来缓存段信息和已删除文档信息的内部内存结构。  
cache.memory.large_cahce_size:定义大缓存的大小，默认值是500MB。大缓存是用来缓存除段信息和已删除文档信息外的索引文件的内部内存结构。  
**默认存储类型**  
ElasticSearch默认使用基于文件系统的存储类型。虽然针对不同的操作系统往往会选择不同的存储类型，但终究都使用了基于文件系统的存储类型。例如，ElasticSearch在32位的Windows系统上使用simplefs类型，在Solaris和64位的Windows系统上使用mmapfs，其他系统则使用niofs。

## 发现模块的配置 ##

Zen发现(Zen discovery)是ElasticSearch自带的默认发现机制。Zen发现默认使用 多播来发现其他的节点。  

有时多播会由于各种原因而失效，或者在一个大型集群中使用多播发现会产生大量不 必要的流量，这可能都是不想使用多播的合理理由。在这些情况下，Zen发现使用了第二种发现方法:单播模式。  
**多播**  
多播(Multicast)是ElasticSearch的默认模式。当节点还没有加人任何集群时(如节点刚刚启动或重启)，它会发出一个多播的ping请求，这相当于通知所有可见的节点和集群，它已经可用并准备好加入集群了。

Zen发现模块的多播部分有如下设置:

- discovery.zen.ping.multicast.address:通信接口，可赋值为地址或接口名。默认值是 所有可用接口。  
- discovery.zen.ping.multicast.port:通信端口，默认为54328。
- discovery.zen.ping.multicast.group:代表发送多播消息的地址，默认为224.2.2.4。
- discovery.zen.ping.multicast.buffer_size:缓冲区大小，默认为2048
- discovery.zen.ping.multicast.ttl:定义多播消息的生存期，默认是3。每次包通过路 由时，TTL就会递减。这样可以限制广播接收的范围。请注意，路由数的阈值设置可参考TTL的值，但要确保TTL的值不能恰好等于数据包经过的路由数。
- discovery.zen.ping.multicast.enabled;默认是true。如果你打算使用单播方式而需要 关闭多播时，可将其设置为false。  

**单播**  
像前面描述的那样关闭了多播后，你就可以安全地使用单播(Unicast)了。当节点不是集群中的一部分时(如刚刚重启，启动或由于某些故障脱离集群)，它会发送一个ping请求给配置文件所指定的那些地址，通知所有的节点它准备好要加人集群了。

单播的配置非常简单，如下所示:

- discovery.zen.ping.unictas.hosts:代表集群中的初始节点列表，可称之为一个列表或主 机数组。每个主机可以指定一个名称(或者IP地址)，还可以追加一个端口或端口范 围。例如，属性值可以是这样:["master1", "master2:8181", "master3[80000-81000]"]。
一般来说，单播发现的主机列表不需要是集群中所有ElasticSearch节点的完整列 表，因为新节点一旦与列表中任何一个节点相连，就会知晓组成集群的其他全部节点的信息。
- discovery.zen.ping.unicast.concurrent_connects:定义单播发现使用的最大并发连接 数。默认是10个。
**最小主节点数**
对发现模块来说，一个最重要的属性是discovery.zen.minimum_master_nodes属性。它 允许我们设置构建集群所需的最小主节点(master node)候选节点数。这让我们避免了由于某些错误(如网络问题)而出现令人头疼的局面(即多个集群同名)。

- discovery.zen. minimum_master_nodes:设置为大于等于集群节点数的一半加1  

**Zen发现错误检测**  
ElasticSearch在工作中执行两个检测流程。第一个流程是由主节点向集群中其他节点 发送ping请求来检测它们是否工作正常。第二个流程刚好相反，由每个节点向主节点发送请求来验证主节点是否正在运行并能履行其职责。然而，如果网络速度很慢，或者节点部署在不同的地点，那么默认的配置也许就不合适了。  
因此ElasticSearch的发现模块提供了以下可以修改的设置:

- discovery.zen.fd.ping_interval:设置节点向目标节点发送ping请求的频率，默认1秒。  
- discovery.zen.fd.ping_timeout:设置节点等待ping请求响应的时长，默认30秒。如 果节点使用率达到100%或者网速很慢，可以考虑增大该属性值。  
- discovery.zen.fd.ping_retries:设置当目标节点被认为不可用之前ping请求的重试次 数，默认为3次。如果网络丢包比较严重，可以考虑增大该属性值，或者修复你的网络。  

**亚马逊EC2发现**  
**可配置的属性**  
ElasticSearch 的节点可以扮演不同的角色，它们可以作为数据节点(存储数据的节点)，也可以作为主节点，其中主节点(一个集群中只有一个)除了处理查询请求外，还负责集群管理。当然节点也可以配置为既不是主节点也不是数据节点，在这种情形下，该节点将只作为执行用户查询的聚合节点。  ElasticSearch默认每个节点都是数据节点和候选主节点，但是这是可以改变的。为了取消某节点的主节点候选资格，你需要在elasticsearch.yml文件中将node.master属性设为false。为了让某节点成为非数据节点，你需要在elasticsearch.yml文件中将node.data属性设为false。  

除此之外，ElasticSearch还允许我们使用下面的属性来控制网关模块的行为:  

- gateway.recover_after_nodes:该属性为整数类型，表示要启动恢复过程集群中所需 要的节点数。例如，当该属性值为5时，如果要启动恢复过程，集群中就至少要有5个节点(无论它们是候选主节点还是数据节点)存在。
- gateway.recover_after_data_nodes:该属性允许我们设置当恢复过程启动时，集群中 需要存在的数据节点数。
- gateway.recover_after_master_nodes:该属性允许我们设置当恢复过程启动时，集群 中需要存在的候选主节点数。
- gateway.recover_after_time:该属性允许我们设置在条件满足后，启动恢复过程前 需要等待的时间。 (如果我们希望延迟恢复过程直到所有数据节点处于可用状态后至少3分钟才开始 )。

**本地网关**  
本地网关类型使用节点上可用的本地存储来保存元数据、映射和索引。为了使用本地网关，需要有充足的磁盘空间来容纳数据(数据全部写人磁盘，而非保存在内存缓存中)。

本地网关的持久化不同于其他当前存在(但是已经弃用)的网关类型。向本地网关的写操作是以同步的方式进行的，以确保在写人过程中没有数据丢失。

默认gateway.type=local

备份本地网关，例如当升级集群时，希望出错后能够进行回滚。为了实现这个目的，你需要执行下面的操作:

- 停止向ElasticSearch集群索引数据(这意味着停止:fiver或其他向ElasticSearch发 送数据的外部应用)。
- 使用清空(Flush ) API清空所有尚未索引的数据。
- 为分配在集群中的每个分片创建至少一个备份，这至少可以保证一旦发生问题能找 回数据。然而，如果你希望操作能尽可能的简单，那么可以拷贝集群中每个数据节点上的完整数据目录。

**恢复配置**  
集群级的恢复配置：恢复配置大多针对的是集群级别，它允许我们设置恢复模块使用的通用规则，可设置 以下属性:

- indices.recovery.concurrent_streams:这个属性指定了从分片源恢复分片时允许打开 的并发流的数量，默认是3。较大的值会给网络层带来更大的压力，但恢复过程会更快，这依赖于网络的使用情况和吞吐量。
- indices.recovery.max_bytes_per_sec:这个属性指定了在分片恢复过程中每秒传输 数据的最大值，默认是20MB。如果想取消数据传输限制，需要把这个属性设为0。与并发流属性类似，该属性允许我们控制恢复过程中网络的使用。把它设为较大的值会带来较高的网络利用率，而且恢复过程会更快。
- indices.recovery.compress:这个属性默认为true，用来指定ElasticSearch是否在恢 复过程中压缩传输的数据。设为false可以降低CPU的压力，但同样会导致更多的网络数据传输。
- indices.recovery.file_chunk_size:这个属性指定从源分片向目标分片拷贝数据时数 据块(chunk)的大小。默认值为512KB而如果将indices.recovery.compress属性设为true的话，该值也会被压缩。
- indices.recovery.translog_ops:这个属性值默认为1000，指定在恢复过程中分片间 传输数据时，单个请求里最多可以传输多少行事务日志。
- indices.recovery.translog_size:这个属性指定从源分片拷贝事务日志时使用的数据 块的大小。默认值为512KB，且如果indices.recovery.compress属性设为true的话， 该值还会被压缩。

所有前面提到过的设置都可以通过集群的更新API来更新，或者在elasticsearch.yml文件里设置。

## 索引段统计 ##  
ElasticSearch提供了segments API，我们可以通过向 _segments REST端点发送HTTP GET请求来访问它。例如，要查看集群中所有索引的所有段信息，应当执行下面的命令:  
curl -XGET 'localhoat:9200/_segments'  
指定特定索引mastering   
curl -XGET 'localhoat:9200/mastering/_segments'  

## 理解ElasticSearch缓存 ##  
**过滤器缓存**  
过滤器缓存是负责缓存查询中使用的过滤器的执行结果的。   
在ElasticSearch中有两种类型的过滤器缓存:索引级和节点级，即我们可以选择配置索引级或节点级(默认选项)的过滤器缓存。  
不建议索引级过滤器缓存，由于我们不一定能预知给定索引会分配到哪里 （实际上是指索引的分片和副本)，进而无法预测内存的使用，所以不建议使用索引级的过滤器缓存。  

**索引级过滤器缓存的配置**  
ElasticSearch允许我们使用下面的属性来配置索引级过滤器缓存的行为:

- index.cache.filter.type:这个属性设置缓存的类型，我们可以使用resident , soft , weak或node(默认值)。在resident缓存中的记录不能被JVM移除，除非我们想移除它们(通过使用API，设置最大缓存值，或者设置过期时间)，并且也是因为这个原因而推荐使用它(填充过滤器缓存代价很高)。内存吃紧时，JVM可以清除soft和weak类型的缓存，区别是在清理内存时，JVM会优先清除weak引用对象，然后才是soft引用对象。最后的node属性代表缓存将在节点级控制。
- index.cache.filter.max_size:这个属性指定能存储到缓存中的最大记录数(默认 是-1，代表无限制)。需要注意这个设置不是应用在整个索引上，而是应用于指定索引的某个分片的某个索引段上，所以内存的使用量会因索引的分片数和副本数以及索引中段数的不同而不同。通常来说，结合soft类型，使用默认无限制的过滤器缓存就足够了。谨记慎用某些查询以保证缓存的可重用性。
- index.cache.filter.expire:这个属性指定过滤器缓存中记录的过期时间，默认是-1, 代表永不过期。如果我们希望对过滤器缓存设置超时时长，那么可以设置最大空闲时间。例如，希望缓存在最后一次访问后再过60分钟过期，就应当设置该属性值为60m。  
**节点级的过滤器缓存配置**  
节点级过滤器缓存是默认的缓存类型，它应用于分配到给定节点上的所有分片(设置 index.cache.filter.type属性为node，或者不设置这个属性)。ElasticSearch允许我们使用indices.cache.filter.sixe属性来配置这个缓存的大小，既可以使用百分数。  
节点级过滤器缓存是LRU类型(最近最少使用)缓存，这意味着为了给新记录腾出空 间，在删除缓存记录时，使用次数最少的那些会被删除。  

**字段数据缓存**  
字段数据缓存在我们的查询涉及切面计算或基于字段数据排序时使用。ElasticSearch所做的是加载相关字段的全部数据到内存中，从而使ElasticSearch能够快速地基于文档访问这些值。需要注意的是，从硬件资源的角度来看，构建字段数据缓存代价通常很高，因为字段的所有数据都需要加载到内存中，这需要消耗I/O操作和CPU资源。特别注意高基数的字段(拥有大量不同词项的字段)。

索引级字段数据缓存配置

index.fielddata.cache.type：属性为resident或soft ,node(默认)，跟我们在描述过滤器缓存时讨论过的情形类似 。
不建议使用索引级字段数据缓存，与索引级过滤器缓存类似，我们并不建 议使用它。原因就是很难预测哪个分片或索引会分配到哪个节点上，因此我们无法预估缓存每个索引需要的内存大小，而这会带来内存使用方面的问题。

节点级字段数据缓存配置

- index.fielddata.cache.size:这个属性指定了字段数据缓存的最大值，既可以是一个 百分比的值，如20%，也可以是一个绝对的内存大小，如10GB。
- index.fielddata.cache.expire:这个属性指定字段数据缓存中记录的过期时间，默认 值为-1，表示缓存中的记录永不过期。
过滤
除了前面提到的配置项以外，ElasticSearch还允许我们选择性地将某些字段值加载到 字段数据缓存中。这在某些情况下非常有用，尤其是在做基于字段数据排序或切面计算时。 ElasticSearch支持两种类型三种形式的字段数据过滤，即基于词频、基于正则表达式，以及基于两者的组合。

为了引人字段数据缓存过滤信息，需要在映射文件的字段定义部分额外添加两个对象: fielddata对象及其子对象filter。

基于词频过滤

- min/max: 基于词频过滤的结果是只加载那些频率高于指定最小值且低于指定最大值的词项，其 中词频最小值和最大值分别由min和max参数指定。词项的频率范围不是针对整个索引的。
- min_segment_size: 这个属性指定了在构建字段数据缓存时，索引段应满足的最小文档数，小于该文档数的索引段不会被考虑。
例如，只想保存来自容量不小于100的索引段，且词频在段中介于1%和20%之间的 词项到字段数据缓存中，那么可以进行如下字段映射:
    
    {
    "book":{
    "properties":{
     "tag":{
      "type":"string",
      "index":"not_analyzed",
      "fielddata":{
      "filter":{
       "frequency":{
    "min":0.01,
    "max":0.2,
    "min_segment_size":100
        }
       }
      }
     }
     }
     }
    }
  

基于正则表达式过滤  
除了基于词频的过滤，也可以基于正则表达式过滤。这时只有匹配特定正则表达式的 词项会加载到字段数据缓存中。

    {
    "book":{
    "properties":{
     "tag":{
     "type":"string",
     "index":"not_analyzed",
      "fielddata":{
       "filter":{
       "regex":"^#.*"
    }
    }
    }
    }
    }
    }
    

基于词频过滤和基于正则表达式过滤同时使用  
这是一种and的关系。同时符合这两种情况的词项才会被缓存。注意，没有被缓存的词项，在涉及切面计算或基于字段数据排序时是查不到的！要特别注意！

查询期间重建字段数据缓存  
字段数据缓存虽然不是在索引期间构建的，但却可以在查询期间重建，于是我们可 以在运行时改变过滤行为，并具体通过使用映射API更新fielddata配置节来实现。 然而，需谨记在改变字段数据缓存过滤设置后清空缓存，这可以通过使用清理缓存 API来实现。  

**清除缓存（_cache端点）**  
前面已经提到，在改变字段数据过滤以后需要清除缓存，这点很关键。  
单一索引缓存、多索引缓存和全部缓存的清除  
清空全部缓存的最简单的做法是执行下面的命令:  
curl -XPOST 'localhoat:9200/_cache/clear'  
针对特定索引  
curl -XPOST 'localhoat:9200/mastering/_cache/clear'  
清除特定缓存  
清除缓存除了前面提到的方法，我们也可以只清除一种指定类型的缓存。下面列出的 就是可以被单独清除的缓存类型:  

- filter:这类缓存可以通过设置filter参数为true来清除。反之为了避免清除缓存， 需要设置filter参数为false。
- field_data:这类缓存可以通过设置field data参数为true来清除。反之为了避免清 除这种缓存，需要设置field data参数为false。
- bloom:为了清除bloom缓存(如果某种倒排索引格式中使用了Bloom_filter，则可 能会使用这种缓存，参考3.3节)，bloom参数需要设置为true。反之为了避免清除这种缓存，需要设置bloom参数为false。  
例如，要清除mastering索引的字段数据缓存，并保留filter缓存和bloom缓存，则可 以执行下面的命令:  
curl -XPOST 'localhost:9200/mastering/_cache/clear?field_data=true&filter=false&bloom=false'  
清除字段相关的缓存  
除了清除全部或特定的缓存，我们还可以清除指定字段的缓存。为了实现这个，我们需要的请求中增加fields参数，参数值为所要清除缓存的相关字段名，多个字段名用逗号分隔。例如，要清除mastering索引里title和price字段的缓存:  
curl -XPOST 'localhost:9200/mastering/_cache/clear?fields=title,price'

# 6、故障处理 #

## 垃圾回收器 ## 
**Java内存**
JVM的内存空间分为以下区域:

- Eden区(Eden space): JVM初次分配的大部分对象都在该区域内。
- Survivor区(Survivor space):这块区域存储的对象是对Eden区进行垃圾回收后仍然存活的对象。Survivor区分为两部分:Survivor 0区和Survivor 1区。
- 年老代(Tenured generation):这块区域存储的是那些在Surviror区存活较长时间的对象。
- 持久代(Permanent generation ):这是一块非堆空间，用来存储所有JVM自身的数据，如Java类、对象方法等。
- 代码缓存区(Code cache ):这是HotSpot JVM中存在的一块非堆空间，用来编译、存放本地原生代码。
上述分类方法可以进一步简化:Eden区和Survivor区可以合称为年轻代(Young generation)。
Java对象的生命周期和垃圾回收
为了考察垃圾回收器的工作过程，我们来看一个简单Java对象的生命周期。  
Java程序将新创建的对象放置在年轻代的Eden区。如果年轻代执行下一次垃圾回收时，这个对象仍然存活(一般来说，它不是一次性对象，Java程序仍然需要用到它)，那么它会被挪到Survivor区(先进人Survivor 0区，如果经过年轻代的下一轮垃圾回收仍然存活，则它会被挪到Survivor 1区)。

当对象在Survivor 1区存活一段时间后，会被挪到年老代，成为年老代对象的一员，且从此以后，年轻代的垃圾回收不再对它起作用，因而该对象会一直在年老代中生存下去直到Java程序不再使用它。在这种情况下，如果它不再使用，那么在下次做全局垃圾回收时，会把它从堆空间中移除，并回收空间给新对象使用。

基于上面的介绍，我们可以断定(实质上也是如此):截至目前Java还在使用分代的垃圾回收机制;对象经历垃圾回收次数越多，越容易往年老代迁移。因此我们可以说，存在两种垃圾回收器在并行运行:年轻代垃圾回收器(也称为次垃圾回收器)和年老代垃圾回收器(也称为主垃圾回收器)。

**处理垃圾回收问题**  
打开垃圾回收日志  
monitor.jvm.gc.ParNew.warn:1000ms  
monitor.jvm.gc.ParNew.info:700ms  
monitor.jvm.gc.ParNew.debug:400ms  
monitor.jvm.gc.ConcurrentMarkSweep.warn:10s  
monitor.jvm.gc.ConcurrentMarkSweep.info:5s  
monitor.jvm.gc.ConcurrentMarkSweep.debug:2s  
使用jstat  
使用jstat检查垃圾回收器的工作状况，只需输人如下简单命令:  
jstat -gcutil 123456 2000 1000  
其中，-gcutil开关用来告知jstat监控垃圾回收器的工作状态;123456用于标识ElasticSearch所在的JVM ; 2000表示抽样间隔，单位是毫秒;1000表示的抽样数。

接下来，我们看一个关于jstat命令输出的示例:


首先阐述一下每一列的含义:

- S0:表示survivor 0区的使用率，用百分比表示。
- S1:表示survivor 1区的使用率，用百分比表示。
- E:表示Eden区的使用率，用百分比表示。
- O:表示年老代的使用率，用百分比表示。
- YGC:表示年轻代垃圾回收事件的次数。
- YGCT:表示年轻代垃圾回收耗时，单位为秒。
- FGC:表示年老代垃圾回收事件的次数。
- FGCT:表示年老代垃圾回收耗时，单位为秒。
- GCT:表示垃圾回收总耗时。  
如果你遇到以下情况，ElasticSearch运行不正常，或者S0, S1, E列的值达到100%，并且垃圾回收工作对这些堆空间不起作用，那么原因可能是:
年轻代太小了，你需要把它调大一些(当然，前提是拥有足够的物理内存);内存出问题了，比如说因为一些资源没有释放占用的内存而导致内存泄露。还有一种情况，如果年老代使用率达到100%且垃圾回收器多次尝试仍无法释放它的空间，这大概意味着没有足够的堆空间让ElasticSearch节点正常运作了。此时，如果你不想改变索引结构，就只能通过增大运行ElasticSearch的JVM的堆空间来解决了。

生成Memory Dump（jmap）  
JVM还拥有把堆空间转储到文件的能力。Java允许我们获取特定时间点的一个内存快照，并通过分析快照的存储内容，从而发现问题。  

jmap -dump:file=heap.dump 123456  
123456表示需要转储的Java进程号。-dump:file=heap.dump指定转储的目标文件为heap.dump。

**在类UNIX系统中避免内存交换**  
内存交换是一个把内存中的页( page)写人外存磁盘(Linux系统中指swap分区)的过程，且发生在物理内存不足时，或者操作系统由于某些原因需要把部分内存数据写人磁盘时。

使用ElasticSearch时，我们要确保它的内存空间不会被换出。可以想象一下，如果让ElasticSearch的部分内存交换到磁盘中，紧接着读取这块被换出的数据，就会对查询和索引的性能造成负面影响。因此ElasticSearch允许我们关闭针对它的内存交换，具体通过设置elasticsearch.yml的bootstrap.mlockall为true来实现。

除了前面的设置，我们还需要确保JVM的堆大小固定。要做到这一点，我们需要设置Xmx和Xms参数为相同值(或者设置ES MAX MEM和ES MIN MEM为相同值)。仍需谨记，要有足够的物理内存来支持以上设置。

此时运行ElasticSearch，可以看到如下日志:  
Unknown mlockall error 0

这个错误意味着内存锁定未起作用，因而我们还需修改两个系统文件(需要系统管理员权限)。在做修改之前，了段定运行ElasticSearch服务的用户为elasticsearch。  
首先修改/etc/security/limits.conf文件，添加如下两行记录:  
elasticsearch - nofile 64000  
elasticsearch - memlock unlimited  
然后修改/etc/pam.d/common-session文件，添加一行:  
session required pam_limits.so  
重新用elasticsearch用户登录后，再次运行ElasticSearch，就不会看到mlockall error的日志了。

## 关于I/O调节 ##
**控制IO节流**  
索引合并的过程是异步的，从Lucene的角度看是不会干扰索引和查询过程 的。然而，这很可能会出现问题，因为合并操作非常消耗I/O，需要先读取旧索引段，然后合并写人新索引段中。如果在此同时进行查询和索引，那么I/O子系统的负荷会非常大，这个问题在那些I/O速度较慢的系统中表现得尤为突出。这就是I/O节流的切入点。我们可以控制ElasticSearch使用的I/O量。
 
**配置**  
在节点级和索引级都可以配置I/O节流。这意味着你可以分别配置节点和索引的资源使用量。  
节流类型  
在节点级配置节流，indices.store.throttle.type属性。它支持none , merge , all这三个属性值。  
none为默认值，表示不作任何限制;  
merge表示在节点上进行索引合并时 限制I/O使用量;  
AlI表示对所有基于存储模块的操作都做I/O限制。  
在索引级配置节流，可以使用index.store.throttle.type属性，还支持一个默认的node属性值，即表示使用节点级配置取代索 引级配置。  
每秒最大吞吐量  
索引级的配置， 可以使用index.store.throttle.max_bytes_per_sec属性。  
节点级的配置，则可以使用 indices.store.throttle.max_bytes_per_sec属性。  
以上配置都可以通过elasticsearch.yml文件配置，也可以动态更新: 使用集群更新设 置接口来更新节点级配置，使用索引更新设置接口来更新索引级配置。

示例

- 节点级别（整个集群所有节点）设置  
curl -XPUT 'localhost:9200/_cluster/settings' -d '{
  "persistent":{
      "indices.store.throttle.type":"merge",
      "indices.store .throttle.max_bytes_per_sec":"50mb"
  }
}'   
- 索引级别设置   
curl -XPUT 'localhost:9200/payments/_settings' -d '{
    "index.store.throttle.type":"merge",
    "index.store.throttle.max_bytes_per_sec":"10mb"
}'   
- 使得设置生效   
curl -XPOST 'localhost:9200/payments/_close'
curl -XPOST 'localhost:9200/payments/_open'   
- 检查是否生效   
curl -XGET 'localhoat:9200/payments/_settings?pretty'


## 用预热器提升查询速度 ##
**为什么使用预热器**  
ElasticSearch需要提前加载一些数据到缓存中，目的是使用一些如父-子关系、切面计算和基于字段的排序等特定功能。预加载过程需要花费一些时间和资源，在某些时候会使查询变慢。更要命的是，如果索引频繁更新，缓存就需要频繁刷新，而查询性能也就更糟了。

这也是ElasticSearch 0.20版本引人预热器API的原因。预热器是一些标准查询，这些查询在ElasticSearch尚未对外提供查询服务时，先在冷的(尚未使用的)索引段上执行。查询操作不仅会在ElasticSearch启动时执行，在新索引段提交后也会执行。

**操作预热器**  
ElasticSearch允许我们创建、检索和删除预热器。每个预热器都关联了一个索引，或 者索引和类型的组合。我们在如下场合引入预热器:

创建索引的请求中携带预热器;  模板中包含预热器;使用PUT预热器API创建预热器。  
当然也可以完全禁用所有预热器而不是 先删除它们，所以在不需要它们时，可以简单地禁用它们。

使用PUT Warmer API (_warmer 端点)   
添加预热器最简单的方法是使用PUT Warmer API。为此我们需要给_warmer REST端 点发送一个带查询的HTTP PUT请求。  

curl -XPUT 'localhost:9200/mastering/doc/_warmer/testWarmer' -d '{
  "query":{
    "match all":{}
  },
  "facets":{
    "nameFacet":{
        "terms":(
              "field":"name"
        }
    }
  }
}’  
每个预热器都有唯一的名称(如本例中的testWarmer)。我们可以使用这个名称来检索和删除它。  

也可以针对整个mastering索引建立预热器。  

在创建索引时添加预热器   
我们需要在请求体中和mappings配置节同一层级的地方添加一个warmers配置节。

curl -XPUT 'localhost:9200/mastering' -d '{ 
    "warmers":{
        "testWarmer":{
            "types":[  "doc" ], 
            "source":{
                "query":{
                    "match_all":{ } 
                },
                "facets":{
                    "nameFacet":{
                        "terms":{
                            "field":"name"
                        }
                    }
                }
            }
        },
        "mappings":{
            "doc":{
                "properties":{
                    "name":{
                        "type":"string",
                        "store":"yes",
                        "index":"analyzed"
                    }
                }
            }
        }
    }
}'  

在模板中添加预热器

curl -XPUT 'localhost:9200/_template/templateone' -d '{ 
    "warmers":{
        "testWarmer":{
            "types":[
                "doc"
            ],
            "source":{
                "query":{
                    "match_all":{ } 
                },
                "facets":{
                    "nameFacet":{
                        "terms":{
                            "field":"name"
                        }
                    }
                }
            }
        }
    },
    "template":"test*"
}'

检索预热器   
可以通过名称来检索某个预热器:   
curl -XGET 'localhost:9200/mastering/_warmer/warmerOne' 

通配符检索名字带有特定前缀的预热器 :   
curl -XGET 'localhost:9200/mastering/_warmer/w*' 

还可以获取某个索引下的所有预热器:  
curl -XGET 'localhost:9200/mastering/_warmer/' 

删除预热器：   
与检索预热器类型，不过使用-XDELETE代替-XGET。

禁用预热器：  
预热器如果暂不使用，又不想删除，则可以禁用它们，只需将index.warmer.enabled属 性设置为false即可。这个属性可以在elasticsearch.yml文件中设置，也可以通过更新设置API设置，例如:  
curl -XPUT 'localhost:9200/mastering/_settings' -d '{
  "index.warmer.enabled":false
}'   

## 热点线程 ##  
当集群变慢且占用较多CPU资源时，你有必要采取一些处理措施来恢复它。然而，热 点线程API就提供了必要的信息，帮助你找到问题根源。热点线程指CPU占用高且执行时间较长的Java线程，而热点线程API可以返回如下信息:从CPU视角看到的ElasticSearch 中执行最频繁的代码段，以及ElasticSearch卡在什么地方。

检查所有节点上的热点线程:   
curl localhost:9200/_nodes/hot_threads  

热点线程API支持如下参数:  

- threads(默认值为3):经分析后输出的线程数。ElasticSearch会根据type参数指定 的信息挑选出最“热”的threads个线程。
- interval(默认值SOOms ) : ElasticSearch需要分两次检查线程，目的是计算特定线程与type参数对应操作的耗时百分比。两次检查的间隔时间由interval参数设置。
- type(默认值为cpu):本参数确定了要检查的线程状态的类型，具体支持如下状 态类型:指定线程的CPU耗时(cpu), BLOCK状态耗时(block), WAITING状态耗时(wait )。
- snapshots(默认值10):堆栈轨迹快照的数量。其中，堆栈轨迹指特定时间点的嵌 套函数调用。
例如：   
curl `localhost:9200/_nodes/hot_threads?type=wait&interval=1s'

## 现实场景 ##  
收集一些统计信息，执行下面命令:   
curl 'localhost:9200/_stats?pretty'

查看集群状态:    
curl localhoat:9200/_cluster/state?pretty


负载均衡   
回顾一下这些知识，然后再判断应该给ElasticSearch节点 安排哪个角色:   
node.data    node.master	  说明   
true    true    节点可以存储索引数据、可以被选为主节点，可以处理查询请求   
true	false	节点可以存储索引数据，但永远不会被选为主节点， 可以处理查询请求  
false	true	节点不存储索引数据，但可以被选为主节点，也可以处理查询请求   
false	false	节点永远不会存储索引数据，也不会被选为主节点，但可以处理查询请求-聚合器节点   
 
若大量使用了切面计算，因此我们决定将其中一部分节点分离出来，命名为聚合器节点，即该节点不持有数据，不做主节点，只负责处理查询请求:多亏了这类节点，我们可以把请求只发送给它们而不会给数据节点带来聚合操作的压力。总之，这意味着给予了数据节点处理更多索引请求的能力。

# 7、改善用户搜索体验 #

## 改正用户拼写错误 ##
**suggesters**  
在我们继续进行查询和分析响应结果之前，先简单交代一下可用的suggester类型。 ElasticSearch目前允许我们使用三种suggesters，即term suggester, phrase suggester和autocomplete suggester。前两种suggester可以用来改正拼写错误，而第三种suggester用来开发出迅捷且自动化的补全功能。从ElasticSearch的0.90.3版本开始，我们可以使用基于前缀匹配的Suggester来便捷地实现自动补全功能。


## 改善查询相关性 ##
首先我们将执行一个简单的查询并返回想要的结果，然后修改这个查询，引人不同的ElasticSearch查询来使结果更好，接着我们会使用过滤器，并降低垃圾文档的得分，然后引入切面计算，用来提供下拉菜单让用户缩小查询结果范围。最后，我们还将探讨如何查看这些查询的改变以及衡量用户搜索体验的变化。

1. 标准查询  
ElasticSearch默认把文档内容存人all字段中。既然这样，我们为什么还 要为如何同时查询多个字段伤脑筋呢，仅仅使用一个all字段就可以了，对吗? 按照这个构建查询如下：  

curl  -XGET 'localhost:9200/wikipedia/_search?fields=title&pretty' -d '{ 
    "query":{
        "match:":{
            "_all":{
                "query":"australian system",
                "operator":"OR"
            }
        }
    }
}'

结果  

{
  //...
  "hits" : {
    "total" : 562264,
    "max_score" : 3.3271418,
    "hits" : [ {
      "_index" : "wikipedia",
      "_type" : "page",
      "_id" : "3706849",
      "_score" : 3.3271418,
      "fields" : {
        "title" : "List of Australian Awards"
      }
    }, {
      "_index" : "wikipedia",
      "_type" : "page",
      "_id" : "26663528",
      "_score" : 2.9786692,
      "fields" : {
        "title" : "Australian rating system"
      }
    }, {
      "_index" : "wikipedia",
      "_type" : "page",
      "_id" : "7239275",
      "_score" : 2.9361649,
      "fields" : {
        "title" : "AANBUS"
      }
    }
//...
 ]
  }
}

第二个文档比第一个文档更相关，不是吗?让我们来尝试改善一下。

2. 多匹配查询  
在这里title字段比存储页面内容的text字段更重要。

curl  -XGET 'localhost:9200/wikipedia/_search?fields=title&pretty' -d '{ 
    "query":{
        "multi_match":{
            "query":"australian system",
            "fields":[ "title^100", "text^10", "_all" ] 
        }
    }
}'

结果

//...
  "hits" : {
    "total" : 562264,
    "max_score" : 5.3996744,
    "hits" : [ {
      "_index" : "wikipedia",
      "_type" : "page",
      "_id" : "7239222",
      "_score" : 5.3996744,
      "fields" : {
        "title" : "Australian Antarctic Building System"
      }
    }, {
      "_index" : "wikipedia",
      "_type" : "page",
      "_id" : "26663528",
      "_score" : 5.3996744,
      "fields" : {
        "title" : "Australian rating system"
      }
    }, {
      "_index" : "wikipedia",
      "_type" : "page",
      "_id" : "21697612",
      "_score" : 5.3968987,
      "fields" : {
        "title" : "Australian Integrated Forecast System"
      }
    }
//...
]
}

第一个文档的标题是“Australian Antarctic Building System "，而 第二个文档的标题是“Australian rating system "。我希望第二个文档的得分更高。  

3. 引入短语查询  
接下来我们应该想到的是引入短语查询。短语查询可以解决刚才描述的问题。不过， 我们还是希望能在与短语匹配的查询结果后面列出那些没有包含完整短语的结果。

curl  -XGET 'localhost:9200/wikipedia/_search?fields=title&pretty' -d '{ 
    "query":{
        "bool":{
            "must":[
                {
                    "multi_match":{
                        "query":"australian system",
                        "fields":[
                            "title^100",
                            "text^10",
                            "_all"
                        ]
                    }
                }
            ],
            "should":[
                {
                    "match_phrase":{
                        "title":"australian system"
                    }
                },
                {
                    "match_phrase":{
                        "text":"australian system"
                    }
                }
            ]
        }
    }
}'  

结果  

{
 //...
  "hits" : {
    "total" : 562264,
    "max_score" : 3.5905828,
    "hits" : [ {
      "_index" : "wikipedia",
      "_type" : "page",
      "_id" : "363039",
      "_score" : 3.5905828,
      "fields" : {
        "title" : "Australian honours system"
      }
    }, {
      "_index" : "wikipedia",
      "_type" : "page",
      "_id" : "7239222",
      "_score" : 1.7996382,
      "fields" : {
        "title" : "Australian Antarctic Building System"
      }
    }, {
      "_index" : "wikipedia",
      "_type" : "page",
      "_id" : "26663528",
      "_score" : 1.7996382,
      "fields" : {
        "title" : "Australian rating system"
      }
    }
//...
] 
  }
}  

4. 引入短语查询 - 加入slop   

尽管结果有所改善，但和我们的期望还是有点距离，因为没有找到与短语完全匹配的 记录。我们可以考虑引人slop参数。slop参数用来设置单词间的最大间隔，而在最大间隔之内的单词可被认为是和查询中的短语相匹配的。例如，这里使用的“australian system"短语查询，如果设置slop为大于等于1的数，那么就可以和标题为“australian education system”的文档匹配上。   
为了加入slop参数，将title、text提取出来作为key，内置query和slop对象：   

curl  -XGET 'localhost:9200/wikipedia/_search?fields=title&pretty' -d '{  
    "query":{
        "bool":{
            "must":[
                {
                    "multi_match":{
                        "query":"australian system",
                        "fields":[
                            "title^100",
                            "text^10",
                            "_all"
                        ]
                    }
                }
            ],
            "should":[
                {
                    "match_phrase":{
                        "title":{
                            "query":"australian system",
                            "slop":1
                        }
                    }
                },
                {
                    "match_phrase":{
                        "text":{
                            "query":"australian system",
                            "slop":1
                        }
                    }
                }
            ]
        }
    }
}'  

结果  

{
 //...
  "hits" : {
    "total" : 562264,
    "max_score" : 5.4896,
    "hits" : [ {
      "_index" : "wikipedia",
      "_type" : "page",
      "_id" : "7853879",
      "_score" : 5.4896,
      "fields" : {
        "title" : "Australian Honours System"
      }
    }, {
      "_index" : "wikipedia",
      "_type" : "page",
      "_id" : "363039",
      "_score" : 5.4625454,
      "fields" : {
        "title" : "Australian honours system"
      }
    }, {
      "_index" : "wikipedia",
      "_type" : "page",
      "_id" : "11268858",
      "_score" : 4.7900333,
      "fields" : {
        "title" : "Wikipedia:Articles for deletion/Australian university system"
      }
    } 
//...
] 
  }
}

5. 扔掉垃圾信息  
现在可以设法移除查询结果中的垃圾信息。因而需要移除重定向页面和特殊文档(如那 些被标记为已删除的文档)。我们在这里引入一个过滤器。过滤器的引入不会干扰其他文档 的排序(因为过滤器不具备评分功能)

curl  -XGET 'localhost:9200/wikipedia/_search?fields=title&pretty' -d '{ 
    "query":{
        "bool":{
            "must":[
                {
                    "multi_match":{
                        "query":"australian system",
                        "fields":[
                            "title^100",
                            "text^10",
                            "_all"
                        ]
                    }
                }
            ],
            "should":[
                {
                    "match_phrase":{
                        "title":{
                            "query":"australian system",
                            "slop":1
                        }
                    }
                },
                {
                    "match_phrase":{
                        "text":{
                            "query":"australian system",
                            "slop":1
                        }
                    }
                }
            ]
        }
    },
    "filter":{
        "bool":{
            "must_not":[
                {
                    "term":{
                        "redirect":"true"
                    }
                },
                {
                    "term":{
                        "special":"true"
                    }
                }
            ]
        }
    }
}'



6. 创建一个可纠正拼写错误的查询系统   
回顾一下索引的映射配置，你会发现有一个title字段被定义为multi_field类型，而 其中的一个字段需要经过ngram分析器的分词处理。默认情况下，ngram分析器会产生bigram，例如，对于单词system，将生成sy ys st teem等几个bigram。想象一下，我们可 以在查询时忽略其中的一些分词结果，从而使查询系统具备自动纠正拼写错误的能力。接下来用一个简单的查询来演示具体操作:  

curl  -XGET 'localhost:9200/wikipedia/_search?fields=title&pretty' -d '{
    "query":{
        "query_string":{
            "query":"austrelia",
            "default_field":"title",
            "minimum_should_match":"100%"
        }
    }
}'    
注意，上述查询故意写错austrelia,且title字段为multi_field类型：

其中，keyword_ngram分词定义入下：


在这里我们针对title字段发送了一个带有错误拼写的查询命令。由于索引中没有与拼 错词项完全匹配的文档，因而什么也没得到。然后我们转而使用title.ngram字段，并忽略一些bigram，这样ElasticSearch就可以找到一些匹配的文档了。修改后的查询命令如下:

curl  -XGET 'localhost:9200/wikipedia/_search?fields=title&pretty' -d '{
    "query":{
        "query_string":{
            "query":"austrelia",
            "default_field":"title.ngram",
            "minimum_should_match":"85%"
        }
    }
}'

引入了minimum_should_match，把它设置为85%，从而让 ElasticSearch知道，我们并不想要所有的bigram分词结果都匹配上，而是匹配其中一部分， 且具体哪些词项会匹配上我们也不关心。

结果  

{
  //...
  "hits" : {
    "total" : 67633,
    "max_score" : 1.9720218,
    "hits" : [ {
      "_index" : "wikipedia",
      "_type" : "page",
      "_id" : "11831449",
      "_score" : 1.9720218,
      "fields" : {
        "title" : "Aurelia (Australia)"
      }
    }, {
      "_index" : "wikipedia",
      "_type" : "page",
      "_id" : "2568010",
      "_score" : 1.8479118,
      "fields" : {
        "title" : "Australian Kestrel"
      }
    }, {
      "_index" : "wikipedia",
      "_type" : "page",
      "_id" : "8952722",
      "_score" : 1.778588,
      "fields" : {
        "title" : "Austrlia"
      }
    }
//...
 ] 
  }
}

7. 继续探讨切面计算   
以下查询中，字段category.untouched是不分词的，定义如下：


curl  -XGET 'localhost:9200/wikipedia/_search?fields=title&pretty' -d
'{
    "query":{
        "match_all":{
        }
    },
    "filter":{
        "term":{
            "category.untouched":"English-language films"
        }
    },
    "facets":{
        "category_facet":{
            "terms":{
                "field":"category.untouched",
                "size":10
            },
            "facet_filter":{
                "term":{
                    "category.untouched":"English-language films"
                }
            }
        }
    }
}'   

注意一件事:除了标准过滤器，我们还为category_facet引入了一个facet_filter片段，用来缩小切面计算的范围。这样做是正确的。否则，标准过滤器仅仅缩小了搜索结果范围，却不会缩小切面计算的范围，而我们希望能够展示给用户下一级嵌套信息。

# 8、ElasticSearch Java API #

## 连接到集群 ##
第一种连接到ElasticSearch节点的方式是把应用程序当成ElasticSearch集群中的一个节点。

    Node node=nodeBuilder().clusterName("escluster2").client (true).node();
    Client client=node.client();

第二种使用传输机连接方式

    Settings settings = ImmutableSettings.settingsBuilder().put("cluster.name","escluster2").build();
    TransportClient client=new TransportClient(settings);
    client.addTransportAddress(new InetSocketTransportAddress("127.0.0.1",9300))

注意这里的端口号不是9200，而是9300。 9200端口用来让HTTP REST API访问ElasticSearch，而9300端口是传输层监听的默认端口。
我们再回过头来看TransportClient类的可用配置:

- client.transport.sniff(默认值:false):如果设为true , ElasticSearch会读取集群中的 节点信息。因此你不需要在建立TransportClient对象时提供所有节点的网络地址。 ElasticSearch很智能，它会自动检测到活跃节点并把它们加人到列表中。
- client.transport.ignore_cluster_name(默认值:false ):如果设为true，则ElasticSearch会忽视配置中的集群名称并尝试连接到某个可连接集群上，而不管集群名称是否匹配。这一点很危险，你可能会首先连接到你不想要的集群上。
- client.transport.ping_timeout(默认值:5s):该参数指定了ping命令响应的超时时间。 如果客户端和集群间的网络延迟较大或连接不稳定，可能需要调大这个取值。
- client.transport.nodes_sampler_interval默认值:5s):该参数指定了检查节点可用 性的时间间隔。与前一个参数类似，如果网络延迟较大或者网络不稳定，可能需要调大这个值。

第一种方式让 启动顺序复杂化了，即客户端节点必须加人集群并建立与其他节点的连接，而这需要时间和资源。然而，操作却可以更快地执行，因为所有关于集群、索引、分片的信息都对客户端节点可见。

另一方面，使用TransportClient对象启动速度更快，需要的资源更少，如更少的socket连接。然而，发送查询和数据却需要消耗更多的资源，TransportClient对象对 集群和索引拓扑结构的信息一无所知，所以它无法把数据直接发送到正确的节点，而是先把数据发给某个初始传输节点，然后再由ElasticSearch来完成剩下的转发工作。此外，TransportClient对象还需要你提供一份待连接节点的网络地址列表。

在选择合适的连接方式时请记住，第一种方式并不总是可行的。例如，当要连接的ElasticSearch集群处于另一个局域网中，唯一可选的方式就是使用TransportClient对象。

## API部析 ##
获取特定文档记录

    GetResponse response=client
    .prepareGet(”library","book",”1")
    .setFields("title"，”_source")
    .execute().actionGet();

在完成准备工作之后，我们用构造器对象创建一个请求 (利用request()方法)以备随后使用，或者直接通过execute()调用送出查询请求，这里我们选择后者。

ELasticSearch的API是天生异步的。这意味着execute()调用 不会等待ElasticSearch的响应结果，而是直接把控制权交回给调用它的代码段，而查询请求在后台执行。本例中我们使用actionGet()方法，这个方法会等待查询执行完毕并返回数据。这样做比较简单，但在更复杂的系统中这显然不够。接下来是一个使用异步API的例子。首先引人相关声明:

    ListenableActionFuture<GetResponse> future=client
    .prepareGet("library","book","1")
    .setFields("title",”_source")
    .execute();
    future.addListener( new ActionListener<GetResponse>(){
      @Override
      public void onResponse(GetResponse response){
    System.out.println("Document:"+response.getIndex()
       +”/“
       +response .getType()
       +”/"
       +response.getId());
    }
      @Override
      public void onFailure(Throwable e){
      throw new RuntimeException(e);
      }
    });

## CRUD操作 ##
**读取文档**

    GetResponse response=client
    .prepareGet("library","book","1")
    .setFields("title","_source")
    .execute().actionGet();

GetRequestBuilder的方法

    setFields(String):
    setIndex(String) , setType(String) , setId(String):
    setRouting(String):
    setParent(String):
    setPreference(String): 【_local, _primary】
    setRefresh(Boolean): 【false】
    setRealtime(Boolean):【true】

响应GetResponse的方法

    isExists():
    getindex():
    getType():
    getId():
    getVersion():
    isSourceEmpty():
    getSourceXXX():【getSourceAsString()，getSourceAsMap()，getSourceAsBytes()】
    getField(String):

**索引文档**

    IndexResponae response=client.prepareIndex("library","book","2")
      .setSource("{\"title\":\"Mastering ElasticSearch\"}")
      .execute().actionGet();

构造器对象提供了如下方法:

    setSource():
    setlndex(String), setType(String), setId(String):
    setRouting(String) , setParent(String):
    setOpType():【index，create】
    setRefresh(Boolean):【false】
    setReplicationType():【sync，async，default】
    setConsistencyLevel():【DEFAULT，ONE，QUORUM,ALL】多少副本活跃才进行该操作
    setVersion(long):多亏了这个方法。程序可以确保在读取和更新文档期间没有别人更改这个文档。
    setVersionType(VersionType):
    setPercolate(String)：
    setTimestamp(String)：
    setTTL(long):超过这个时长后文档会被自动删除
    getlndex():返回请求的索引名称
    getType():返回文档类型名称
    getId():返回被索引文档的ID
    getVersion():返回被索引文档的版本号
    getMatches():返回匹配被索引文档的过滤器查询列表，如果没有匹配的查询，则返回null

**更新文档**

    Map<String, Object>params=Maps.newHashMap();
    params.put("ntitle","ElasticSearch Server Book");
    UpdateResponse response = client.prepareUpdate("library","book","2")
      .setScript("ctx._source.title=ntitle")
      .setScriptParams(params)
      .execute().actionGet();

UpdateRequestBuilder 方法

setIndex(String) , setType(String) , setId(String):  
setRouting(String), setParent(String):  
setScript(String):  
setScriptLang(String):  
setScriptParams(Map<String, Object>):  
addScriptParam(String, Object):  
setFields(String...):  

setRetryOnConflict(int):默认值为0。在ElasticSearch中，更新一个文档意味着先检索出文档的旧版本，修改它的结构，从索引中删除旧版本然后重新索引新版本。这意味着，在检索出旧版本和写人新版本之间，目标文档可能被其他程序修改。ElasticSearch通过比较文档版本号来检测修改，如果发现修改则返回错误。除了直接返回错误，还可以选择重试本操作。而这个重试次数就可以由本方法指定。  
setRefresh(Boolean):【false】  
setRepliactionType():【sync, async, default】。本方法用于控制在更新过程中的复制类型。默认情况下，只有所有副本执行更新后才认为更新操作是成功的，对应这里的取值为sync或等价枚举值。另一种选择是不等待副本上的操作完成就直接返回，对应取值为async或等价枚举值。还有一种选择是让ElasticSearch根据节点配置来决定如何操作，对应取值为default或等价枚举值。  
setConsistencyLevel():【DEFAULT，ONE，QUORUM，ALL】本方法设定有多少个活跃副本时才能够执行更新操作。  
setPercolate(String):这个方法将导致索引文档要经过percolator的检查，其参数是 一个用来限制percolator查询的查询串。取值为`*`表示所有查询都要检查。  
setDoc():这类方法用来设置文档片段，而这些文档片段将合并到索引中相同ID的 文档上。如果通过setScript()方法设置了script，则文档将被忽略。ElasticSearch提供了本方法的多种版本，分别需要输入字符串、字节数组、XContentBuilder, Map 等多种类型表示的文档。  
setSource():  
setDocsAsUpsert(Boolean):默认值为false。设置为true后，如果指定文档在索引中不存在，则setDoc()方法使用的文档将作为新文档加人到索引中。    
更新请求返回的响应为一个UpdateResponse对象方法：   
getIndex():   
getType():  
getld():  
getVersion():  
getMatches():  
getGetResult():  

**删除文档**
DeleteResponse response=client.prepareDelete("library","book","2")  
    .execute().actionGet();  

DeleteRequestBuilder 方法   
setIndex(String), setType(String), setId(String):  
setRouting(String), setParent(String):  
setRefresh(Boolean):  
setVersion(long):本方法指定索引时被删除文档的版本号。如果指定ID的文档不 存在，或者版本号不匹配，则删除操作会失败。这个方法确保了程序中没有别人更改这个文档。   
setVersionType(VersionType):本方法告知ElasticSearch使用哪个版本类型。  
setRepliactionType():【sync, async和default】  
setConsistencyLevel():本方法设定有多少个活跃副本时才能够执行更新操作。【DEFAULT，ONE，QUORUM，ALL】  
删除操作的响应DeleteResponse类提供了如下方法:   
getIndex():返回请求的索引名称。  
getType():返回文档类别名称。  
getId():返回被索引文档的ID。  
getVersion():返回被索引文档的版本号。  
isNotFound():如果请求未找到待删除文档，则返回true  

## ElasticSearch查询 ##
**准备查询请求**

    SearchResponse response=client.prepareSearch("library"
    .addFields('title"，"_source")
    .execute().actionGet();
    for(SearchHit hit:response.getHits().getHits()){
      System.out.println(hit.getId());
      if (hit.getFields().containsKey("title"}){
      System.out.println("field.title: "
    +hit.getFields().get("title").getValue())
    }
      System.out.println("source.title: "
     +hit.getSource().get("title"));
     }
   
**构造查询**

    QueryBuilder queryBuilder=QueryBuilders
    .disMaxQuery()
      .add(QueryBuilders.termQuery("title","Elastic"))
      .add(QueryBuilders.prefixQuery("title","el"));
    
    
    System.out.println(queryBuilder.toString());
    SearchResponse response=client.prepareSearch("library")
      .setQuery(queryBuilder)
      .execute().actionGet();

匹配查询

    queryBuilder=QueryBuilders
    .matchQuery("message"，"a quick brown fox")
    .operator(Operator.AND)
    .zeroTermsQuery(ZeroTermsQuery.ALL);

使用地理位置查询

    queryBuilder = QueryBuilders.geoShapeQuery("location",
      ShapeBuilder.newRectangle()
      .topLeft(13, 53)
      .bottomRight(14, 52)
    .build()); 

**分页**

    SearchResponse response=client.prepareSearch("library")
      .setQuery(QueryBuilders.matchAllQuery())
    .setFrom(10)
    .setSize(20)
    .execute().actionGet();

**排序**

    SearchResponse response=client.prepareSearch("library")
      .setQuery(QueryBuilders.matchAllQuery())
      .addSort(SortBuilders.fieldSort("title"))
      .addsort("_score"，SortOrder .DESC)
      .execute().actionGet();

除了上述排序方法外，ElasticSearcb还提供了一种基于脚本的排序方法:scriptSort(String, String)，以及一种基于空间距离排序的方法:geoDistanceSort(String)。

**过滤**

    FilterBuilder filterBuilder=FilterBuilders
      .andFilter(
     FilterBuilders .existsFilter("title").filterName("exist”),
     FilterBuilders.termFilter("title"，"elastic")
      );
    SearchResponse response=client.prepareSearch("library")
      .setFilter(filterBuilder)
      .execute().actionGet();

**切面计算**

    FacetSBuilder facetBuilder=FacetBuilders
      .filterFacet("test")
      .filter(FilterBuilders .termFilter("title","elastic"));
    SearchResponse response=client.prepareSearch("library")
      .addFacet(facetBuilder)
      .execute()actionGet();
    
**高亮**

    SearchResponse response=client.preparesearch("wikipedia")
      .addHighlightedField("title")
      .setQuery(QueryBuilders.termQuery("title"，"actress"))
      .setHighlighterPreTags("<1>", "<2>")
      .setHighlighterPostTags("</1>"," </2>")
    .execute().actionGet();

**高亮处理**

    for(SearchHit hit:response.getHits().getHits()){
      HighlightField hField = hit.getHighlightFields()get("title");
      for (Text t:hField.fragments()){
    System.out.println(t.string());
      }
    }

**查询建议**

    SearchResponse response=client.prepareSearch("wikipedia")
      .setQuery(QueryBuilders.matchAllQuery())
      .addSuggestion(new TermSuggestionBuilder("first_suggestion")
      .text("graphics designer")
      .field("_all")
      )
      .execute().actionGet()

结果处理

    for(Entry<? extends Option> entry: response.getSuggest()
      .getSuggestion("first_suggestion").getEntries()){
    System.out.println("Check for: "
      +entry.getText()
      +". Options:");
      for(Option option: entry.getOptions()){
    System.out.println("\t"+option.getText());
      }

**计数**

有时候，我们不在乎具体会返回哪些文档，而只想知道匹配的结果数量。在这种情况下，我们需要使用计数请求，因为它的性能更好:不需要做排序，也不需要从索引中取出文档。

    CountResponse response=client.prepareCount("library")
      .setQuery(QueryBuilders.termQuery("title","elastic"))
      .execute().actionGet();

**滚动**

要使用ElasticSearch Java API的滚动功能获取大量文档集合:

    SearchResponse responseSearch=client.prepareSearch("library" )
      .setScroll("1m")
      .setSearchType(SearchType.SCAN)
      .execute().actionGet();
    
    String scrolled=responseSearch.getScrollId();
    SearchResponse response = client.prepareSearchScroll(scrollld)
      .execute().actionGet()，

其中有两个请求:第一个请求指定了查询条件以及滚动属性，如滚动的有效时长(使用 setScroll()方法);第二个请求指定了查询类型，如这里我们切换到扫描模式(scan mode)。扫描模式下会忽略排序而仅执行取出文档操作。

**集群管理API**

    ClusterAdminClient cluster=client.admin().cluster();

**索引管理API**

    IndicesAdminClient cluster = client.admin().indices();