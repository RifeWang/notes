# 配置
https://docs.influxdata.com/influxdb/v1.7/administration/config/

# 验证及权限
https://docs.influxdata.com/influxdb/v1.7/administration/authentication_and_authorization/

# 日志
service 日志默认输出为 stderr ，文件为 `/var/log/influxdb/<node-type>.log`。
可配置文件 `/etc/default/influxdb`: `STDERR=/dev/null` 修改输入文件位置。

# 恢复

# 重建 TSI index
`/data/<dbName>/_series`, 
`/data/<dbName/<rpName>/<shardID>/index`

# 安全管理

# 服务监控
https://docs.influxdata.com/influxdb/v1.7/administration/server_monitoring/

# 订阅管理
https://docs.influxdata.com/influxdb/v1.7/administration/subscription-management/

以 line protocal 复制写入的数据，`write-concurrency` 控制复制写入的并发数（goroutine），设置的越高处理高写入载荷的能力越强，但是由于写入进程和传输层的纳秒级差异会导致写入的顺序不一致，设置为 1 可以保证顺序一致。

### 创建订阅者：`CREATE SUBSCRIPTION`

-- Pattern:
`CREATE SUBSCRIPTION "<subscription_name>" ON "<db_name>"."<retention_policy>" DESTINATIONS <ALL|ANY> "<subscription_endpoint_host>"`

-- Examples:
-- Create a SUBSCRIPTION on database 'mydb' and retention policy 'autogen' that sends data to 'example.com:9090' via HTTP.
`CREATE SUBSCRIPTION "sub0" ON "mydb"."autogen" DESTINATIONS ALL 'http://example.com:9090'`

-- Create a SUBSCRIPTION on database 'mydb' and retention policy 'autogen' that round-robins the data to 'h1.example.com:9090' and 'h2.example.com:9090' via UDP.
`CREATE SUBSCRIPTION "sub0" ON "mydb"."autogen" DESTINATIONS ANY 'udp://h1.example.com:9090', 'udp://h2.example.com:9090'`


### 查询订阅者：`SHOW SUBSCRIPTIONS`


### 删除订阅者：`DROP SUBSCRIPTION`:

-- Pattern:
`DROP SUBSCRIPTION "<subscription_name>" ON "<db_name>"."<retention_policy>"`

-- Example:
`DROP SUBSCRIPTION "sub0" ON "mydb"."autogen"`

InfluxDB 假设所有的订阅端都应该收到数据，直到订阅者被手动移除，只会作用于添加了订阅者之后的新数据，而且失败了不会重试。