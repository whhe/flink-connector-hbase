---
title: HBase
weight: 9
type: docs
aliases:
  - /zh/dev/table/connectors/hbase.html
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# HBase SQL 连接器

{{< label "Scan Source: Bounded" >}}
{{< label "Lookup Source: Sync Mode" >}}
{{< label "Sink: Batch" >}}
{{< label "Sink: Streaming Upsert Mode" >}}

HBase 连接器支持读取和写入 HBase 集群。本文档介绍如何使用 HBase 连接器基于 HBase 进行 SQL 查询。

HBase 连接器在 upsert 模式下运行，可以使用 DDL 中定义的主键与外部系统交换更新操作消息。但是主键只能基于 HBase 的 rowkey 字段定义。如果没有声明主键，HBase 连接器默认取 rowkey 作为主键。

依赖
------------

{{< sql_connector_download_table "hbase" >}}

HBase 连接器不是二进制发行版的一部分，请查阅[这里]({{< ref "docs/dev/configuration/overview" >}})了解如何在集群运行中引用 HBase 连接器。

如何使用 HBase 表
----------------

所有 HBase 表的列簇必须定义为 ROW 类型，字段名对应列簇名（column family），嵌套的字段名对应列限定符名（column qualifier）。用户只需在表结构中声明查询中使用的的列簇和列限定符。除了 ROW 类型的列，剩下的原子数据类型字段（比如，STRING, BIGINT）将被识别为 HBase 的 rowkey，一张表中只能声明一个 rowkey。rowkey 字段的名字可以是任意的，如果是保留关键字，需要用反引号。

```sql
-- 在 Flink SQL 中注册 HBase 表 "mytable"
CREATE TABLE hTable (
 rowkey INT,
 family1 ROW<q1 INT>,
 family2 ROW<q2 STRING, q3 BIGINT>,
 family3 ROW<q4 DOUBLE, q5 BOOLEAN, q6 STRING>,
 PRIMARY KEY (rowkey) NOT ENFORCED
) WITH (
 'connector' = 'hbase-1.4',
 'table-name' = 'mytable',
 'zookeeper.quorum' = 'localhost:2181'
);

-- 用 ROW(...) 构造函数构造列簇，并往 HBase 表写数据。
-- 假设 "T" 的表结构是 [rowkey, f1q1, f2q2, f2q3, f3q4, f3q5, f3q6]
INSERT INTO hTable
SELECT rowkey, ROW(f1q1), ROW(f2q2, f2q3), ROW(f3q4, f3q5, f3q6) FROM T;

-- 从 HBase 表扫描数据
SELECT rowkey, family1, family3.q4, family3.q6 FROM hTable;

-- temporal join HBase 表，将 HBase 表作为维表
SELECT * FROM myTopic
LEFT JOIN hTable FOR SYSTEM_TIME AS OF myTopic.proctime
ON myTopic.key = hTable.rowkey;
```

连接器参数
----------------

