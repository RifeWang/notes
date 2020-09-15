# 数据格式设计

支持的数据类型
- measurement: string
- field key: string
- field value: string, float, integer, boolean
- tag key: string
- tag value: string
- timestamp: InfluxDB 中都是 UTC 时间，且默认纳秒精度

Tags 会被索引，fields 不会索引，tags 查询性能更高。
- 常用的查询数据存储为 tags
- 计划使用 `GROUP BY()` 的数据存储为 tags
- 计划使用 InfluxQL function 的数据存储为 fields
- 数据不只是 string 类型的存储为 fields ，tags 总是 string

避免使用 InfluxQL keywords 作为识别名称

- 不要有大量的 series : 使用 UUID, hash, 随机字符串的 tags 将会导致数量庞大的 series，这也是导致高内存使用率的主要因素。如果系统内存有限制，对于大基数 series 应该使用 field
- measurement 名称不要包含数据，使用不同的 tags 区分数据而不是 measurement 名称
- 不要在一个 tag 中放置多条信息，拆分有助于简化查询并减少使用正则


# in-memory / tsi 索引
config 文件配置，对于大基数的 series 而言，不可能用 in-memory 存储索引，TSI 可以处理千万级别的 series 。

inmem index 相关：
- `max-series-per-database = 1000000`: 每个数据库允许写入的最大 series 数量，若超出会删除老的数据，设置为 0 可以禁用，达到上限数量再写入则会触发 500 错误。
- `max-values-per-tag = 100000`: 每个 tag key 允许的最大 tag values 数量，超出触发删除，设置为 0 可以禁用，达到上限不会写入。

tsi1 index 相关：
- `max-index-log-file-size = "1m"`: WAL文件压缩成索引文件的阈值大小，会影响写入性能，较小的大小将导致日志文件更快地压缩，并导致较低的堆使用率，但会降低写入吞吐量。较高的大小将被更不频繁地压缩，在内存中存储更多的序列，并提供较高的写入吞吐量。
- `series-id-set-cache-size = 100`: 使用缓存的大小，影响查询性能和 heap usage
