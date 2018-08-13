ealticsearch6.x版本

##基本概念：##

1. 准实时系统

elsticsearch是一个准实时的平台，从数据入库到可搜索几乎零时差（大约1s）

2. 集群

es集群由多个节点组成，可以容纳所有的数据，每个节点都可以入库和提供查询的能力，每个集群都有一个唯一的名字（默认为elasticsearch），这个节点名非常重要因为节点都是根据这个唯一的集群名字加入集群，确保不同环境的es的集群名字应该不同

3.节点

节点在启动加入集群的时候的时候默认都会有一个UUID，节点通过集群名加入es集群，集群上es节点原则上可以无限水平扩展

4.索引index

索引是文档的集合，一个索引（索引名必须为小写字母），提供对文档进行入库，查询，更新和删除的各个功能

5.Type
type用于对索引数据进行逻辑的切分，例如索引数据可以根据type分为用户信息，博客信息等

6.文档document
文档指的是可以入库的最小单位

7.分片和副本
一个索引可以存储的数据往往超过了当个节点上的磁盘存储空间，为了解决这个问题，es将索引数据拆分为多个分片存储在不同的节点上，每个节点是独立并且具备入库，查询，搜索，删除所有功能，创建索引的时候你可以随机定义你所需要的分片数，该分片数在索引创建之后不可被修改

当一个网络环境不稳定存在故障的环境下，配置分片副本可以防止某个节点由于某种原因下线。配置副本有两个比较重要的两个原因：
   
   * 配置副本可以防止节点故障，每个分片及其副本都不会分布在同一个节点上
   * 副本可以提高es的查询销量
   
   
es默认为5分片和单副本，就是说一个索引将存在10个分片

*每个es的分片本质上为lucense的index，lucense的index有一个文档最大数目存储的限制，例如LUCENE-5843，最大文档存储个数为2,147,483,519，大约为21亿条*

你可以查看/_cat/shards可以看到所有的shards的情况


## 安装 ##
es6.x需要使用到java8的版本，这里es强烈建议使用的是 Oracle JDK version 1.8.0_131，

