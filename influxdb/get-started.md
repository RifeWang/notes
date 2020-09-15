确保已经通过 `service influxdb start` 或者 `influxd` 开启了服务。

#### service 配置文件： 
service: `/etc/init.d/influxdb`,  /var/log/influxdb/influxd.log
systemd: `/usr/lib/influxdb/scripts/influxdb.service`
以上两个文件都要修改，一定要注意用户和用户组权限。

### ----
`influx -precision rfc3339`: 开启命令行，时间精度使用 RFC3339 标准 YYYY-MM-DDTHH:MM:SS.nnnnnnnnnZ

`CREATE DATABASE <db-name>`: 创建数据库

`SHOW DATABASES`: 显示数据库，_internal 是内部使用的库

`USE <db-name>`: 使用某个具体的数据库

`measurement` 相关于一个表，主键就是时间，`tags` 和 `fields` 就是表中的列，`tags` 会被索引，`fields` 不会。InfluxDB 可以有几百万个 `measurements`，你不需要预先定义 schema ，空值也不会被存储。

`Points` 由以下部分组成（注意逗号和空格）：
`<measurement>[,<tag-key>=<tag-value>...] <field-key>=<field-value>[,<field2-key>=<field2-value>...] [unix-nano-timestamp]`

插入数据：
`INSERT <point>`: `INSERT cpu,host=serverA,region=us_west value=0.64`

查询数据：
- `SELECT "host", "region", "value" FROM "cpu"`
- `SELECT * FROM /.*/ LIMIT 1`
- `SELECT * FROM "cpu_load_short"`
- `SELECT * FROM "cpu_load_short" WHERE "value" > 0.9`

暂不支持 count(*) 统计所有数据条数

查询必须有一个非 time field

查询结果 statement 、 series

字段类型

无法 count(time) , 无法 count(tag)