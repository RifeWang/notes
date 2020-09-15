# 存储引擎
存储引擎由以下部分组成：
- In-Memory Index : 跨分片的共享索引，可以快速访问 measurements, tags, series
- WAL : 写优化存储格式
- Cache : WAL 数据的缓存，它在运行时查询并与存储在TSM文件中的数据合并。
- TSM Files : 以列式格式存储压缩系列数据。
- FileStore : 调节对磁盘上所有TSM文件的访问。它确保在替换现有TSM文件时以原子方式安装TSM文件以及删除不再使用的TSM文件。
- Compactor : 压缩器负责将优化程度较低的Cache和TSM数据转换为更多读取优化的格式。它通过压缩序列，删除已删除的数据，优化索引以及将较小的文件组合成较大的文件来实现。
- Compaction Planner : 确定哪些TSM文件已准备好进行压缩，并确保多个并发压缩不会相互干扰。
- Compression: 对于特定数据类型，压缩由各种编码器和解码器处理。有些编码器是相当静态的，并且总是以相同的方式编码相同的类型;其他人根据数据的形状切换压缩策略。
- Writers/Readers : 每种文件类型（WAL段，TSM文件，逻辑删除等）都有Writers和Readers用于处理格式。

# 硬件配置
集群是商业版的，这里只看单机版的硬件配置。

通过三个指标去定义负载的情况：
- `the number of field writes per second`: 每秒写入。
- `the number of queries per second`: 每秒查询。
- `the number of unique series`: series 基数。

查询可大致分为三类：
- 简单查询：
    - 几乎没有使用函数和正则表达式
    - 时间范围在几分钟，几小时，或者一天之内
    - 执行时间通常在几毫秒到几十毫秒
- 中等查询：
    - 使用了多个函数和一两个正则表达式
    - 可能使用了复杂的 `GROUP BY` 语，或者时间范围有几个星期
    - 执行时间通常在几百毫秒到几千毫秒
- 复杂查询：
    - 使用了多个聚合函数、转换函数或者多个正则表达式
    - 时间跨度很大，几个月或是几年
    - 通常执行时间需要几秒

不同的负载情况以及推荐的硬件配置（ 针对 TSM ，TSI 未测试 ）：

负载                |   每秒写入 |  每秒中等查询 |  series 基数 |  硬件配置
---- | ---- |  ---- | ---- | ----
低	                |  < 5 千   |  < 5        |  < 10 万    | CPU: 2-4 核, RAM: 2-4 GB , IOPS: 500
中等                |	< 25 万  | < 25        |  < 1 百万   | CPU: 4-6 核, RAM: 8-32 GB , IOPS: 500-1000
高	                |   > 25 万 |  > 25       |  > 1 百万   | CPU: 8+ 核, RAM: 32+ GB , IOPS: 1000+
可能不可行           |   > 75 万  | > 5        |  > 1 千万     

IOPS ( Input/Output Operations Per Second )：每秒读写数，衡量存储设备（如 SSD 固态硬盘，HDD 机械硬盘，SAN 存储区域网络）的性能指标。

上述配置都是基于 SSD 的，如果你使用 HDD 的话性能会低一个数量级，官方建议你使用 SSD 。

database name, measurement, tag key, tag value, field key 都是作为元数据只存储一次。只有 field value 和 timestamp 每个 point 都存储。非字符串的值大约需要三个字节，字符串的值需要的空间大小不固定，需要由压缩情况确定。


一般来讲，内存越多，查询的速度越快，增加更多的内存总没有坏处。
影响内存的最主要的因素是series基数，series的基数大约或是超过千万时，就算有更多的内存也可能导致OOM，所以在设计数据的格式的时候需要考虑到这一点。
内存的增长和series的基数存在一个指数级的关系。

如果将 wal 和 data 目录分开到不同的存储设备上，有利于减少磁盘争用，从而应对更高的写入负载。
```
[data]
    dir = "/var/lib/influxdb/data"
    wal-dir = "/var/lib/influxdb/wal"
```