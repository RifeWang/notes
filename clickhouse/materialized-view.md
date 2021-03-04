# Materialized View  物化视图

- 物化视图实质上就是触发器，将源数据进行转换然后写入目标表。
- 物化视图解决的问题：
    - 预先聚合计算，加速查询响应。
    - 独立存储聚合计算结果（源数据删除后，也能得到之前的聚合结果）。
- 物化视图的两种风格：
    - 隐式建表，自动创建 `.inner.` 开头的目标表（新版没有 `.inner.` 前缀了）。
    - 显示建表，然后通过 `TO` 指定目标表。


创建物化视图的语法：
```SQL
CREATE MATERIALIZED VIEW [IF NOT EXISTS] [db.]table_name
[ON CLUSTER]
[TO [db.]name]
[ENGINE = engine]
[POPULATE]
AS SELECT ...
```


物化视图存储由相应的 `SELECT` 查询转换的数据。

- 当不使用 `TO [db.]name` 时，必须指定 `ENGINE` 表引擎。
- 当使用 `TO [db.]name` 时，不要使用 `POPULATE`。

物化视图的实现方式如下：当将数据插入 `SELECT` 中指定的表中时，此 `SELECT` 查询将转换部分插入的数据，并将结果插入视图中。

物化视图只针对新插入的数据进行处理，源表的数据发生任何改变（例如更新、删除等）都不会改变物化视图。

如果你在创建物化视图的时候声明了 `POPULATE` , 那么 `SELECT` 表的所有已存在数据都将被插入到视图中进行处理，如同 `CREATE TABLE ... AS SELECT ...` 一样。否则，查询仅包含创建视图后插入表中的数据。官方不建议使用 `POPULATE` ，因为在视图创建期间插入表中的数据不会被插入到视图中。

`SELECT` 查询可以包含 `DISTINCT`，`GROUP BY`，`ORDER BY`，`LIMIT`... 。请注意，相应的转换是对插入数据的每个块独立进行的。例如，如果设置了 `GROUP BY`，则数据将在插入过程中聚合，但仅在单个插入的数据 packet 中聚合。数据将不会进一步汇总。例外是使用独立执行数据聚合的引擎（例如SummingMergeTree）时。

对物化视图进行 `ALTER` 存在局限性，因此可能不方便。如果物化视图使用了 `TO [db.]name`，你可以先 `DETACH` 视图，对目标表进行 `ALTER`，然后再 `ATTACH` 回来。

视图看起来与普通表相同，例如，你可以通过 `SHOW TABLES` 查询到它们。

删除视图也是使用 `DROP TABLE`。




## 示例

### basic summing

原始查询：

```SQL
SELECT pickup_location_id,
    sum(passenger_count) / pc_cnt AS pc_avg,
    count() AS pc_cnt
FROM tripdata GROUP BY pickup_location_id
ORDER BY pc_cnt DESC LIMIT 10
```

使用物化视图：

1. 创建物化视图：

```SQL
CREATE METERIALIZED VIEW tripdata_smt_mv
ENGINE = SummingMergeTree
PARTITION BY toYYYYMM(pickup_date)
ORDER BY (pickup_location_id, dropoff_location_id)
POPULATE
AS SELECT
    pickup_date,
    pickup_location_id,
    dropoff_location_id,
    sum(passenger_count) AS passenger_count_sum,
    count() AS trips_count
FROM tripdata
GROUP BY
    pickup_date, pickup_location_id, dropoff_location_id
```

2. 查询物化视图：

```SQL
SELECT pickup_location_id,
    sum(passenger_count_sum) / sum(trips_count) AS pc_avg,
    sum(trips_count) AS pc_cnt
FROM tripdata_smt_mv GROUP BY pickup_location_id
ORDER BY pc_cnt DESC LIMIT 10
```


### SQL aggregates 多种聚合

`source value -> partial aggregate -> merged aggregate` ：
原始数据 -> `-State` 部分聚合（中间状态） -> `-Merge` 合并聚合得到最终结果。

![-State](https://clickhouse.tech/docs/en/sql-reference/aggregate-functions/combinators/#agg-functions-combinator-state)

AggregateFunctions hold partially aggregated data :
    - Functions like avgState(X) to put data into view
    - Functions like avgMerge(X) to get results out
    - Functions like argMaxState(X, Y) link values across columns

------

原始查询：

```SQL
SELECT min(fare_amount), avg(fare_amount),
    max(fare_amount), sum(fare_amount), count()
FROM tripdata
WHERE (fare_amount > 0) AND (fare_amount < 500)
```

使用物化视图：

```SQL
CREATE METERIALIZED VIEW tripdata_agg_mv
ENGING = SummingMergeTree
PARTITION BY toYYYYMM(pickup_date)
ORDER BY (pickup_location_id, dropoff_location_id)
POPULATE AS SELECT
    pickup_date, pickup_location_id, dropoff_location_id,
    minState(fare_amount) as fare_amount_min,
    avgState(fare_amount) as fare_amount_avg,
    maxState(fare_amount) as fare_amount_max,
    sumState(fare_amount) as fare_amount_sum,
    countState() as fare_amount_count
FROM tripdata
WHERE (fare_amount > 0) AND (fare_amount < 500)
GROUP BY
    pickup_date, pickup_location_id, dropoff_location_id
```

查询物化视图：

```SQL
SELECT minMerge(fare_amount_min) AS fare_amount_min,
    avgMerge(fare_amount_min) AS fare_amount_avg,
    maxMerge(fare_amount_min) AS fare_amount_max,
    sumMerge(fare_amount_min) AS fare_amount_sum,
    countMerge(fare_amount_count) AS fare_amount_count
FROM tripdata_agg_mv
```




`PATITION BY tuple()` : 保持原始数据不分区


- 链式视图构建 pipeline
- 处理数据：通过 where 条件只接受未来的数据，使用 insert into ... select ... 导入过去的数据
- 修改表结构：
    - 隐式表：detach 视图，然后 alter table `.inner.` ，最后 attach 视图
    - 显式表：drop 视图，然后 alter 目标表，最后重新 create 物化视图


More use cases for views -- Any kind of transformation !
- Reading from specialized table engines like KafkaEngine
- Different sort orders
- Using views as indexes
- Down-sampling data for long-term storage
- Creating build pipelines