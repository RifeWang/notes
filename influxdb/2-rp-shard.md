`/meta` 目录：存储元数据，`meta.db` 文件

`/data` 目录：存储最终数据（TSM files）
```
—— data
    —— database
        —— _series
        —— RP
            —— shard
                —— .tsm 数据文件
                —— .idx 文件
                —— index
                    —— 索引分隔目录
                        —— .tsl 索引文件
```

`/wal` 目录：存储预写文件（WAL files）
```
|—— wal/
    |—— database/
        |—— RP/
            |—— shard/
                |—— .wal 预写文件
```               
                
---
1、数据分片：
- `shard`: 每个分片存储实际的编码和压缩后的数据，对应一个磁盘上的 TSM 文件。每个分片从属于唯一一个 shard group ，多个 shard 可能存在于一个 shard group 中，每个 shard 包含一组特定的 series 。属于给定分片组中给定系列的所有点都将存储在磁盘上的同一分片（TSM文件）中。
- `shard duration`: shard group 划分的时间范围。
- `shard group`: 逻辑上的概念，包含有多个 shard 。

---
2、RP 策略管理

默认值：
RP Duration                 |   Shard Group Duration
---- | ----
< 2 days	                |   1 hour
>= 2 days and <= 6 months   |	1 day
> 6 months	                |   7 days

---
3、shard group duration 管理

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