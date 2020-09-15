# 连续查询
处理实时数据，自动定期的执行并存储查询结果。


基础语法：
```
CREATE CONTINUOUS QUERY <cq_name> ON <database_name>
BEGIN
  <cq_query>
END
```

`cq_query` :
```
SELECT <function[s]> 
  INTO <destination_measurement> 
  FROM <measurement> 
  [WHERE <stuff>] 
  GROUP BY time(<interval>)[,<tag_key[s]>]
```

必须包含有 `function` ，一个 `INTO` 子句，和一个 `GROUP BY time()` 子句。
cq_query 不需要在 WHERE 子句指定时间范围，即使指定了也会被忽略，InfluxDB 会自动生成时间范围在执行 CQ 时。

什么时候执行？
执行的数据范围？
写入哪些数据？

### schedule and coverage
连续查询作用于实时数据。他们使用本地服务器的时间戳，`GROUP BY time()`间隔和InfluxDB数据库的预设时间边界来确定执行时间以及查询所涵盖的时间范围。

CQ的执行间隔与 cq_query 的 `GROUP BY time()` 间隔相同，并且它们在InfluxDB数据库的预设时间范围的开始处运行。如果 `GROUP BY time()` 间隔为一小时，则CQ在每小时开始时执行。

执行CQ时，它将对 `now()` 和 `now()` 减去 `GROUP BY time()` 间隔之间的时间范围运行单个查询。如果 `GROUP BY time()` 间隔为一小时且当前时间为17:00，则查询的时间范围为16:00至16：59.999999999。

```
If the group by is 10m and you create the cq at 9:11 for example the first execution will be at 09:20 , and the next at 09:30 , 40 ,50 ,00,10, 20. … ( so in that case it works after 9 minutes :slight_smile: )
00 , 10 , 20 , 30 , 40 , 50 are also called InfluxDB’s preset time boundaries
```

自动下采样：
```
CREATE CONTINUOUS QUERY "cq_basic" ON "transportation"
BEGIN
  SELECT mean("passengers") INTO "average_passengers" FROM "bus_data" GROUP BY time(1h)
END
```

指定非默认的 RP 保留采样数据 `INTO <database_name>.<retention_policy_name>.<measurement_name>`：
```
CREATE CONTINUOUS QUERY "cq_basic_rp" ON "transportation"
BEGIN
  SELECT mean("passengers") INTO "transportation"."three_weeks"."average_passengers" FROM "bus_data" GROUP BY time(1h)
END
```


使用通配符 `*` 和反向引用 `:MEASUREMENT`, 采样所有 measurements ：
```
CREATE CONTINUOUS QUERY "cq_basic_br" ON "transportation"
BEGIN
  SELECT mean(*) INTO "downsampled_transportation"."autogen".:MEASUREMENT FROM /.*/ GROUP BY time(30m),*
END
```

#### 时间边界
在`GROUP BY time（）`子句中使用偏移间隔来更改CQ的默认执行时间和预设的时间范围。
```
CREATE CONTINUOUS QUERY "cq_basic_offset" ON "transportation"
BEGIN
  SELECT mean("passengers") INTO "average_passengers" FROM "bus_data" GROUP BY time(1h,15m)
END
```
15分钟的偏移时间间隔会强制CQ在默认执行时间之后15分钟执行； cq_basic_offset在8:15而不是8:00执行。
15分钟的偏移间隔使CQ的WHERE子句中生成的预设时间范围向前移动； cq_basic_offset在7:15和8：14.999999999之间而不是7:00和7：59.999999999之间进行查询。

没有数据不会生成统计结果。

### 高级语句
#### resample every : 执行的间隔
CQ的数据时间范围与`RESAMPLE`子句中的`EVERY`间隔相同，它们在InfluxDB预设时间边界的开始处运行。如果 `EVERY` 间隔为两个小时，则InfluxDB每隔一小时执行一次CQ。

执行CQ时，它将对now（）和now（）减去`RESAMPLE`子句中的`FOR`间隔之间的时间范围运行单个查询。如果FOR间隔是两个小时，当前时间是17:00，则查询的时间范围是15:00和16：59.999999999。

```
CREATE CONTINUOUS QUERY "cq_advanced_every" ON "transportation"
RESAMPLE EVERY 30m
BEGIN
  SELECT mean("passengers") INTO "average_passengers" FROM "bus_data" GROUP BY time(1h)
END
```
时间范围是 1小时，每 30 分钟统计一次，但是统计结果的 time 仍然是 1小时划分的，也就是会说产生结果的覆盖。

#### resample for : 数据的时间范围
```
CREATE CONTINUOUS QUERY "cq_advanced_for" ON "transportation"
RESAMPLE FOR 1h
BEGIN
  SELECT mean("passengers") INTO "average_passengers" FROM "bus_data" GROUP BY time(30m)
END
```
每隔30分钟，运行一次查询，查询的范围是now（）和now（）减去FOR间隔，即now（）和now（）之前一小时之间的时间范围。
每次执行，写入两个点。

#### every for 
```
CREATE CONTINUOUS QUERY "cq_advanced_every_for" ON "transportation"
RESAMPLE EVERY 1h FOR 90m
BEGIN
  SELECT mean("passengers") INTO "average_passengers" FROM "bus_data" GROUP BY time(30m)
END
```

#### for fill()
必须要有一个点落在 for 时间范围内，fill 才会填充数据。
```
CREATE CONTINUOUS QUERY "cq_advanced_for_fill" ON "transportation"
RESAMPLE FOR 2h
BEGIN
  SELECT mean("passengers") INTO "average_passengers" FROM "bus_data" GROUP BY time(1h) fill(1000)
END
```

1、EVERY 比 GROUP BY time() 大：CQ以与EVERY间隔相同的间隔执行，并运行一个查询，该查询覆盖now（）和now（）减去EVERY间隔

2、If the FOR interval is less than the execution interval：报错


CQ 无法修改，只能创建和删除。