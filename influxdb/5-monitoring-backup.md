# 服务器监控
InfluxDB 可以显示节点的统计和诊断信息，有助于故障排除和性能监控。

## SHOW STATS
`SHOW STATS` 返回的统计信息只存在内存中，一旦重启会重新记录。会存储到 _internal 数据库中。

## SHOW DIAGNOSTICS
`SHOW DIAGNOSTICS`，诊断信息，包括：构建版本信息，正常运行时间，主机名，服务器配置，内存使用情况以及Go运行时。不一定是数字格式，不会存储到 _internal 数据库中。
https://docs.influxdata.com/platform/monitoring/influxdata-platform/tools/show-diagnostics/

## Internal monitoring
InfluxDB 会将统计和诊断信息写入到 `_internal` 数据库中，记录有关内部运行时和服务性能的指标。

[monitor] 自有监控系统。默认情况下，InfluxDB把这些数据写入_internal 数据库，如果这个库不存在则自动创建。 _internal 库默认使用名称为 monitor 的 RP，数据保留7天，如果你想使用一个自己的retention策略，需要自己创建。
- `store-enabled = true`: 是否记录统计数据
- `store-database = "_internal"`: 存储统计数据的数据库
- `store-interval = "10s"`: 统计间隔时间

`SHOW STATS [ FOR '<component>' | 'indexes' ]`

1、`use _internal` 使用数据库
2、`show measurements` 显示采集消息分类 component
3、`show stats for 'queryExecutor'` 查询指定 component 的统计信息

`show stats for 'indexes'` 返回所有索引使用的内存大小预估值

## Useful performance metrics commands
性能指标相关命令。

查询每秒写入的点数，必须启用 `monitor` 服务：
`influx -execute 'select derivative(pointReq, 1s) from "write" where time > now() - 5m' -database '_internal' -precision 'rfc3339'`

通过日志文件，查询每个 database 的写入数：
`grep 'POST' /var/log/influxdb/influxd.log | awk '{ print $10 }' | sort | uniq -c`

或者对于 systemd 记录到 journald 的：
`journalctl -u influxdb.service | awk '/POST/ { print $10 }' | sort | uniq -c`

## InfluxDB /metrics HTTP endpoint
`/metrics` 接口生成 Prometheus 指标格式的 Go 指标，比如垃圾回收、内存分配等等。

`/debug/vars` 返回 _internal 数据



# backup & restore 在线备份和恢复

### 配置远程连接
在线备份和恢复是通过TCP连接进行的。

远程备份需要修改 influxdb.conf 配置文件中的 `bind-address = "127.0.0.1:8088"` 为本机在网络上可通信的地址。
然后，命令通过 `-host` 参数提供地址：
    `influxd backup -portable -database mydatabase -host <remote-node-IP>:8088 /tmp/mysnapshot`

本地备份则不需要。

#### backup 命令：
```
influxd backup
    [ -database <db_name> ]
    [ -portable ]
    [ -host <host:port> ]
    [ -retention <rp_name> ] | [ -shard <shard_ID> -retention <rp_name> ]
    [ -start <timestamp> [ -end <timestamp> ] | -since <timestamp> ]
    <path-to-backup>
```

- `[ -database <db_name> ]`: 要备份的数据库，未指定则备份所有数据库。
- `[ -portable ]`: 使用新的兼容商业版的备份格式，强烈建议使用。
- `[ -host <host:port> ]`: InfluxDB 实例的地址和端口，如果是远程连接则需要。
- `[ -retention <rp_name> ]`: 需要备份的 RP 策略，若指明了需要 `-database` 也指明，未指定则备份所有 RP 。
- `[ -shard <ID> ]`: 需要备份的 Shard ID , 若指明了则也需要 `-retention <name>` 。
- `[ -start <timestamp>]`: 开始时间，不兼容 -since ，RFC3339 格式，如 `-start 2015-12-24T08:12:23Z`
- `[ -end <timestamp> ]`: 结束时间，开始时间未指定则默认为 1970-01-01 ，不兼容 -since 。
- `[ -since <timestamp> ]`: 使用 -start 替代，除非需要支持老版本的备份。

`influxd backup -portable <path-to-backup>`
`influxd backup -portable -start <timestamp> <path-to-backup>`
`influxd backup -portable -database telegraf <path-to-backup>`
`influxd backup -portable -database mytsd -start 2017-04-28T06:49:00Z -end 2017-04-28T06:50:00Z /tmp/backup/influxdb`

#### restore 命令：
```
influxd restore [ -db <db_name> ]
    -portable | -online
    [ -host <host:port> ]
    [ -newdb <newdb_name> ]
    [ -rp <rp_name> ]
    [ -newrp <newrp_name> ]
    [ -shard <shard_ID> ]
    <path-to-backup-files>
```

- `-portable | -online`: 备份格式，与 backup 一致。
- `[ -host <host:port> ]`: InfluxDB 实例的地址和端口，如果是远程连接则需要。
- `[ -db <db_name> | -database <db_name> ]`: 要从备份还原的数据库的名称。如果未指定，将还原所有数据库。
- `[ -newdb <newdb_name> ]`: 恢复到新的数据库，未指定则与 -db 一致。
- `[ -rp <rp_name> ]`: 要还原的备份中保留策略的名称。要求设置-db。如果未指定，将使用所有保留策略。
- `[ -newrp <newrp_name> ]`: 要在目标系统上创建的保留策略的名称。要求设置-rp。如果未指定，则使用-rp值。
- `[ -shard <shard_ID> ]`: 要恢复的分片的分片ID。如果指定，则需要-db和-rp。

