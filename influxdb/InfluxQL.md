# InfluxQL 语法

时间字面量度量：

Units |	Meaning
-- | --
ns	    |   nanoseconds (1 billionth of a second)
u or µ	|   microseconds (1 millionth of a second)
ms	    |   milliseconds (1 thousandth of a second)
s	    |   second
m	    |   minute
h	    |   hour
d	    |   day
w	    |   week

## 基本的 SELECT 语句
一个完整的 select 语句包含有：
```
SELECT_子句
    INTO_子句
    FROM_子句
    WHERE_子句
    GROUP_BY_子句
    ORDER_BY_子句
    LIMIT_子句
    OFFSET_子句
    SLIMIT_子句
    SOFFSET_子句
```

`SELECT <field_key>[,<field_key>,<tag_key>] FROM <measurement_name>[,<measurement_name>]`

select 语句必须要有 select 子句和 from 子句。

### select 子句
`SELECT *` : 返回所有 fields 和 tags 。
`SELECT "<field_key>"` : 返回指定的 field 。
`SELECT "<field_key>","<field_key>"` : 返回多个 field 。
`SELECT "<field_key>","<tag_key>"` : 返回指定的 field 和 tag ，如果有 tag 则必须至少有一个 field 。
`SELECT "<field_key>"::field,"<tag_key>"::tag` : 通过 `::[field | tag]` 标明是 field 还是 tag ，作用于 field 和 tag 同名的时候。

### from 子句
`FROM <measurement_name>` : 当前数据库、默认 RP 下的 measurement 。
`FROM <measurement_name>,<measurement_name>` : 指定多个 measurement 。
`FROM <database_name>.<retention_policy_name>.<measurement_name>` : 完整声明的 measurement 。
`FROM <database_name>..<measurement_name>` : 指定数据库，默认 RP 下的 measurement 。

### 引号：建议对于标识符或者关键字总是使用双引号。

示例：
`SELECT * FROM "table"`
`SELECT "field1","tag1","field2" FROM "table"`
`SELECT "field1"::field,"tag1"::tag,"field2"::field FROM "table"`
`SELECT *::field FROM "table"`
`SELECT ("field1" * 2) + 4 from "table"`
`SELECT * FROM "table1","table2"`
`SELECT * FROM "db"."autogen"."table"`
`SELECT * FROM "db".."table"`

### select 语句的常见问题
select 子句必须要有至少一个 field key 才能返回数据，如果只有一个或多个 tag key 则相应结果为空。

---------
## WHERE 子句
过滤数据，基于 fields、tags、timestamps 。

`SELECT_clause FROM_clause WHERE <conditional_expression> [(AND|OR) <conditional_expression> [...]]`
where 子句支持 fiels、tags、timestamps 的条件表达式。

注意：where 子句不支持使用 OR 指定多个时间范围，比如：`WHERE time = '2016-07-31T20:07:00Z' OR time = '2016-07-31T23:07:17Z'` , 响应结果为空。

#### field
`field_key <operator> ['string' | boolean | float | integer]`
WHERE子句支持对字符串，布尔值，浮点数和整数字段值进行比较。
WHERE子句中，string 类型的 field value 需要使用单引号，不加引号或者双引号将不返回任何数据，并且在大多数情况下，将不返回错误。

比较操作符：
Operator |	Meaning
-- | --
=	| 等于
<>	| 不等于
!=	| 不等于
>	| 大于
>=	| 大于等于
<	| 小于
<=	| 小于等于

#### tags
`tag_key <operator> ['tag_value']`

tag value 需要使用单引号，不加引号或者双引号将不返回任何数据，并且在大多数情况下，将不返回错误。

Operator |	Meaning
-- | --
=	| 等于
<>	| 不等于
!=	| 不等于

#### timestamps
对于大多数 select 语句，默认的时间范围是 `1677-09-21T00:12:43.145224194Z` (-9223372036854775806) 到 `2262-04-11T23:47:16.854775806Z` (9223372036854775806) UTC 。
对于使用了 `GROUP BY time()` 子句的 select 语句，默认的时间范围是 `1677-09-21T00:12:43.145224194Z` UTC 到 `now()` 。

