---
title: Elasticsearch基础
date: 2023-01-20 09:14:15 +0800 
categories: [storage, Elasticsearch]
tags: [history, Elasticsearch, middleware, storage] 
---

### Elasticsearch
官方文档：[https://www.elastic.co/guide/index.html](https://www.elastic.co/guide/index.html)
中文版（内容可能过时）：[https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)
#### 1.场景和概念
##### 1.1 适用场景
- 分布式的搜索引擎和数据分析引擎
- 全文检索，结构化检索，数据分析
- 对海量数据进行近实时的处理
##### 1.2 不适用的场景
- 一致性要求高的数据，不适用多个数据的复杂事物性、回滚要求严格的场景
> 实时支付，实时订单的生命周期等，ES没有事务，不无法满足多个Update操作的过程中事务隔离要求。

- 不适用数据写入之后立即查询的场景
> 用户信息保存，登录，验证码等，ES为提升性能，内部会将写入与查询完全分离，且所有持久化操作异步，这意味着一个写请求完成后ES可能未实际写入数据，读请求也不可能读到，此类问题会造成用户写入数据后短期查询不到。

- 不适合大量的文本读取，IO特别高场景
> 获取用户1年以内的订单列表，1年内的银行转帐流水等，因ES的数据分布在多个实例中，过大的IO会被放大节点数量倍，并最终在一个节点的内存中聚合，整个集群开销非常大，GC负担大，性能低下。
 
- 不适用存储低附加值的数据
> 历史归档数据存储等，ES的存储数据前要进行一系列非常复杂繁重的操作建立倒排索引，存储成本远大于任何一个数据持久化中间件，不值。
 
- 不适用于简单条件查询
  + **Elasticsearch：**选定索引->路由分片节点->分词->TF-IDF计算->评分排序->文档读取→解压缩→返回。
  + **MySQL：**选定索引->遍历索引树->分区磁盘读取→返回。

##### 1.3 与MySQL、Nosql（redis）区别
**索引数据结构**
>**Elasticsearch：**基于lucene的倒排索引，

>**Mysql：**基于MyISAM和InnoDB两个存储引擎的B+Tree作为索引结构

>**Nosql：**内存Key-value结构

**数据存储结构**
> **Elasticsearch：**Elasticsearch --> Indices --> Types --> Documents --> Fields；非结构化文档数据

>**Mysql：**Relational DB --> Databases --> Tables --> Rows --> Columns；结构化数据，关系型数据

>**Nosql：**key–通用value结构+key的注册表如时间，命中次数等；非结构化内存数据

**数据单元**
>**Elasticsearch：**Index>type(<6.0)>document>field>mapping>everything isIndexed>DSL

>**Mysql：**database>table>Row>column>Schema>index>SQL

>**Nosql：**Key-value

**机器成本**
> **Elasticsearch：**磁盘性能要求高，机器配置高，官方建议16c 64G，成本高

>**Mysql：**机器配置适中，成本低

>**Nosql：**比mysql略高，主要限制在内存容量，网络带宽，存储数据量较小，存储能力强依赖内存大小

**分布式扩展**
>**Elasticsearch：**原生支持分片，快速水平拓展能力强，迁移成本较低

>**Mysql：**基于中间件和业务路由策略，水平拓展迁移成本较高

>**Nosql：**水平拓展能力较弱，迁移成本高

**数据字段拓展性**
>**Elasticsearch：**1000字段以下，拓展能力强，1000字段以上需要拆分索引；字段创建后，历史数据不可修改  

>**Mysql：**依据单表数据量，1000w数据量以下，拓展能力适中，1000w数据量以上，表变更性能损耗较大；能修改_字段类型_、类型长度、默认值、注释  

>**Nosql：**随时变更结构，查询存储都是key的纬度，但需要考虑数据的淘汰策略

**一致性，事务能力**
>**Elasticsearch：**无法保证，需要业务支持

>**Mysql：**具备多个事务隔离级别，一致性好；支持：复杂的业务逻辑控制数据，频繁更改数据；保证原子性的数据,多个同时成功存储的数据；可以应用于：业务逻辑严谨，权限，角色，菜单等场景；

>**Nosql：**无法保证，需要业务支持

**查询性能**
>**Elasticsearch：**复杂查询性能高：可以执行 跨索引，跨表关系性查询少的数据，全文检索，文本模糊查询，全表检索的数据；简单查询性能浪费，查询上限依赖机器数量。

>**Mysql：**复杂查询容易导致cpu标高性能问题，简单查询依赖读写量占比和读写分离。

>**Nosql：**复杂查询不支持，简单查询性能高，查询上限依赖于带宽和连接数。支持：关联相对独立的数据，更改量大， 简单查询量大的数据；缓冲流量洪峰，大量简单查询类的场景

##### 1.4 核心概念
(1) **cluster（集群）**：每个集群至少包含两个节点.  
(2) **node**：集群中的每个节点，一个节点不代表一台服务器  
(3) **field**：一个数据字段，与index和type一起，可以定位一个doc  
(4) **document**：ES最小的数据单元 Json  
```json
{
  "id": "1",
  "name": "小米",
  "price": {
     "标准版": 3999,
     "尊享版": 4999,
     "吴磊签名定制版": 19999
  }
}
```
（5）**Type**：逻辑上的数据分类，es 7.x中删除了type的概念  
（6）**Index**：一类相同或者类似的doc，比如一个员工索引，商品索引。  

**Shard分片：**  
1：一个index包含多个Shard，默认5P（6版本），默认每个P分配一个R，P的数量在创建索引的时候设置，如果想修改，需要重建索引。  
2：每个Shard都是一个Lucene实例，有完整的创建索引的处理请求能力。  
3：ES会自动在nodes上为我们做shard 均衡。  
4：一个doc是不可能同时存在于多个PShard中的，但是可以存在于多个RShard中。  

#### 2. 简单使用 CRUD
一个 Elasticsearch 请求和任何 HTTP 请求一样由若干相同的部件组成：  
`curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'`  

被 `< >` 标记的部件：  
 
| `VERB` | **适当的 HTTP _方法_ 或 _谓词_ : **`GET`**、 **`POST`**、 **`PUT`**、 **`HEAD`** 或者 **`DELETE`**。** |
| --- | --- |
| `PROTOCOL` | `http` 或者 `https`（如果你在 Elasticsearch 前面有一个 `https` 代理） |
| `HOST` | Elasticsearch 集群中任意节点的主机名，或者用 `localhost` 代表本地机器上的节点。 |
| `PORT` | 运行 Elasticsearch HTTP 服务的端口号，默认是 `9200` 。 |
| `PATH` | API 的终端路径（例如 `_count` 将返回集群中文档数量）。Path 可能包含多个组件，例如：`_cluster/stats` 和 `_nodes/stats/jvm` 。 |
| `QUERY_STRING` | 任意可选的查询字符串参数 (例如 `?pretty` 将格式化地输出 JSON 返回值，使其更容易阅读) |
| `BODY` | 一个 JSON 格式的请求体 (如果请求需要的话) |

&emsp;&emsp;**创建索引**  
类似与db中的update or create，- 如果对具有给定类型的文档进行索引，并且要插入原先不存在的ID。  
```
PUT或者POST /{index}/{type}/{id}
{
  "field": "value",
  ...
}
```
若不写{id}， 则es会自动生成一个id。自动生成的 ID 是 URL-safe、 基于 Base64 编码且长度为20个字符的 GUID 字符串。 这些 GUID 字符串由可修改的 FlakeID 模式生成，这种模式允许多个节点并行生成唯一 ID ，且互相之间的冲突概率几乎为零。  
&emsp;&emsp;**查询**，取回结果  
为了从 Elasticsearch 中检索出文档，我们仍然使用相同的 `_index` , `_type` , 和 `_id` ，但是 HTTP 谓词更改为 `GET` :
```
GET /{index}/{type}/{id}
```
加/_source 只返回 _source 端点；  
`HEAD` 代替 `GET` 能检查文档是否存在  
&emsp;&emsp;**更新文档**  
在 es中 文档 是 不可改变的，不能修改。直接使用**相同的id**进行更新就行；  
通过version字段查看文档版本，“更新”方式如下：  
- 从旧文档构建 JSON
- 更改该 JSON
- 删除旧文档
- 索引一个新文档

&emsp;&emsp;**删除**
```
DELETE /{index}/{type}/{id}
```

#### 3. Elasticsearch查询语法
##### 3.1 Search timeout
`GET /megacorp/employee/_search?timeout=1ms`  
> 默认没有timeout，如果设置了timeout，那么会执行timeout机制。 
> Timeout机制：假设用户查询结果有1W条数据，但是需要10″才能查询完毕，但是用户设置了1″的timeout，那么不管当前一共查询到了多少数据，都会在1″后ES讲停止查询，并返回当前数据。

##### 3.2 ES常用查询
Query_string:
- 查询所有：`GET /cargo_index/cargo/_search`
- 带参数：`GET /cargo_index/cargo/_search?q=channel:10`
-  分页：`GET /cargo_index/cargo/_search?from=0&size=2&sort=channel:desc`

**Query DSL**:
- **match_all**:匹配所有
```
GET /cargo_index/_search
{  
   "query": {  
      "match_all": {}  
   }  
}
```
-  **match**： name=“鱼肉”的
```
GET /cargo_index/_search
{ 
   "query": { 
      "match": {  
         "name": "鱼肉"  
      }  
   }  
}
```
- **sort**：按照channel（字段名）倒序排序
```
// 省略
"sort": [ 
  { 
     "channel": "desc"  
  } 
]
```

- **multi_match**：根据多个字段查询一个关键词
- **_source 元数据：**想要查询多个字段
- **分页（deep-paging）**关键字 `from`, `size`
- **Phrase search**: 关键字`match_phrase`
- **Query and filter**：查询和过滤
  + **must**：必须满足，子句（查询）必须出现在匹配的文档中，并将有助于得分。
  + **filter**：过滤器 不计算相关度分数，cache，子句（查询）必须出现在匹配的文档中。但是不像 `must`查询的分数将被忽略。Filter子句在filter上下文中执行，这意味着计分被忽略，并且子句被考虑用于缓存。
  + **should**：可能满足 or，子句（查询）应出现在匹配的文档中。
  + **must_not**：必须不满足 不计算相关度分数 not，子句（查询）不得出现在匹配的文档中。子句在过滤器上下文中执行，这意味着计分被忽略，并且子句被视为用于缓存。由于忽略计分，`0`因此将返回所有文档的分数。
  + **minimum_should_match**：参数指定should返回的文档必须匹配的子句的数量或百分比。如果bool查询包含至少一个should子句，而没有must或 filter子句，则默认值为1。否则，默认值为0
  + **Highlight search**
  + **组合查询**：bool


### 补充：
#### 关系型数据库（Mysql） 与 Elastic Search 对应关系
在Elasticsearch中，索引是类型的集合，因为数据库是RDBMS(关系数据库管理系统)中表的集合。每个表都是行的集合，就像每个映射都是JSON对象的Elasticsearch集合一样。

| **Elasticsearch** | **关系型数据库** |
| --- | --- |
| 索引 | 数据库 |
| 碎片 | 碎片 |
| 映射 | 表 |
| 字段 | 字段 |
| JSON对象 | 元组 |

#### es搭建注意事项
##### 1. can not run elasticsearch as root,不能以root用户启动，需要切换成其他用户，据说因为安全
> 新增用户 adduser elk  
> 更改es文件夹的拥有者： chown -R elk es文件夹  
> 启动： su elk  

##### 2. es启动在前台，ctrl+c会终止进程
解决方法：
- 用`sh ./ bin/elasticsearch -d`来后台启动es的。
- 再起一个远程连接，进行操作

##### 3. `max virtual memory area vm.max_map_count [65530] is too low`问题，用户权限问题
（待尝试）解决方法（针对第1条）：先执行`sysctl -w vm.max_map_count=262144`（具体的值可以根据服务器配置修改下，2的n次方），然后在`/etc/sysctl.conf`文件最后添加一行`vm.max_map_count=262144`，使永久生效。

##### 4. 可视化操作界面kibana安装与使用
解压，配置es地址，后就可以使用
![es查询](/assets/img/2023-01-20-Elasticsearch基础/es查询.png)
#### 注意：es 不是实时的，官方解释：[Es查询不是实时的原因](https://www.elastic.co/guide/cn/elasticsearch/guide/current/near-real-time.html)
 
