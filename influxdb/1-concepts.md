# 核心概念
- `database`
- `retention policy`: 数据的保留时间 + 集群中的副本数 + shard duration ，默认 autogen 策略，数据不过期且副本数为1
- `series`: 一个系列由同一个 retention policy、同一个 measurement、同一个 tag set 组成
- `point`: 一个点由以下四个部分组成
- `measurement`: 包含 tags, fields, time 的容器
- `tag set` : `tag key` = `tag value`
- `field set` : `field key` = `field value`
- `timestamp`

# 词汇表
- `aggregation`: 聚合
- `batch`
- `continuous query (CQ)`: 自动定期运行查询，连续查询必须有 `GROUP BY time()`
- `database`
- `duration`: RP策略定义的数据存储时长，过期的数据会被删除
- `field`
- `field key`
- `field set`
- `field value`
- `function`
- `identifier`: token
- `InfluxDB line protocol`
- `measurement`
- `metastore`: 包含了系统内部信息
- `node`: 一个独立的 influxd 进程
- `now()`: 服务器本地时间 纳秒
- `point`
- `points per second`
- `query`
- `replication factor`: 数据副本数量，集群模式才有效
- `retention policy（RP）`
- `schema`
- `selector`
- `series`
- `series cardinality`
- `server`
- `shard`: 每个分片存储实际的编码和压缩后的数据，对应一个磁盘上的 TSM 文件。每个分片从属于唯一一个 shard group ，多个 shard 可能存在于一个 shard group 中，每个 shard 包含一组特定的 series 。属于给定分片组中给定系列的所有点都将存储在磁盘上的同一分片（TSM文件）中。
- `shard duration`: shard group 划分的时间范围。
- `shard group`: 逻辑上的概念，包含有多个 shard 。
- `subscription`
- `tag`
- `tag key`
- `tag set`
- `tag value`
- `timestamp`
- `transformation`
- `tsm (Time Structured Merge tree)`: 特定存储格式
- `user`
- `values per second`
- `wal (Write Ahead Log)`: 预写缓存

# 比较 InfluxDB 和 SQL 数据库
InfluxDB 就是被设计用于处理时间序列的数据。SQL 数据库虽然可以处理时间序列的数据，但并不是专门以此为目标。InfluxDB 可以更高效快速的存储大量时间序列数据并实时分析这些数据。

时间是绝对的主角，InfluxDB 不需要预先定义数据格式，支持 continuous queries 和 retention policies ，不支持跨 measurement 的 JOIN 查询，时间戳必须是 UNIX epoch（GMT）或者 RFC3339 格式字符串。

InfluxQL 和 SQL 非常相似。

InfluxDB 不是 CRUD：
- 更新 point 只需要插入相同 measurement、tag set、timestamp
- 你可以删除 series，但是不能基于 field 值去删除独立的 points，作为解决方法，你需要查询 field 值的时间，然后根据时间删除
- 你无法更新或重命名 tags ，只能创建新的并导入数据然后删除老的。
- 你无法通过 tag key 删除 tag

# InfluxDB 设计之道和权衡
https://docs.influxdata.com/influxdb/v1.7/concepts/insights_tradeoffs/


# 数据格式设计

Tags 会被索引，fields 不会索引，tags 查询性能更高。
- 常用的查询数据存储为 tags
- 计划使用 `GROUP BY()` 的数据存储为 tags
- 计划使用 InfluxQL function 的数据存储为 fields
- 数据不只是 string 类型的存储为 fields ，tags 总是 string

避免使用 InfluxQL keywords 作为识别名称

- 不要有大量的 series : 使用 UUID, hash, 随机字符串的 tags 将会导致数量庞大的 series，这也是导致高内存使用率的主要因素。如果系统内存有限制，对于大基数 series 应该使用 field
- measurement 名称不要包含数据，使用不同的 tags 区分数据而不是 measurement 名称
- 不要在一个 tag 中放置多条信息，拆分有助于简化查询并减少使用正则