示例：
`SELECT * FROM "table" WHERE "field1" > 8`
`SELECT * FROM "table" WHERE "field2" = 'string_content'`
`SELECT * FROM "table" WHERE "field1" + 2 > 11.9`
`SELECT "field1" FROM "table" WHERE "tag1" = 'string_content'`
`SELECT "field1" FROM "table" WHERE "tag1" <> 'string_content' AND (field1 < -0.59 OR field1 > 9.95)`
`SELECT * FROM "table" WHERE time > now() - 7d`

#### where 子句常见问题
返回空数据：string 类型的 field value 需要使用单引号，或者 tag value 需要使用单引号。
在大多数情况下，将不返回错误，是指的是语法解析的情况，比如：
`select * from "table" where "field" = string content`
这里的 string content 因为空格的原因肯定是解析出错的。

但是
`select * from "table" where "field" = string_content`
`select * from "table" where "field" = "string content"`
将不会返回数据。

--------
## 基础 GROUP BY 子句
基于 tags 集或者时间范围将数据分组。

### GROUP BY tags
`GROUP BY <tag>` ，每个 tag 对应一组数据结果。

`SELECT_clause FROM_clause [WHERE_clause] GROUP BY [* | <tag_key>[,<tag_key]]`

`GROUP BY *` : 对所有 tags 分组。
`GROUP BY <tag_key>` : 对指定 tag 分组。
`GROUP BY <tag_key>,<tag_key>` : 指定多个 tag 分组。

示例：
`SELECT MEAN("field1") FROM "table" GROUP BY "tag1"`
`SELECT MEAN("field1") FROM "table" GROUP BY tag1,tag2`
`SELECT MEAN("field1") FROM "table" GROUP BY *`

### GROUP BY time intervals
`GROUP BY time()` 时间范围

`SELECT <function>(<field_key>) FROM_clause WHERE <time_range> GROUP BY time(<time_interval>),[tag_key] [fill(<fill_option>)]`

基本的 `GROUP BY time()` 查询必须要在 select 子句中有一个 function ，并且 where 子句中有时间范围。

`GROUP BY time(<time_interval>)` 中的 `time_interval` 是时间字面量，比如 5m（五分钟），将 where 子句的时间范围分组。

`fill(<fill_option>)` 是可选的，无数据时填充默认值。

示例：
`SELECT COUNT("field1") FROM "table" WHERE "tag1"='tag value' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m)`

`SELECT COUNT("field1") FROM "table" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m),"tag1"`

### 常见问题
查询结果出现令人意外的 timestamps 和 values。
InfluxDB 依赖于 `GROUP BY time()` 的时间范围，以及系统的预设时间边界来确定每个时间间隔中包含的原始数据以及查询返回的 timestamp 。

```
> SELECT "water_level" FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:18:00Z'
name: h2o_feet
--------------
time                   water_level
2015-08-18T00:00:00Z   8.12
2015-08-18T00:06:00Z   8.005
2015-08-18T00:12:00Z   7.887
2015-08-18T00:18:00Z   7.762
```

意外的结果：
```
> SELECT COUNT("water_level") FROM "h2o_feet" WHERE "location"='coyote_creek' AND time >= '2015-08-18T00:06:00Z' AND time < '2015-08-18T00:18:00Z' GROUP BY time(12m)

name: h2o_feet
time                   count
----                   -----
2015-08-18T00:00:00Z   1        <----- Note that this timestamp occurs before the start of the query's time range
2015-08-18T00:12:00Z   1
```

说明：
InfluxDB 使用预设的时间边界作为 `GROUP BY` 间隔，该间隔与 where 子句中的任何时间条件都无关。

Time Interval Number	| Preset Time Boundary  |	GROUP BY time() Interval    |	Points Included	|Returned Timestamp
-- | --| --| -- | --
1	|   time >= 2015-08-18T00:00:00Z AND time < 2015-08-18T00:12:00Z	|    time >= 2015-08-18T00:06:00Z AND time < 2015-08-18T00:12:00Z    |	8.005   |   2015-08-18T00:00:00Z
2	|   time >= 2015-08-12T00:12:00Z AND time < 2015-08-18T00:24:00Z    |	time >= 2015-08-12T00:12:00Z AND time < 2015-08-18T00:18:00Z    |	7.887   |	2015-08-18T00:12:00Z