具体的不同的os的安装教程可以查看[安装教程](https://www.elastic.co/guide/en/elasticsearch/reference/current/_installation.html)


## The REST API ##

1. 查看集群的健康状态

```
GET /_cat/health?v

epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1475247709 17:01:49  elasticsearch green           1         1      0   0    0    0        0             0                  -                100.0%

```

集群的状态有三种，green（全功能），yello（全功能，存在副本没有分配的情况），red（存在主片没有分配的情况）

*集群为red的时候，es集群无法提供入库的功能，只能提供可用的分片的查询功能*



2.查看节点的各个情况

```
GET /_cat/nodes?v
```


3. 查看集群所有的索引


```
GET /_cat/indices?v
```

4. 创建索引

```
PUT /customer?pretty
```

5.删除索引

```
DELETE /customer?pretty
```

6.删除特定的数据

```
POST twitter/_delete_by_query
{
  "query": { 
    "match": {
      "message": "some message"
    }
  }
}
```

7. bulk

```
POST /customer/_doc/_bulk?pretty
{"index":{"_id":"1"}}
{"name": "John Doe" }
{"index":{"_id":"2"}}
{"name": "Jane Doe" }
```

8.导入json数据

```
curl -H "Content-Type: application/json" -XPOST "localhost:9200/bank/_doc/_bulk?pretty&refresh" --data-binary "@accounts.json"
```

9.查询api分为两种，普通查询和dsl查询

```
普通查询
GET /bank/_search?q=*&sort=account_number:asc&pretty

DSL查询
GET /bank/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ]
}
```

## ES配置 ##

es默认的配置参数比较少，大部分的参数可以使用setting update的接口进行修改

```
GET /_cluster/settings 查看es的配置
```

更新setting可以是永远的，persistent的,集群在重启之后还可以继续生效的

```
PUT /_cluster/settings
{
    "persistent" : {
        "indices.recovery.max_bytes_per_sec" : "50mb"
    }
}
```

更新setting可以是临时的，transient的,集群在重启之后还可以继续生效的

```
PUT /_cluster/settings?flat_settings=true
{
    "transient" : {
        "indices.recovery.max_bytes_per_sec" : "20mb"
    }
}
```

es有三种配置文件
    
```
elasticsearch.yml   配置es的参数
log4j2.properties.  配置es的log文件
jvm.options         配置es的jvm参数
```


jvm的参数一般极少改动，需要改动的地方一般为heap size. 

es的重要的配置参数

```
目前的path.data支持多目录配置，本质上可以支持到单盘，无需做raid，存在数据丢失的问题
path.logs
path.data


cluster.name是各个节点加入集群的重要信息
cluster.name
node.name


network.host 默认es的绑定在lo口，为了与集群的其他节点相交互，我们需要配置network.host为本机的ip
network.host

自动发现配置，默认es会绑定在lo口并且扫描端口9300到9305去连接其他的节点
discovery.zen.ping.unicast.hosts:
   - 192.168.1.10:9300
   - 192.168.1.11 
   - seeds.mydomain.com

为了防止脑裂的参数，该配置的值为（master个数／2+1）
discovery.zen.minimum_master_nodes

默认es使用java heap至少为1G至多也为1G的配置，下面是一些heap配置的建议
1.配置minimum heap size (Xms) 和maximum heap size (Xmx)相同

2.maximum heap size (Xmx)不能大于物理内存的一半，确保有一半的内存分配给内核的cache中
-Xms2g 
-Xmx2g

系统上的配置配置
1.禁用swap 使用swapoff -a禁用掉swap内存，需要修改 /etc/fstab使得配置在主机重启的时候可以生效，修改内核参数vm.swappiness为1，最大限制先使用物理内存

bootstrap.memory_lock: true 不允许es使用到swap内存，可以通过GET _nodes?filter_path=**.mlockall查看具体的情况，这对于独占机器的情况是合适的，但对于共享的则不好，因为mlock后即使进程用不到这么多内存，别的进程也用不到

2.配置打开句柄最大叔为set nofile to 65536 in /etc/security/limits.conf，同样可以同GET _nodes/stats/process?filter_path=**.max_file_descriptors
查看打开文件句柄打开最大数

3.配置一个进程可以拥有的VMA，sysctl -w vm.max_map_count=262144，永久生效需要在/etc/sysctl.conf进行配置

4.配置系统线程数ulimit -u 4096 

```

##ES原理##

Elasticsearch的索引(index)是用于组织数据的逻辑命名空间（如数据库）。Elasticsearch的索引有一个或多个分片(shard)（默认为5）。分片是实际存储数据的Lucene索引，它本身就是一个搜索引擎。每个分片可以有零个或多个副本(replicas)（默认为1）。Elasticsearch索引还具有“类型”（如数据库中的表），允许您在索引中对数据进行逻辑分区。Elasticsearch索引中给定“类型”中的所有文档(documents)具有相同的属性（如表的模式）

elasticsearch节点

* master  master节点主要负责控制es集群，负责在集群范围内删除，新增索引，跟踪集群的信息，将分片分配给相对应的节点，master处理集群状态，并且将集群状态广播到其他节点，其他节点需要响应并且确认信息，在elasticsearch.yml中，将nodes.master属性设置为true（默认），可以将节点配置为有资格成为主节点的节点。

* data  数据节点用来保存数据和倒排索引。默认情况下，每个节点都配置为一个data节点，并且在elasticsearch.yml中将属性node.data设置为true。如果您想要一个专用的master节点，那么将node.data属性更改为false
* client     如果将node.master和node.data设置为false，则将节点配置为客户端节点，并充当负载平衡器，将传入的请求路由到集群中的不同节点


###es写入原理###
当数据往es节点写入数据的时候，Elasticsearch将文档ID以murmur3作为散列函数进行散列，并通过索引中的主分片数量进行取模运算，以确定文档应被索引到哪个分片。 

```
shard = hash(document_id) % (num_of_primary_shards)
```

当分片节点上的收到请求节点上的请求的时候，请求被写入到translog，并将该文档添加到内存缓冲区，如果请求在主分片上成功，则请求将并行发送到副本分片。只有在所有主分片和副本分片上的translog被fsync’ed后，客户端才会收到该请求成功的确认。

内存缓冲区以固定的间隔刷新（默认为1秒），并将内容写入文件系统缓存中的新段，此分段的内容更尚未被fsync’ed(未被写入文件系统)，分段是打开的，内容可用于搜索，此时的数据还未被持久化

translog被清空，并且文件系统缓存每隔30分钟进行一次fsync，或者当translog变得太大时进行一次fsync。这个过程在Elasticsearch中称为flush，在刷新过程中，内存缓冲区被清除，内容被写入新的文件分段(segment)。当文件分段被fsync’ed并刷新到磁盘，会创建一个新的提交点（其实就是会更新文件偏移量，文件系统会自动做这个操作）。旧的translog被删除，一个新的开始

![das](https://img-blog.csdn.net/20170814225416168?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvemdfaG92ZXI=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

###es查询原理###

在此阶段，协调节点将搜索请求路由到索引(index)中的所有分片(shards)（包括：主要或副本）。分片独立执行搜索，并根据相关性分数创建一个优先级排序结果（稍后我们将介绍相关性分数）。所有分片将匹配的文档和相关分数的文档ID返回给协调节点。协调节点创建一个新的优先级队列，并对全局结果进行排序。可以有很多文档匹配结果，但默认情况下，每个分片将前10个结果发送到协调节点，协调创建优先级队列，从所有分片中分选结果并返回前10个匹配


###translog###
Translog在发生故障的情况下确保数据的完整性，其基本原则是，在数据的实际更改提交到磁盘之前必须先记录并提交预期的更改

当新文档被索引或旧文档被更新时，Lucene索引会更改，这些更改将被提交到磁盘以进行持久化。每次写请求之后进行持久化操作是一个非常消耗性能的操作，它通过一次持久化多个修改到磁盘的方式运行（译注：也就是批量写入）。正如我们在之前的一篇博客中描述的那样，默认情况下每30分钟执行一次flush操作（Lucene commit），或者当translog变得太大（默认为512MB）时）。在这种情况下，有可能失去两次Lucene提交之间的所有变化。为了避免这个问题，Elasticsearch使用translog。所有的索引/删除/更新操作都先被写入translog，并且在每个索引/删除/更新操作之后（或者每默认默认为5秒），translog都被fsync’s，以确保更改被持久化。在主文件和副本分片上的translog被fsync’ed后，客户端都会收到写入确认。

建议在重新启动Elasticsearch实例之前显式刷新translog，因为启动将更快，因为要重播的translog将为空。POST / _all / _flush命令可用于刷新集群中的所有索引。 

###Lucene Segments###
Lucene索引由多个片段(segment)组成，片段本身是完全功能的倒排索引。片段是不可变的，这允许Lucene增量地向索引添加新文档，而无需从头开始重建索引。对于每个搜索请求，搜所有段都会被搜素，每个段消耗CPU周期，文件句柄和内存。这意味着分段数越多，搜索性能就越低。 
    为了解决这个问题，Elasticsearch将小段合并成一个更大的段（如下图所示），将新的合并段提交到磁盘，并删除旧的较小的段，合并操作将会自动发生在后台，而不会中断索引或搜索。由于分段合并可能会浪费资源并影响搜索性能，因此Elasticsearch会限制合并过程的资源使用，以获得足够的资源进行搜索。
    
   es每个间隔（index.refresh_interval）都会refresh一次，每次refresh都会生成一个新的segment，按照这个速度过不了多久segment的数量就会爆炸，所以存在太多的segment是一个大问题，因为每一个segment都会占用文件句柄，内存资源，cpu资源，更加重要的是每一个搜索请求都必须访问每一个segment，这就意味着存在的segment越多，搜索请求就会变的更慢。更加重要的是每一个搜索请求都必须访问每一个segment，这就意味着存在的segment越多，搜索请求就会变的更慢
    实际上elasticsearch有一个后台进程专门负责segment的合并，它会把小segments合并成更大的segments，然后反复这样。在合并segments的时候标记删除的document不会被合并到新的更大的segment里面，所有的过程都不需要我们干涉，es会自动在索引和搜索的过程中完成，合并的segment可以是磁盘上已经commit过的索引，也可以在内存中还未commit的segment：
    如果不加控制，合并一个大的segment会消耗比较多的io和cpu资源，同时也会搜索性能造成影响，所以默认情况下es已经对合并线程做了资源限额以便于它不会搜索性能造成太大影响。
    
```
PUT /_cluster/settings
{
    "persistent" : {
        "indices.store.throttle.max_bytes_per_sec" : "100mb"
    }
}
```
	
es的api也提供了我们外部发送命令来强制合并segment，这个命令就是optimize，它可以强制一个shard合并成指定数量的segment，这个参数是：max_num_segments ，一个索引它的segment数量越少，它的搜索性能就越高，通常会optimize成一个segment。需要注意的是optimize命令不要用在一个频繁更新的索引上面，针对频繁更新的索引es默认的合并进程就是最优的策略，optimize命令通常用在一个静态索引上,例如每晚对前一天的索引进行合并segment，这样搜索速度便会大大提升
	
```
	POST /logstash-2014-10/_optimize?max_num_segments=1
```
由外部发送的optimize命令是没有限制资源的，也就是你系统有多少IO资源就会使用多少IO资源，即lucene3.5中使用writer.forceMerge(1)来替换optimize()方法.
    
##es常见排障##   

###如何优化io过高的问题（IOwait过高的问题）###
linux操作系统默认情况下写都是有写缓存的，可以使用direct IO方式绕过操作系统的写缓存。当你写一串数据时，系统会开辟一块内存区域缓存这些数据，这块区域就是我们常说的page cache（操作系统的页缓存）,cat /proc/meminfo 查看详细的内存,例如cached，dirty，dirty表示有多少脏页存储在page cache（默认为140M），等待后台刷入磁盘

一个写IO会先写入page cache，然后等待后台pdflush把page cache中脏数据刷入磁盘，pdflush是linux系统后台运行的一个线程，这个进程负责把page cahce中的dirty状态的数据定期的输入磁盘

cat /proc/sys/vm/nr_pdflush_threads查看当前系统运行pdflush数量

重要的IO写相关参数

```
dirty_writeback_centisecs. cat /proc/sys/vm/dirty_writeback_centisecs查看这个值，默认一般是500（单位是1/100秒）。这个参数表示5s的时间pdflush就会被唤起去刷新脏数据没有官方文档说明减少这个值就会有更多的pdflush参与刷数据。比如2.6或者更早的内核，linux中mm/page-writeback.c的源码中有这样一段描述“如果pdflush刷新脏数据的时间超过了这个配置时间，则完成刷新后pdflush会sleep 1s“。这种拥塞的保护机制描述只是写在源码里，并没有写入官方文档或者形成规范，所以也就意味着这种机制在不同的版本可能有不同的表现。
所以修改dirty_writeback_centisecs并不一定能给你带来多少性能的提升，相反有可能出现你意想不到的问题。一般建议用户使用默认值。

dirty_expire_centisecscat /proc/sys/vm/dirty_expire_centicecs查看这个值，默认是3000（单位是1/100秒）。这个值表示page cache中的数据多久之后被标记为脏数据。只有标记为脏的数据在下一个周期到来时pdflush才会刷入到磁盘，这样就意味着用户写的数据在30秒之后才有可能被刷入磁盘，在这期间断电都是会丢数据的。如果想pdfflush刷新频率大写可以减小这个值，比如：echo 1000 >> /proc/sys/vm/dirty_expire_centicecs 设置为10s一个刷新周期

dirty_ratio  cat /proc/sys/vm/dirty_ratio查看这个值，默认是20（单位是百分比，不同的内核版本可能有不同的默认值）。表示当脏数据占用总内存的百分比超过20%的时候，内核会把所有的写操作阻塞掉，等待pdflush把这些脏数据刷入到磁盘后才能恢复正常的IO写。要注意的是当这个事件发生时，会阻塞掉所有写操作。这样会产生一个很大的问题，一个长时间大IO会抢占更多的IO写资源，可能把其它的小IO饿死。因为大IO产生的脏数据较多，很快达到这个阀值，此时就会系统会阻塞掉所有的写IO，从而小写IO无法进行写操作。

dirty_backgroud_ratio. cat /proc/sys/vm/dirty_backgroud_ratio查看这个值，默认是10（单位是百分比，不同的内核版本可能有不同的默认值）。很多的描述文档中描述这个值表示最多缓存脏数据的空间占总内存的百分比。其实不然，查看源码的描述，它的真实意义是占（MemFree + Cached - Mapped）的百分比。达到这个上限后会唤醒pdflush把这些脏数据刷新到磁盘，在把脏数据输入磁盘之前所有写IO会被阻塞。所以如果这个值设的过大，则会周期的出现一个写IO峰值，而且这个峰值持续比较长时间，在这段时间内用户的写IO会被阻塞。对于一些业务场景需要把这个值设置的小写，把峰值写IO平分为多次小的写IO。

```

pdflush写入硬盘看两个参数：

1 数据在页缓存中是否超出30秒，如果是，标记为脏页缓存，写入磁盘;

2 脏页缓存是否达到工作内存的10%;

可以根据具体的场景调整以上的参数


###es常用命令###
1.按查询进行删除
```
curl -XPOST "yottasearch10:9200/yotta*/_delete_by_query?q=appname:test"
``` 
    
2.查看线程池的状态，当出现了reject的情况时，一般为线程数太低或者集群已经超载，需要调整配置
```
curl -XGET "http://yottasearch10:9200/_cat/thread_pool?v"
```

3. 查看segment的占用内存和segment的情况
```
curl yottasearch10:9200/_cat/thread_pool?v
```

4.设置一个shard的段segment最大数
```
curl -XPUT 'http://xxxxx:9400/index_rfm_test/_optimize?max_num_segments=5' 
```


###后台什么在运行导致CPU飙升？如何排查###
```
GET /nodes/?pretty
```


##高负载场景Elasticsearch优化##

1.硬件原因，长时间的gc，内存，io成为瓶颈，没有好的办法，只能进行扩容

2.按需设定刷新频率index.refresh_interval（默认为1s），就是配置内存刷新到文件缓存的配置（未写入磁盘持久化，但是可以被搜索到），因为刷新是非常昂贵的操作，提升索引吞吐量的方式之一就是增大refresh_interval；更少的刷新意味着更低的负载

3.配置分片均衡，入库速度稳定

4.合理配置 indices.memory.index_buffer_size，一般为配置的jvm的10%

5.适当增大节点threadpool参数,参数可以根据自身的需求进行配置，前提先查看是否出现了reject的现象
```
curl -XPUT 'http://xxxxx:9200/_cluster/settings' -d '{"transient":{"threadpool.index.queue_size":5000,"threadpool.bulk.queue_size": 5000}}'
```

6.空闲时分merger那些几乎没有入库数量小的index
```
curl -XPOST "http://localhost:9200/indexname/_forcemerge?max_num_segments=1
```