<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left" style="width: 25%">参数</th>
        <th class="text-center" style="width: 10%">是否必选</th>
        <th class="text-center" style="width: 10%">默认值</th>
        <th class="text-center" style="width: 10%">数据类型</th>
        <th class="text-center" style="width: 45%">描述</th>
      </tr>
    </thead>
    <tbody>
    <tr>
      <td><h5>connector</h5></td>
      <td>必选</td>
      <td style="word-wrap: break-word;">(none)</td>
      <td>String</td>
      <td>指定使用的连接器, 支持的值如下 :
        <ul>
            <li><code>hbase-1.4</code>: 连接 HBase 1.4.x 集群</li>
            <li><code>hbase-2.2</code>: 连接 HBase 2.2.x 集群</li>
        </ul>
      </td>
    </tr>
    <tr>
      <td><h5>table-name</h5></td>
      <td>必选</td>
      <td style="word-wrap: break-word;">(none)</td>
      <td>String</td>
      <td>连接的 HBase 表名。默认该表在 "default" 命名空间下，指定命名空间下的表需要使用 "namespace:table"。</td>
    </tr>
    <tr>
      <td><h5>zookeeper.quorum</h5></td>
      <td>必选</td>
      <td style="word-wrap: break-word;">(none)</td>
      <td>String</td>
      <td>HBase Zookeeper quorum 信息。</td>
    </tr>
    <tr>
      <td><h5>zookeeper.znode.parent</h5></td>
      <td>可选</td>
      <td style="word-wrap: break-word;">/hbase</td>
      <td>String</td>
      <td>HBase 集群的 Zookeeper 根目录。</td>
    </tr>
    <tr>
      <td><h5>null-string-literal</h5></td>
      <td>可选</td>
      <td style="word-wrap: break-word;">null</td>
      <td>String</td>
      <td>当字符串值为 <code>null</code> 时的存储形式，默认存成 "null" 字符串。HBase 的 source 和 sink 的编解码将所有数据类型（除字符串外）将 <code>null</code> 值以空字节来存储。</td>
    </tr>
    <tr>
      <td><h5>sink.buffer-flush.max-size</h5></td>
      <td>可选</td>
      <td style="word-wrap: break-word;">2mb</td>
      <td>MemorySize</td>
      <td>写入的参数选项。每次写入请求缓存行的最大大小。它能提升写入 HBase 数据库的性能，但是也可能增加延迟。设置为 "0" 关闭此选项。
      </td>
    </tr>
    <tr>
      <td><h5>sink.buffer-flush.max-rows</h5></td>
      <td>可选</td>
      <td style="word-wrap: break-word;">1000</td>
      <td>Integer</td>
      <td>写入的参数选项。 每次写入请求缓存的最大行数。它能提升写入 HBase 数据库的性能，但是也可能增加延迟。设置为 "0" 关闭此选项。
      </td>
    </tr>
    <tr>
      <td><h5>sink.buffer-flush.interval</h5></td>
      <td>可选</td>
      <td style="word-wrap: break-word;">1s</td>
      <td>Duration</td>
      <td>写入的参数选项。刷写缓存行的间隔。它能提升写入 HBase 数据库的性能，但是也可能增加延迟。设置为 "0" 关闭此选项。注意："sink.buffer-flush.max-size" 和 "sink.buffer-flush.max-rows" 同时设置为 "0"，刷写选项整个异步处理缓存行为。
      </td>
    </tr>
    <tr>
      <td><h5>sink.parallelism</h5></td>
      <td>可选</td>
      <td style="word-wrap: break-word;">(none)</td>
      <td>Integer</td>
      <td>为 HBase sink operator 定义并行度。默认情况下，并行度由框架决定，和链在一起的上游 operator 一样。</td>
    </tr>
    <tr>
      <td><h5>lookup.async</h5></td>
      <td>可选</td>
      <td style="word-wrap: break-word;">false</td>
      <td>Boolean</td>
      <td>是否启用异步查找。如果为真，查找将是异步的。注意：异步方式只支持 hbase-2.2 连接器</td>
    </tr>
    <tr>
      <td><h5>lookup.cache</h5></td>
      <td>可选</td>
      <td style="word-wrap: break-word;">NONE</td>
      <td><p>枚举类型</p>可选值: NONE, PARTIAL</td>
      <td>维表的缓存策略。 目前支持 NONE（不缓存）和 PARTIAL（只在外部数据库中查找数据时缓存）。</td>
    </tr>
    <tr>
      <td><h5>lookup.partial-cache.max-rows</h5></td>
      <td>可选</td>
      <td style="word-wrap: break-word;">(none)</td>
      <td>Long</td>
      <td>查找缓存的最大行数，超过这个值，最旧的行将过期。使用该配置时 "lookup.cache" 必须设置为 "PARTIAL”。</td>
    </tr>
    <tr>
      <td><h5>lookup.partial-cache.expire-after-write</h5></td>
      <td>可选</td>
      <td style="word-wrap: break-word;">(none)</td>
      <td>Duration</td>
      <td>在记录写入缓存后该记录的最大保留时间。
      使用该配置时 "lookup.cache" 必须设置为 "PARTIAL”。</td>
    </tr>
    <tr>
      <td><h5>lookup.partial-cache.expire-after-access</h5></td>
      <td>可选</td>
      <td style="word-wrap: break-word;">(none)</td>
      <td>Duration</td>
      <td>在缓存中的记录被访问后该记录的最大保留时间。
      使用该配置时 "lookup.cache" 必须设置为 "PARTIAL”。</td>
    </tr>
    <tr>
      <td><h5>lookup.partial-cache.caching-missing-key</h5></td>
      <td>可选</td>
      <td style="word-wrap: break-word;">true</td>
      <td>Boolean</td>
      <td>是否缓存维表中不存在的键，默认为true。
        使用该配置时 "lookup.cache" 必须设置为 "PARTIAL”。</td>
    </tr>
    <tr>
      <td><h5>lookup.max-retries</h5></td>
      <td>可选</td>
      <td style="word-wrap: break-word;">3</td>
      <td>Integer</td>
      <td>查找数据库失败时的最大重试次数。</td>
    </tr>
    <tr>
      <td><h5>properties.*</h5></td>
      <td>可选</td>
      <td style="word-wrap: break-word;">(无)</td>
      <td>String</td>
      <td>
         可以设置任意 HBase 的配置项。后缀名必须匹配在 <a href="http://hbase.apache.org/2.3/book.html#hbase_default_configurations">HBase 配置文档</a> 中定义的配置键。Flink 将移除 "properties." 配置键前缀并将变换后的配置键和值传入底层的 HBase 客户端。
         例如您可以设置 <code>'properties.hbase.security.authentication' = 'kerberos'</code> 等kerberos认证参数。
      </td>
    </tr>
    </tbody>
</table>