### shard group duration
数据被存放在 shard group 中，shard group 根据 retention policy 进行组织，并使用时间戳存储特定时间间隔内的数据，时间间隔就是 shard group duration 。

默认值：
RP Duration                 |   Shard Group Duration
---- | ----
< 2 days	                |   1 hour
>= 2 days and <= 6 months   |	1 day
> 6 months	                |   7 days

Long shard group duration 性能更好，可以存储更大数据在相同的逻辑位置上，这减少了数据重复，提高了压缩效率，并在某些情况下查询更快。
Short shard group duration 允许系统更有效的丢弃数据并记录增量备份，强制执行 RP 时丢弃的是整个 shard groups 而不是单个数据点。

高吞吐量或长时间运行的实例将受益于使用更长的分片组持续时间。
建议配置：
RP Duration	             |   Shard Group Duration
--- | ----
<= 1 day	             |   6 hours
> 1 day and <= 7 days	 |   1 day
> 7 days and <= 3 months |	 7 days
> 3 months	             |   30 days
infinite	             |   52 weeks or longer

- shard group 应该是最频繁查询的最长时间范围的两倍
- 每个 shard group 应该包含超过 100,000 points
- shard group 每个 series 应该包含超过 1,000 points

对数百或数千个分片的并发访问和开销很快就会导致性能降低和内存耗尽，例如大量写入历史数据的情况，强烈建议临时设置较长的 shard group duration 以便创建更少的 shard ，通常 52 周时间比较合适。

# In-memory indexing and the Time-Structured Merge Tree (TSM)
InfluxDB 为每个时间块创建一个 shard 分片，每个分片映射到一个底层存储引擎数据库，每个数据库有自己独立的 WAL 和 TSM 文件。

### Storage engine
存储引擎由以下部分组成：
- In-Memory Index : 跨分片的共享索引，可以快速访问 measurements, tags, series
- WAL : 写优化存储格式
- Cache : WAL 数据的缓存，它在运行时查询并与存储在TSM文件中的数据合并。
- TSM Files : 以列式格式存储压缩系列数据
- FileStore : 调解对磁盘上所有TSM文件的访问。它确保在替换现有TSM文件时以原子方式安装TSM文件以及删除不再使用的TSM文件。
- Compactor : 压缩器负责将优化程度较低的Cache和TSM数据转换为更多读取优化的格式。它通过压缩序列，删除已删除的数据，优化索引以及将较小的文件组合成较大的文件来实现。
- Compaction Planner : 确定哪些TSM文件已准备好进行压缩，并确保多个并发压缩不会相互干扰。
- Compression: 对于特定数据类型，压缩由各种编码器和解码器处理。有些编码器是相当静态的，并且总是以相同的方式编码相同的类型;其他人根据数据的形状切换压缩策略。
- Writers/Readers : 每种文件类型（WAL段，TSM文件，逻辑删除等）都有Writers和Readers用于处理格式。

WAL: 一系列的形如 _000001.wal 的文件集合，序号是单调递增的，单个文件就是一个 segment 片段，每个片段 10MB 。写入 WAL 是 `fsync` 并且会添加到 in-memory index ，使用 batch 效率更快，建议 5000 - 10000 points 每个 batch 。

Cache: 缓存 WAL 数据。`cache-snapshot-memory-size` 超出内存大小则刷盘到 TSM 文件并删除对应的 WAL segments ，配合 `cache-snapshot-write-cold-duration` 。`cache-max-memory-size` 超出内存大小会导致 cache 拒绝新的写入。这些配置用来防止内存不足以及客户端写入压力比实例的存储速度更快。

TSM files: 只读文件

# Time Series Index (TSI)
对于大基数的 series 而言，不可能用 in-memory 存储索引，TSI 可以处理千万级别的 series 。