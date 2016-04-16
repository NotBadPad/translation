##hbase-indexer  
### 简介  
HBase Indexer为存储在hbase中的数据提供了索引(通过solr)。它使用scale提供了一种灵活和可扩展的方式来定义索引规则。
由于索引是异步建立，所以不会对hbase的写入产生影响。SolrCloud被用来存储实际索引，以用来增强hbase的索引能力。  
### 如何工作  
HBase Indexer会作为hbase的一个副本进行工作。当数据被写入hbase的region server时，副本会被异步的复制到HBase Indexer进程。
indexer会分析hbase的数据变更事件，适配并创建对应的solr documents，并发送到solrcloud。
solr中被索引的documents文件关联到hbase唯一的行id上(row-key),这样你就可以通过solr查询被存储在hbase中的数据了。
HBase的复制是基于读取hbase的log文件实现的，这些文件也是hbase实际存储的内容：这样就不会丢失或者有额外的事件。无论什么情况下，这些log文件都会包含需要索引的数据，所以不需要在habse上进行代价较大的随机读操作(read-row属性参见[Indexer Configuration](https://github.com/NGDATA/hbase-indexer/wiki/Indexer-configuration))