### 已弃用的配置
这些弃用配置已经被上述的新配置代替，而且最终会被弃用。请优先考虑使用新配置。
<table>
    <thead>
      <tr>
        <th class="text-left" style="width: 25%">Option</th>
        <th class="text-left" style="width: 8%">Required</th>
        <th class="text-left" style="width: 8%">Forwarded</th>
        <th class="text-left" style="width: 7%">Default</th>
        <th class="text-left" style="width: 10%">Type</th>
        <th class="text-left" style="width: 42%">Description</th>
      </tr>
    </thead>
    <tbody>
        <tr>
          <td><h5>lookup.cache.max-rows</h5></td>
          <td>optional</td>
          <td>yes</td>
          <td style="word-wrap: break-word;">(none)</td>
          <td>Integer</td>
          <td>请配置 "lookup.cache" = "PARTIAL" 并使用 "lookup.partial-cache.max-rows" 代替</td>
        </tr>
        <tr>
          <td><h5>lookup.cache.ttl</h5></td>
          <td>optional</td>
          <td>yes</td>
          <td style="word-wrap: break-word;">(none)</td>
          <td>Duration</td>
          <td>请配置 "lookup.cache" = "PARTIAL" 并使用 "lookup.partial-cache.expire-after-write" 代替</td>
        </tr>
    </tbody>
<table>



数据类型映射表
----------------

HBase 以字节数组存储所有数据。在读和写过程中要序列化和反序列化数据。

Flink 的 HBase 连接器利用 HBase（Hadoop) 的工具类 `org.apache.hadoop.hbase.util.Bytes` 进行字节数组和 Flink 数据类型转换。

Flink 的 HBase 连接器将所有数据类型（除字符串外）`null` 值编码成空字节。对于字符串类型，`null` 值的字面值由`null-string-literal`选项值决定。

数据类型映射表如下：

<table class="table table-bordered">
    <thead>
      <tr>
        <th class="text-left">Flink 数据类型</th>
        <th class="text-left">HBase 转换</th>
      </tr>
    </thead>
    <tbody>
    <tr>
      <td><code>CHAR / VARCHAR / STRING</code></td>
      <td>
{{< highlight "java" >}}
byte[] toBytes(String s)
String toString(byte[] b)
{{< /highlight >}}
      </td>
    </tr>
    <tr>
      <td><code>BOOLEAN</code></td>
      <td>
{{< highlight "java" >}}
byte[] toBytes(boolean b)
boolean toBoolean(byte[] b)
{{< /highlight >}}
      </td>
    </tr>
    <tr>
      <td><code>BINARY / VARBINARY</code></td>
      <td>返回 <code>byte[]</code>。</td>
    </tr>
    <tr>
      <td><code>DECIMAL</code></td>
      <td>
{{< highlight "java" >}}
byte[] toBytes(BigDecimal v)
BigDecimal toBigDecimal(byte[] b)
{{< /highlight >}}
      </td>
    </tr>
    <tr>
      <td><code>TINYINT</code></td>
      <td>
{{< highlight "java" >}}
new byte[] { val }
bytes[0] // returns first and only byte from bytes
{{< /highlight >}}
      </td>
    </tr>
    <tr>
      <td><code>SMALLINT</code></td>
      <td>
{{< highlight "java" >}}
byte[] toBytes(short val)
short toShort(byte[] bytes)
{{< /highlight >}}
      </td>
    </tr>
    <tr>
      <td><code>INT</code></td>
      <td>
{{< highlight "java" >}}
byte[] toBytes(int val)
int toInt(byte[] bytes)
{{< /highlight >}}
      </td>
    </tr>
    <tr>
      <td><code>BIGINT</code></td>
      <td>
{{< highlight "java" >}}
byte[] toBytes(long val)
long toLong(byte[] bytes)
{{< /highlight >}}
      </td>
    </tr>
    <tr>
      <td><code>FLOAT</code></td>
      <td>
{{< highlight "java" >}}
byte[] toBytes(float val)
float toFloat(byte[] bytes)
{{< /highlight >}}
      </td>
    </tr>
    <tr>
      <td><code>DOUBLE</code></td>
      <td>
{{< highlight "java" >}}
byte[] toBytes(double val)
double toDouble(byte[] bytes)
{{< /highlight >}}
      </td>
    </tr>
    <tr>
      <td><code>DATE</code></td>
      <td>从 1970-01-01 00:00:00 UTC 开始的天数，int 值。</td>
    </tr>
    <tr>
      <td><code>TIME</code></td>
      <td>从 1970-01-01 00:00:00 UTC 开始天的毫秒数，int 值。</td>
    </tr>
    <tr>
      <td><code>TIMESTAMP</code></td>
      <td>从 1970-01-01 00:00:00 UTC 开始的毫秒数，long 值。</td>
    </tr>
    <tr>
      <td><code>ARRAY</code></td>
      <td>不支持</td>
    </tr>
    <tr>
      <td><code>MAP / MULTISET</code></td>
      <td>不支持</td>
    </tr>
    <tr>
      <td><code>ROW</code></td>
      <td>不支持</td>
    </tr>
    </tbody>
</table>

{{< top >}}