高级 `GROUP BY time()` 语法可以更改预设时间范围的开始时间。

## 高级 GROUP BY time() 子句
`SELECT <function>(<field_key>) FROM_clause WHERE <time_range> GROUP BY time(<time_interval>,<offset_interval>),[tag_key] [fill(<fill_option>)]`

高级的 `GROUP BY time()` 查询必须要在 select 子句中有一个 function ，并且 where 子句中有时间范围。

`time(time_interval,offset_interval)` 中的 offset_interval 确定时间边界的平移


...... 此处省略了一大堆


## GROUP BY 时间间隔和 fill()
`SELECT <function>(<field_key>) FROM_clause WHERE <time_range> GROUP BY time(time_interval,[<offset_interval])[,tag_key] [fill(<fill_option>)]`

默认情况下，`GROUP BY time()` 时间间隔内没有数据时会返回 `null` ，fill 用于填充数据。

`fill_option`:
- `linear` : 线性数据
- `none` : 不返回 timestamp 和数据
- `null` : 返回 timestamp ，数据为空 
- `previous` : 填充上一个数据

### fill() 常见问题
- 当前，如果没有数据落在查询时间范围内，则会忽略 fill() ，返回结果为空。
- `fill(previous)` 如果上一条结果在查询时间范围之外，则填充为 null 。
- `fill(linear)` 如果上一条结果在查询时间范围之外，则填充为 null 。

---------------------------
## INTO 子句
将查询结果写入 measurement 。

`SELECT_clause INTO <measurement_name> FROM_clause [WHERE_clause] [GROUP_BY_clause]`

- `INTO <measurement_name>`
- `INTO <database_name>.<retention_policy_name>.<measurement_name>`
- `INTO <database_name>..<measurement_name>`
- `INTO <database_name>.<retention_policy_name>.:MEASUREMENT FROM /<regular_expression>/`

示例：
`SELECT * INTO "copy_NOAA_water_database"."autogen".:MEASUREMENT FROM "NOAA_water_database"."autogen"./.*/ GROUP BY *`
实际上拷贝了所有数据，这样做可以重命名数据库。`GROUP BY *` 维持了所有 tag 不变，否则写入的数据会把之前的 tag 作为 field 。

对于大批量的数据建议划分 measurement 和时间范围分批进行。


`SELECT "water_level" INTO "h2o_feet_copy_1" FROM "h2o_feet" WHERE "location" = 'coyote_creek'`
`SELECT "water_level" INTO "where_else"."autogen"."h2o_feet_copy_2" FROM "h2o_feet" WHERE "location" = 'coyote_creek'`
`SELECT MEAN("water_level") INTO "all_my_averages" FROM "h2o_feet" WHERE "location" = 'coyote_creek' AND time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m)`
`SELECT MEAN(*) INTO "where_else"."autogen".:MEASUREMENT FROM /.*/ WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:06:00Z' GROUP BY time(12m)`

### INTO 子句常见问题
- 缺少数据：如果 INTO 查询在 SELECT 子句中包含 tag ，则该查询会将当前 measurement 中的 tag 转换为目标 measurement 的 field 。这可能会导致数据被覆盖。为了保持 tag 一定要使用 `GROUP BY <tag>` 或者 `GROUP BY *` .
- 自动执行，连续查询相关。


## ORDER BY time DESC
默认情况下按照时间升序，`ORDER BY time DESC` 指定时间降序


## LIMIT 子句
`LIMIT <N>` 返回 N 个 point 数据。

## SLIMIT 子句
`SLIMIT <N>` 返回 N 个 series 里的所有 point 数据。

### LIMIT and SLIMIT
`LIMIT <N1> SLIMIT <N2>` 


## OFFSET 子句
`OFFSET <N>`

## SOFFSET 子句
`SOFFSET <N>`


## Time Zone 子句
`tz()` 将 UTC 时间转换为特定时区的时间。
`SELECT_clause [INTO_clause] FROM_clause [WHERE_clause] [GROUP_BY_clause] [ORDER_BY_clause] [LIMIT_clause] [OFFSET_clause] [SLIMIT_clause] [SOFFSET_clause] tz('<time_zone>')`



