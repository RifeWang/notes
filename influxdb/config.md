## install
见官方文档，https://docs.influxdata.com/influxdb/v1.7/introduction/installation/

## config
默认配置文件 `/etc/influxdb/influxdb.conf` ，启动时通过 `-config` 指定配置文件，也可以通过环境变量 `INFLUXDB_CONFIG_PATH` 指定配置文件。`-config` 优先于环境变量。

配置文件的配置项：
- `reporting-disabled = false`: 每24小时自动上报数据使用情况给 usage.influxdata.com ，建议设置为 `true` 禁用它。
- `bind-address = "127.0.0.1:8088"`: RPC服务地址，用于备份和恢复

[meta] 对于集群使用的是 raft 协议
- `dir = "/var/lib/influxdb/meta"`: metadata/raft 数据存储位置
- `retention-autocreate = true`: 创建数据库时自动默认一个存储策略
- `logging-enabled = true`: 是否打印 meta service 日志

[data]
- `dir = "/var/lib/influxdb/data"`: 最终数据（TSM files）存储目录，最好根据自己的系统调整
- `wal-dir = "/var/lib/influxdb/wal"`: 预写文件（WAL files）存储目录
- `wal-fsync-delay = "0s"`: WAL文件刷盘间隔时间，0s 意味着直接直接写入，对于 non-SSD 非固态硬盘建议设置 0-100ms
- `index-version = "inmem"`: 用于 new shard 的分片索引类型，默认 inmem 使用内存索引，对于更高基数的数据集可以使用 `tsi1` 基于磁盘索引
- `trace-logging-enabled = false`: 提供更多追踪日志，注意只在调试时开启
- `query-log-enabled = true`: 查询日志
- `validate-keys = false`: 验证写入的数据，确保只有有效的 unicode 字符，会导致额外的开销

- `cache-max-memory-size = "1g"`: shard cache 的最大空间，大于该值会拒绝写入
- `cache-snapshot-memory-size = "25m"`: 抓取内存快照写入TSM文件的临界值（内存大小）
- `cache-snapshot-write-cold-duration = "10m"`: 刷写TSM文件的时间间隔，与上一个配置项配合使用

- `compact-full-write-cold-duration = "4h"`: 该时间间隔内，若分片仍未写入或删除数据，则将会压缩分片所有的TSM文件
- `max-concurrent-compactions = 0`: 一次可以运行的最大压缩并发数，0 意味着 `runtime.GOMAXPROCS(0)` CPU核数的 50%
- `compact-throughput = "48m"`: 压缩写入磁盘的速率限制，每秒48MB
- `compact-throughput-burst = "48m"`: 压缩写入磁盘的速率限制，与上一项配置配合使用

- `tsm-use-madv-willneed = false`: 默认关闭，某些内核可能会有问题

inmem index 相关：
- `max-series-per-database = 1000000`: 每个数据库允许写入的最大 series 数量，若超出会删除老的数据，设置为 0 可以禁用
- `max-values-per-tag = 100000`: 每个 tag 允许的最大 values 数量，超出触发删除，设置为 0 可以禁用

tsi1 index 相关：
- `max-index-log-file-size = "1m"`: WAL文件压缩成索引文件的阈值大小，会影响写入性能
- `series-id-set-cache-size = 100`: 使用缓存的大小，影响查询性能和 heap usage

[coordinator]
- `write-timeout = "10s"`: 写入超时
- `max-concurrent-queries = 0`: 最大并发查询数， 0 无限制
- `query-timeout = "0s"`: 查询超时, 0 无限制
- `log-queries-after = "0s"`: 慢查询记录， 0 不记录
- `max-select-point = 0`: SELECT 一次能处理的最大 points 数量， 0 无限制
- `max-select-series = 0`: SELECT 一次能处理的最大 series 数量，0 无限制
- `max-select-buckets = 0`: SELECT 可以处理的最大"GROUP BY time()"的时间周期数量，0 无限制

[retention]
- `enabled = true`: 数据保留策略，是否启用
- `check-interval = "30m"`: 检查保留策略的时间间隔

[shard-precreation]
- `enabled = true`: 分片预创建，是否启用
- `check-interval = "10m"`: 分片预创建的检查时间间隔
- `advance-period = "30m"`: 预创建分片的最大提前时间

[monitor] 自有监控系统。默认情况下，InfluxDB把这些数据写入_internal 数据库，如果这个库不存在则自动创建。 _internal 库默认的retention策略是7天，如果你想使用一个自己的retention策略，需要自己创建。
- `store-enabled = true`: 是否记录统计数据
- `store-database = "_internal"`: 存储统计数据的数据库
- `store-interval = "10s"`: 统计间隔时间