`influxd restore -portable path-to-backup`
`influxd restore -portable -db telegraf path-to-backup`

无法直接恢复到一个已经存在的数据库或RP中，你需要先导入到一个临时的数据库中，然后重新插入到已有的数据库中（比如使用 select ... into 语句）

`influxd restore -portable -db telegraf -newdb telegraf_bak path-to-backup`

```
> USE telegraf_bak
> SELECT * INTO telegraf..:MEASUREMENT FROM /.*/ GROUP BY *
> DROP DATABASE telegraf_bak
```

数据过多会导致超时的情况，这时使用 where 过滤时间：
`SELECT * INTO "db"."rp"."measurement" FROM "db_bak"."rp"."measurement"  where time>now()-1d GROUP BY *`

恢复到已经存在的 RP 中：

`influxd restore -portable -db telegraf -newdb telegraf_bak -rp autogen -newrp autogen_bak path-to-backup`

```
> USE telegraf_bak
> SELECT * INTO telegraf.autogen.:MEASUREMENT FROM /telegraf_bak.autogen_bak.*/ GROUP BY *
> DROP DATABASE telegraf_bak
```

所有备份都是完整备份，不支持增量备份。

-start 和 -end 参数指定的时间间隔的备份是在数据块上执行的，并不是逐点执行。由于大多数块都是高度压缩的，因此提取每个块以检查每个点都会给正在运行的系统造成计算和磁盘空间负担。每个数据块都以该块中包含的时间间隔的开始和结束时间戳记进行注释。当您指定 -start 或 -end 时，将备份所有指定的数据，但是也会备份同一块中的其他数据点。
- 还原数据时，您可能会看到指定时间段之外的数据。
- 如果备份文件中包含重复的数据点，则将再次写入这些点，从而覆盖所有现有数据。


---
# _internal 数据
https://docs.influxdata.com/platform/monitoring/influxdata-platform/tools/measurements-internal/#influxdb-internal-measurements-and-fields

`cq`: 连续查询
    - `queryFail`: 连续查询执行失败的总次数
    - `queryOk`: 连续查询执行成功的总次数

`database`: 数据库
    - `numMeasurements`: 指定数据库下的 measurement 数量，估算值，当数量过大时会有一点较小的错误。
    - `numSeries`: 指定数据库下的 series 基数，估算值。

`httpd`: HTTP 相关
    - `authFail`: 在身份验证开启的情况下验证失败的次数。
    - `clientError`: 4xx 客户端请求错误的次数。
    - `fluxQueryReq`: flux 查询请求数。
    - `fluxQueryReqDurationNs`: 执行 flux 查询请求的耗时，纳秒。
    - `pingReq`: `/ping` 接口响应次数。
    - `pointsWrittenDropped`: 存储引擎删除的 point 数。
    - `pointsWrittenFail`: /write 接口接受到但无法写入的 point 数。
    - `pointsWrittenOK`: /write 接口接受且成功写入的 point 数。
    - `promReadReq`: 对接 Prometheus /read 接口的请求数。
    - `promWriteReq`: 对接 Prometheus /write 接口的请求数。
    - `queryReq`: 查询请求数。
    - `queryReqDurationNs`: 总的查询请求耗时，纳秒。
    - `queryRespBytes`: 查询响应中返回的总字节数。
    - `recoveredPanics`: http 服务 panic 然后恢复的次数。
    - `req`: http 请求总数。
    - `reqActive`: 当前活动的请求数。
    - `reqDurationNs`: HTTP请求内部花费的持续时间。
    - `serverError`: 服务器错误的 http 响应数。
    - `statusReq`: /status 接口的请求数。
    - `writeReq`: /write 接口的请求数。
    - `writeReqActive`: 当前进行中的写入请求数。
    - `writeReqBytes`: /write 接口接收到的 line protocol 数据的总字节数。
    - `writeReqDurationNs`: /write 接口的内部耗时，纳秒。

`queryExecutor`: 查询执行器的使用情况
    - `queriesActive`: 当前正在处理的查询数。
    - `queriesExecuted`: 已执行的查询数(从开始算起)。
    - `queriesFinished`: 完成执行的查询数。
    - `queryDurationNs`: 执行查询的耗时。
    - `recoveredPanics`: 查询执行器异常恢复的次数。

`runtime`: 运行时消息
    - `Alloc`: 当前分配的 heap 字节数。
    - `Frees`: 释放 heap 的累计数。
    - `HeapAlloc`: 所有 heap 对象的大小，字节单位。
    - `HeapIdle`: 空闲的 heap 对象大小，字节。
    - `HeapInUse`: in-use 的字节数。
    - `HeapObjects`: 分配的 heap 对象数。
    - `HeapReleased`: 返回给操作系统的物理内存的字节数。


`shard`

`subscriber`

`tsm1_cache`

`tsm1_engine`

`tsm1_filestore`

`tsm1_wal`

`write`