## Time 语法
绝对时间，相对时间

`SELECT "water_level" FROM "h2o_feet" WHERE time > '2015-09-18T21:24:00Z' + 6m`
`SELECT "level description" FROM "h2o_feet" WHERE time > '2015-09-18T21:18:00Z' AND time < now() + 1000d`

### Time 语法常见问题
- 不支持 OR 选择多个时间范围。
- 大多数 SELECT 语句的默认时间范围是 `1677-09-21 00:12:43.145224194` 和 `2262-04-11T23:47:16.854775806Z` ，对于带有 GROUP BY time() 的默认时间范围是 `1677-09-21 00:12:43.145224194` 到 `now()` 。为了查询 now() 之后的数据，必须指定时间范围上限。


## Regular expressions

正则表达式可用于：
- SELECT 子句中的 field keys 和 tag keys
- FROM 子句中的 measurements
- WHERE 子句中的 tag values 和 string field values
- GROUP BY 子句中的 tag keys

正则表达式当前必须都是字符串，以 `/` 表示，使用的是 golang 的正则表达式语法。

不支持 `/<regular_expression>/::[field | tag]` 区分 field 和 tag 。

`=~` 匹配，`!~` 不匹配。

示例：
`SELECT /l/ FROM "h2o_feet" LIMIT 1`
`SELECT MEAN("degrees") FROM /temperature/`
`SELECT MEAN(water_level) FROM "h2o_feet" WHERE "location" =~ /[m]/ AND "water_level" > 3`
`SELECT * FROM "h2o_feet" WHERE "location" !~ /./`
`SELECT MEAN("water_level") FROM "h2o_feet" WHERE "location" =~ /./`
`SELECT MEAN("water_level") FROM "h2o_feet" WHERE "location" = 'santa_monica' AND "level description" =~ /between/`
`SELECT FIRST("index") FROM "h2o_quality" GROUP BY /l/`


## Data types and cast operations 数据类型和转换操作
`::` 转换类型

`SELECT_clause <field_key>::<type> FROM_clause` 针对 field 。

当前只支持 float 和 integer 之间的相互转换。
`SELECT "water_level"::float FROM "h2o_feet" LIMIT 4`
`SELECT "water_level"::integer FROM "h2o_feet" LIMIT 4`


## Merge 合并行为
自动合并 series ：
`SELECT MEAN("water_level") FROM "h2o_feet"`

如果你想要区分 series ，你可以
- 使用 where 指定 tag , `SELECT MEAN("water_level") FROM "h2o_feet" WHERE "location" = 'coyote_creek'`
- 或者，使用 GROUP BY ， `SELECT MEAN("water_level") FROM "h2o_feet" GROUP BY "location"`

## 多语句
多条查询语句使用 `;` 分隔。


## Subqueries 子查询
- `SELECT_clause FROM ( SELECT_statement ) [...]`
- `SELECT_clause FROM ( SELECT_clause FROM ( SELECT_statement ) [...] ) [...]`

示例：
- `SELECT SUM("max") FROM (SELECT MAX("water_level") FROM "h2o_feet" GROUP BY "location")`
- `SELECT MAX("water_level") FROM "h2o_feet" GROUP BY "location"`
- `SELECT MEAN("difference") FROM (SELECT "cats" - "dogs" AS "difference" FROM "pet_daycare")`
- `SELECT "cats" - "dogs" AS "difference" FROM "pet_daycare"`
- `SELECT "all_the_means" FROM (SELECT MEAN("water_level") AS "all_the_means" FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m) ) WHERE "all_the_means" > 5`
- `SELECT MEAN("water_level") AS "all_the_means" FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m)`
- `SELECT SUM("water_level_derivative") AS "sum_derivative" FROM (SELECT DERIVATIVE(MEAN("water_level")) AS "water_level_derivative" FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m),"location") GROUP BY "location"`
- `SELECT DERIVATIVE(MEAN("water_level")) AS "water_level_derivative" FROM "h2o_feet" WHERE time >= '2015-08-18T00:00:00Z' AND time <= '2015-08-18T00:30:00Z' GROUP BY time(12m),"location"`

### 子查询常见问题
支持子查询的嵌套，但是不支持单个子查询中有多个查询语句。