[http]
- `enabled = true`: 是否开启 http
- `flux-enabled = false`: 是否开启 flux 查询，默认关闭
- `flux-log-enabled = false`: 是否记录 flux 查询日志
- `bind-address = ":8086"`: http 端口地址
- `auth-enabled = false`: 是否开启权限验证
- `realm = "InfluxDB"`: basic auth 配置JWT realm 域
- `log-enabled = true`: 是否记录 http 请求日志
- `suppress-write-log = false`: 若 http 请求日志开启，是否禁用 http 的写入请求日志
- `access-log-path = ""`: http 请求日志的存储路径，未指定则将会输出为 stderr ，也就是会跟 InfluxDB 内部日志混合在一起
- `access-log-status-filters = []`: 请求日志过滤，比如 5xx 将不会记录对应状态码的请求
- `write-tracing = false`: 是否追踪写日志的详细信息

- `pprof-enabled = true`: 是否开启 pprof 端口，用于故障排除和监控，默认使用 localhost:6060
- `debug-pprof-enabled = false`: 调试 pprof 的启动异常

- `https-enabled = false`: 是否开启 HTTPS
- `https-certificate = "/etc/ssl/influxdb.pem"`: HTTPS 证书
- `https-private-key = ""`: HTTPS 私钥

- `shared-secret = ""`: JWT 验证的 secret
- `max-row-limit = 0`: 查询返回最大行数，0 无限制
- `max-connection-limit = 0`: http 连接的最大数量，0 无限制
- `unix-socket-enabled = false`: http 服务是否使用 unix domain socket
- `bind-socket = "/var/run/influxdb.sock"`: unix domain socket 地址
- `max-body-size = 25000000`: request body 最大 bytes
- `max-concurrent-write-limit = 0`: 最大并发写入量，0 无限制
- `max-enqueued-write-limit = 0`: 最大写入队列，0 无限制
- `enqueued-write-timeout = 0`: 写入队列等待的超时时间

[logging]
- `format = "auto"`: 日志格式
- `level = "info"`: error, warn, info, debug
- `suppress-logo = false`: 禁用 logo

[subscriber] 可以通过订阅备份所有接收到的数据
- `enabled = true`: 是否开启订阅服务
- `http-timeout = "30s"`: http 写入订阅者的超时时间
- `insecure-skip-verify = false`: 是否允许与订阅者建立不安全的 HTTPS 连接
- `ca-certs = ""`: CA 证书文件路径，默认使用系统证书
- `write-concurrency = 40`: 处理写入 channel 的 goroutines 数量
- `write-buffer-size = 1000`: 写入 channel 中的 in-flight buffered 大小

[graphite]
- `enabled = false`:
- `database = "graphite"`:
- `retention-policy = ""`:
- `bind-address = ":2003"`:
- `protocol = "tcp"`
- `consistency-level = "one"`:

- `batch-size = 5000`:
- `batch-pending = 10`:
- `batch-timeout = "1s"`:
- `udp-read-buffer = 0`:
- `separator = "."`:
- `tags = ["region=us-east", "zone=1c"]`:
- `templates = ["*.app env.service.resource.measurement", "server.*"]`

[collectd]
- `enabled = false`
- `bind-address = ":25826"`
- `database = "collectd"`
- `retention-policy = ""`
- `typesdb = "/usr/local/share/collectd"`
- `security-level = "none"`
- `auth-file = "/etc/collectd/auth_file"`
- `batch-size = 5000`
- `batch-pending = 10`
- `batch-timeout = "10s"`
- `read-buffer = 0`
- `parse-multivalue-plugin = "split"`

[opentsdb]
- `enabled = false`
- `bind-address = ":4242"`
- `database = "opentsdb"`
- `retention-policy = ""`
- `consistency-level = "one"`
- `tls-enabled = false`
- `certificate= "/etc/ssl/influxdb.pem"`
- `log-point-errors = true`
- `batch-size = 1000`
- `batch-pending = 5`
- `batch-timeout = "1s"`

[udp]
- `enabled = false`
- `bind-address = ":8089"`
- `database = "udp"`
- `retention-policy = ""`

- `precision = ""`
- `batch-size = 5000`
- `batch-pending = 10`
- `batch-timeout = "1s"`
- `read-buffer = 0`

[continuous_queries]
- `enabled = true`
- `log-enabled = true`
- `query-stats-enabled = false`: CQ 执行的统计信息是否写入默认监控库
- `run-interval = "1s"`: 检查 CQ 是否需要运行的时间间隔

[tls]
- `ciphers = ["TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305", "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"]`
- `min-version = "tls1.2"`
- `max-version = "tls1.2"`