# MergeTree {#table_engines-mergetree}

Clickhouse 中最强大的表引擎当属 `MergeTree`（合并树）引擎及该系列（`*MergeTree`）中的其他引擎。

`MergeTree` 系列的引擎被设计用于插入极大量的数据到一张表当中。数据会被快速地写入到多个数据片段中，并在后台按照一定的规则合并这些片段。相比于在插入时不断修改（重写）已存储的数据，这种策略提升了写入性能。

主要特点:

- 存储的数据按主键排序。

    这使得您能够创建一个小型的稀疏索引来加快数据检索。

- 支持基于[分区键](custom-partitioning-key.md)的分区功能。

    在相同数据集和相同结果集的情况下，ClickHouse 对分区进行的操作会比普通操作更快。查询中指定了分区键时，ClickHouse 会自动截取指定分区数据。

- 支持数据副本。

    `ReplicatedMergeTree` 系列的表提供了数据副本功能。更多信息，请参阅[数据副本](replication.md)一节。

- 支持数据采样。

    您可以给表设置一个采样方法。

!!! note "注意"
    [合并（Merge）](../special/merge.md#merge)引擎并不属于 `*MergeTree` 系列。

## 建表 {#table_engine-mergetree-creating-a-table}

``` sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1] [TTL expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2] [TTL expr2],
    ...
    INDEX index_name1 expr1 TYPE type1(...) GRANULARITY value1,
    INDEX index_name2 expr2 TYPE type2(...) GRANULARITY value2
) ENGINE = MergeTree()
ORDER BY expr
[PARTITION BY expr]
[PRIMARY KEY expr]
[SAMPLE BY expr]
[TTL expr [DELETE|TO DISK 'xxx'|TO VOLUME 'xxx'], ...]
[SETTINGS name=value, ...]
```

若想了解上述语句中参数的说明，请参阅 [CREATE 语句的说明](../../../engines/table-engines/mergetree-family/mergetree.md) 。

<a name="mergetree-query-clauses"></a>

**子句**

- `ENGINE` - 引擎名和引擎参数，也就是 `ENGINE = MergeTree()`。`MergeTree` 引擎没有参数。

- `ORDER BY` — 排序键。

     可以是列的元组或任意的表达式。例如: `ORDER BY (CounterID, EventDate)` 。

     如果没有使用 `PRIMARY KEY` 显式指定主键，ClickHouse 会使用排序键作为主键。

     如果不需要排序，可以使用 `ORDER BY tuple()`。更多信息可参考[选择主键](https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/mergetree/#selecting-the-primary-key)。

- `PARTITION BY` — [分区键](custom-partitioning-key.md) ，可选项。

     要按月分区，可以使用表达式 `toYYYYMM(date_column)`，这里的 `date_column` 是一个 [Date](../../../engines/table-engines/mergetree-family/mergetree.md) 类型的列。分区名的格式会是 `"YYYYMM"` 。

- `PRIMARY KEY` - 可以在此[选择与排序键不同的主键](#choosing-a-primary-key-that-differs-from-the-sorting-key)，可选项。

    默认情况下主键跟排序键（由 `ORDER BY` 子句指定）相同。因此，一般不需要再专门指定 `PRIMARY KEY` 子句。

- `SAMPLE BY` - 用于抽样的表达式，可选项。

     使用抽样表达式时，主键中必须包含这个表达式。例如：`SAMPLE BY intHash32(UserID) ORDER BY (CounterID, EventDate, intHash32(UserID))` 。

- `TTL` - 指定每一行存储的持续时间，配置数据片段在硬盘和卷上的移动规则，可选项。

    表达式中必须存在至少一个 `Date` 或 `DateTime` 类型的列，比如：
    `TTL date + INTERVAl 1 DAY`

    规则的类型 `DELETE|TO DISK 'xxx'|TO VOLUME 'xxx'` 指定了当满足条件（到达指定时间）时所要执行的动作：移除过期的行，还是将数据片段（如果数据片段中的所有行都满足表达式的话）移动到指定的磁盘（`TO DISK 'xxx'`) 或卷（`TO VOLUME 'xxx'`）。默认的规则是移除（`DELETE`）。可以在列表中指定多个规则，但最多只能有一个 `DELETE` 的规则。

    更多信息请查看[表和列的 TTL](#table_engine-mergetree-ttl)

- `SETTINGS` — 控制 `MergeTree` 行为的额外参数，可选项：

  - `index_granularity` — 索引粒度行上限。索引中相邻的“标记”间的数据行数上限。默认值：8192 。详细请参考[数据存储](#mergetree-data-storage)。
  - `index_granularity_bytes` — 索引粒度大小上限，以字节为单位，默认值: 10Mb。若想仅按数据行数限制索引粒度, 请设置为 0（不建议）。
  - `min_index_granularity_bytes` - 索引粒度大小下限，默认值：1024b。该选项用于防止误操作，添加了一个非常低索引粒度的表。详细请参考[数据存储](#mergetree-data-storage)
  - `enable_mixed_granularity_parts` — 是否使用 `index_granularity_bytes` 控制索引粒度的大小。在 19.11 版本之前, 仅能使用 `index_granularity` 来限制索引粒度的大小。当从具有很大的行（数十乃至数百兆字节）的表中查询数据的时候，`index_granularity_bytes` 配置能够提升 ClickHouse 的性能。如果您的表里有很大的行，可以开启这项配置来提升 `SELECT` 查询的性能。
  - `use_minimalistic_part_header_in_zookeeper` — 控制 ZooKeeper 中数据片段头的存储方式。若 `use_minimalistic_part_header_in_zookeeper=1`，会减少在 ZooKeeper 中存储的数据量。更多信息请参考[服务配置参数]([Server Settings | ClickHouse Documentation](https://clickhouse.tech/docs/zh/operations/server-configuration-parameters/settings/))中的[设置描述](../../../operations/server-configuration-parameters/settings.md#server-settings-use_minimalistic_part_header_in_zookeeper) 一节。
  - `min_merge_bytes_to_use_direct_io` — 高于此数据量时，才使用直接 I/O 操作磁盘进行数据合并。合并数据片段时，ClickHouse 会计算要被合并的所有数据的总存储空间。如果大小超过了 `min_merge_bytes_to_use_direct_io` 设置的字节数，则 ClickHouse 将使用直接 I/O 接口（`O_DIRECT` 选项）对磁盘读写。如果设置 `min_merge_bytes_to_use_direct_io = 0`，则会禁用直接 I/O。默认值：`10 * 1024 * 1024 * 1024` 字节。
        <a name="mergetree_setting-merge_with_ttl_timeout"></a>
  - `merge_with_ttl_timeout` — TTL 合并频率的最小间隔时间，单位：秒。默认值: 86400（1 天）。
  - `write_final_mark` — 是否在数据片段尾部写入最终索引标记。默认值: 1。请不要关闭此选项。
  - `merge_max_block_size` — 合并时每一块的最大行数限制。默认值：8192
  - `storage_policy` — 存储策略。详细请参考[使用具有多个块的设备进行数据存储](#table_engine-mergetree-multiple-volumes).
  - `min_bytes_for_wide_part`、`min_rows_for_wide_part` — 当字节数/行数高于配置值时，使用 `Wide` 格式进行存储。您可以不设置、只设置一个，或全都设置。详细请参考[数据存储](#mergetree-data-storage)
  - `max_parts_in_total` - 所有分区中块数量的上限
  - `max_compress_block_size` - 在数据压缩写入表前，未压缩数据块的最大大小。您可以在全局设置中设置该值（参见[max_compress_block_size](https://clickhouse.tech/docs/zh/operations/settings/settings/#max-compress-block-size)）。建表时指定该值会覆盖全局设置。
  - `min_compress_block_size` - 在数据压缩写入表前，未压缩数据块的最小大小。您可以在全局设置中设置该值（参见[min_compress_block_size](https://clickhouse.tech/docs/zh/operations/settings/settings/#min-compress-block-size)）。建表时指定该值会覆盖全局设置。
  - `max_partitions_to_read` - 单次查询中可访问的分区数上限。您可以在全局设置中设置该值（参见[max_partitions_to_read](https://clickhouse.tech/docs/zh/operations/settings/settings/#max_partitions_to_read)）。

**示例配置**

``` sql
ENGINE MergeTree() PARTITION BY toYYYYMM(EventDate) ORDER BY (CounterID, EventDate, intHash32(UserID)) SAMPLE BY intHash32(UserID) SETTINGS index_granularity=8192
```

在这个例子中，我们设置了按月进行分区。

同时我们设置了一个按用户 ID 哈希的抽样表达式。使得您可以对该表中每个 `CounterID` 和 `EventDate` 的数据伪随机分布。如果您在查询时指定了 [SAMPLE](../../../engines/table-engines/mergetree-family/mergetree.md#select-sample-clause) 子句。ClickHouse 会返回对于用户子集的一个均匀的伪随机数据采样。

`index_granularity` 可以省略，因为默认值就是 8192。

<details markdown="1">
<summary>已弃用的建表方法</summary>

!!! attention "注意"
    不要在新项目中使用该方法，建议尽可能地将旧项目迁移到上述的新方法上。

    CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
    (
        name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
        name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
        ...
    ) ENGINE [=] MergeTree(date-column [, sampling_expression], (primary, key), index_granularity)

**MergeTree() 参数**

- `date-column` — [Date](../../../engines/table-engines/mergetree-family/mergetree.md) 类型列的名称。ClickHouse 会自动依据这个列按月创建分区。分区名格式为 `"YYYYMM"`。
- `sampling_expression` — 采样表达式。
- `(primary, key)` — 主键。类型为[元组()](../../../engines/table-engines/mergetree-family/mergetree.md)
- `index_granularity` — 索引粒度。即索引中相邻“标记”间的数据行数。一般设为 8192 即可。

**示例**

    MergeTree(EventDate, intHash32(UserID), (CounterID, EventDate, intHash32(UserID)), 8192)

上述示例配置的作用跟前面的例子一样。
</details>

## 数据存储 {#mergetree-data-storage}

表由按主键排序的数据片段（DATA PART）组成。

当数据被插入到表中时，会创建多个数据片段并按主键的字典序排序。例如，主键是 `(CounterID, Date)` 时，片段中的数据首先按 `CounterID` 排序，相同 `CounterID` 的数据再按 `Date` 排序。

不同分区的数据会被分成不同的片段，ClickHouse 会在后台合并数据片段以优化存储效率。不同分区的数据片段不会进行合并。该合并机制不保证相同主键的行都在同一个数据片段中。

数据片段可以以 `Wide` 或 `Compact` 格式存储。在 `Wide` 格式下，每一列都会在文件系统中存储为单独的文件，在 `Compact` 格式下所有列都存储在一个文件中。在插入数据量小但频率频繁时，使用 `Compact` 格式有利于提升性能。

数据存储格式由 `min_bytes_for_wide_part` 和 `min_rows_for_wide_part` 表引擎参数控制。如果数据片段中的字节数或行数少于相应的设置值，数据片段会以 `Compact` 格式存储，否则会以 `Wide` 格式存储。

每个数据片段被逻辑的分割成颗粒（granules）。颗粒是 ClickHouse 中进行数据查询时的最小不可分割数据集。ClickHouse 不会对行或值进行拆分，所以每个颗粒总是包含整数个行。每个颗粒的第一行会使用该行的主键值进行标记，ClickHouse 会为每个数据片段创建一个索引文件来存储这些标记。对于每列，无论它是否包含在主键当中，ClickHouse 都会存储类似的标记。这些标记让您可以在列文件中直接找到数据。

颗粒的大小由表引擎参数 `index_granularity` 和 `index_granularity_bytes` 控制。颗粒行的数量在 `[1, index_granularity]` 范围之中，这取决于行的大小。如果单行的大小超过了 `index_granularity_bytes` 设置的值，那么一个颗粒的大小会超过 `index_granularity_bytes`。在这种情况下，颗粒的大小等于该行的大小。

## 主键和索引在查询中的表现 {#primary-keys-and-indexes-in-queries}

我们以 `(CounterID, Date)` 以主键。排序好的索引的图示会是下面这样：

``` text
    全部数据  :     [-------------------------------------------------------------------------]
    CounterID:      [aaaaaaaaaaaaaaaaaabbbbcdeeeeeeeeeeeeefgggggggghhhhhhhhhiiiiiiiiikllllllll]
    Date:           [1111111222222233331233211111222222333211111112122222223111112223311122333]
    标记:            |      |      |      |      |      |      |      |      |      |      |
                    a,1    a,2    a,3    b,3    e,2    e,3    g,1    h,2    i,1    i,3    l,3
    标记号:          0      1      2      3      4      5      6      7      8      9      10
```

如果指定查询如下：

- `CounterID in ('a', 'h')`，服务器会读取标记号在 `[0, 3)` 和 `[6, 8)` 区间中的数据。
- `CounterID IN ('a', 'h') AND Date = 3`，服务器会读取标记号在 `[1, 3)` 和 `[7, 8)` 区间中的数据。
- `Date = 3`，服务器会读取标记号在 `[1, 10]` 区间中的数据。

上面例子可以看出使用索引通常会比全表扫描要高效。

稀疏索引会引起额外的数据读取。当读取主键单个区间范围的数据时，每个数据块中最多会多读 `index_granularity * 2` 行额外的数据。

稀疏索引使得您可以处理极大量的行，因为大多数情况下，这些索引常驻于内存。

ClickHouse 不要求主键唯一，所以您可以插入多条具有相同主键的行。

您可以在 `PRIMARY KEY` 与 `ORDER BY` 条件中使用 `Nullable` 类型的表达式，但强烈建议不要这么做。要启用这项功能，可以打开 [allow_nullable_key](https://clickhouse.tech/docs/zh/operations/settings/settings/#allow-nullable-key) 配置。在 `ORDER BY` 条件中有 `NULL` 值的情况下，会适用 [NULLS_LAST](https://clickhouse.tech/docs/zh/sql-reference/statements/select/order-by/#sorting-of-special-values) 规则。

### 主键的选择 {#selecting-the-primary-key}

主键中列的数量并没有明确的限制。依据数据结构，您可以自由增添主键列。这样可以：

- 改善索引的性能。

    如果当前主键是 `(a, b)` ，在下列情况下添加另一个 `c` 列会提升性能：

  - 查询会使用 `c` 列作为条件
  - 数据范围中（`index_granularity` 的数倍）包含大量相同的 `(a, b)`。换言之，加入另一主键列后，查询可以略过很长的数据范围。

- 改善数据压缩。

    ClickHouse 以主键排序片段数据，所以，数据的一致性越高，压缩越好。

- 在 [CollapsingMergeTree](collapsingmergetree.md#table_engine-collapsingmergetree) 和 [SummingMergeTree](summingmergetree.md) 引擎里进行数据合并时会提供额外的处理逻辑。

    在这种情况下，指定与主键不同的 *排序键* 也是有意义的。

长的主键会降低插入性能并增加内存消耗，但向主键添加列并不会影响 `SELECT` 查询的性能。

可以使用 `ORDER BY tuple()` 语法创建没有主键的表。在这种情况下 ClickHouse 根据数据插入的顺序存储。如果在使用 `INSERT ... SELECT` 时希望保持数据的排序，请设置 [max_insert_threads = 1](../../../operations/settings/settings.md#settings-max-insert-threads)。

要使用初始顺序进行数据查询，可以使用[单线程查询](../../../operations/settings/settings.md#settings-max_threads)

### 选择与排序键不同的主键 {#choosing-a-primary-key-that-differs-from-the-sorting-key}

Clickhouse 的主键和排序键不要求是相同的，此时排序键用于在数据片段中进行排序，主键用于在索引文件中进行标记的写入。这种情况下，主键表达式元组必须是排序键表达式元组的前缀（即主键为 `(a,b)`，排序键必须为 `(a,b,******)`）。

当使用 [SummingMergeTree](summingmergetree.md) 和 [AggregatingMergeTree](aggregatingmergetree.md) 引擎时，这个特性非常有用。通常，在使用这类引擎时，表里的列分两种：*维度*（dimensions）和 *度量*（measures）。典型的查询会使用 `GROUP BY` 对度量列进行聚合并通过维度列进行过滤。由于 SummingMergeTree 和 AggregatingMergeTree 会对排序键相同的行进行聚合，所以把所有的维度放进排序键是很自然的做法。但这将导致主键表达式中包含大量的列，并且主键需要伴随着新添加的维度不断的更新。

在这种情况下合理的做法是，只保留少量的列在主键当中用于提升扫描效率，将维度列添加到排序键中。

对排序键进行 [ALTER](../../../sql-reference/statements/alter.md) 是轻量级的操作，因为当一个新列同时被加入到表里和排序键里时，旧有的数据片段并不需要修改。由于旧排序键是新排序键的前缀，并且新添加的列中没有数据，因此不管从哪种排序键的角度上来看，修改表时的数据都是有序的。

### 索引和分区在查询中的应用 {#use-of-indexes-and-partitions-in-queries}

对于 `SELECT` 查询，ClickHouse 会分析是否可以使用索引。如果 `WHERE/PREWHERE` 子句含有下面这些表达式（作为 WHERE 条件的部分或全部）时，可以使用索引：进行相等/不相等的比较；对主键列或分区列（或基于这些列的表达式）进行的固定前缀 `IN`、`LIKE` 运算（如 `name like 'test%'`）或函数运算（仅支持部分函数），以及上述表达式的逻辑运算。

 <!-- It is too hard for me to translate this section as the original text completely. So I did it with my own understanding. If you have good idea, please help me. -->
<!-- It is hard for me to translate this section too, but I think change the sentence struct is helpful for understanding. So I change the phraseology-->

<!--I try to translate it in Chinese,don't worry. -->

因此，在索引键的多个区间上快速地执行查询变为可能。在接下来的示例中，不管是对单个标签、单个标签和日期范围、单个标签和单个日期还是多个标签和日期范围的查询，速度都会非常快。

先看一下引擎的配置

``` sql
    ENGINE MergeTree() PARTITION BY toYYYYMM(EventDate) ORDER BY (CounterID, EventDate) SETTINGS index_granularity=8192
```

此时，对于这些查询：

``` sql
SELECT count() FROM table WHERE EventDate = toDate(now()) AND CounterID = 34
SELECT count() FROM table WHERE EventDate = toDate(now()) AND (CounterID = 34 OR CounterID = 42)
SELECT count() FROM table WHERE ((EventDate >= toDate('2014-01-01') AND EventDate <= toDate('2014-01-31')) OR EventDate = toDate('2014-05-01')) AND CounterID IN (101500, 731962, 160656) AND (CounterID = 101500 OR EventDate != toDate('2014-05-01'))
```

ClickHouse 会依据主键索引剪裁掉不符合条件的数据，依据按月分区的分区键裁剪掉那些不在指定日期范围内的分区。

上述查询展示了，即使使用了复杂的表达式，也能利用到索引。由于从表读取数据是有序的，因此使用索引不会比全表扫描更慢。

但在下面这个例子中则不会使用索引。

``` sql
SELECT count() FROM table WHERE CounterID = 34 OR URL LIKE '%upyachka%'
```

要检查 ClickHouse 执行一个查询时能否使用索引，可设置 [force_index_by_date](../../../operations/settings/settings.md#settings-force_index_by_date) 和 [force_primary_key](../../../operations/settings/settings.md) 。

按月分区的分区列使得只会读取包含指定日期区间的数据块，此时，数据块会包含很多天（最多一个月）的数据。在块中，数据按主键排序，而主键的第一列可能不包含日期数据。因此，当仅指定日期而没有指定主键前缀作为查询条件时，需要读取更多的数据，而不是只读取指定日期的数据。

### 部分单调主键的使用

考虑这样的场景，比如一个月中的天数。它们在一个月的范围内形成一个[单调序列](https://zh.wikipedia.org/wiki/单调函数)，但如果扩展到更大的时间范围它们就不再单调了。这就是一个部分单调序列。如果用户使用部分单调的主键创建表，ClickHouse 同样会创建一个稀疏索引。当用户从这类表中查询数据时，ClickHouse 会对查询条件进行分析。如果用户希望获取在相同月份内两个索引标记之间的数据，由于 ClickHouse 可以计算出查询参数与索引标记之间的距离，可以在这种特殊情况下利用到索引。

如果查询参数范围内的主键不是单调序列，那么 ClickHouse 无法使用索引。在这种情况下，ClickHouse 会进行全表扫描。

ClickHouse 在任何部分单调序列主键的情况下都会使用这个逻辑。

### 数据跳过索引 {#data_skipping-indexes}

要声明这种索引，可以在 `CREATE` 语句的列部分里定义。

``` sql
INDEX index_name expr TYPE type(...) GRANULARITY granularity_value
```

`*MergeTree` 系列的表均支持数据跳过索引。

数据跳过索引将在数据块（数据块包含 `granularity_value` 个颗粒，颗粒的大小可以通过引擎参数 `index_granularity` 配置）上聚集指定的表达式。当进行 `SELECT` 查询时，可以使用这些聚集信息跳过大量不满足 `where` 条件的数据。

**示例**

``` sql
CREATE TABLE table_name
(
    u64 UInt64,
    i32 Int32,
    s String,
    ...
    INDEX a (u64 * i32, s) TYPE minmax GRANULARITY 3,
    INDEX b (u64 * length(s)) TYPE set(1000) GRANULARITY 4
) ENGINE = MergeTree()
...
```

上例中的索引能让 ClickHouse 执行下面这些查询时减少读取数据量。

``` sql
SELECT count() FROM table WHERE s < 'z'
SELECT count() FROM table WHERE u64 * i32 == 10 AND u64 * length(s) >= 1234
```

#### 可用的索引类型 {#available-types-of-indices}

- `minmax`
    存储指定表达式的极值（如果表达式是 `tuple`，则存储 `tuple` 中每个元素的极值），这些信息用于跳过数据块，类似主键。

- `set(max_rows)`
    存储指定表达式的不重复值（不超过 `max_rows` 个，`max_rows=0` 则表示“无限制”）。这些信息可用于检查数据块是否满足 `WHERE` 条件。

- `ngrambf_v1(n, size_of_bloom_filter_in_bytes, number_of_hash_functions, random_seed)`
    存储包含数据块中所有 n 元短语（ngram）的[布隆过滤器](https://zh.wikipedia.org/wiki/%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8) 。仅支持字符串字段。可用于优化 `equals`、`like` 和 `in` 表达式的性能。
  - `n` – 短语长度。
  - `size_of_bloom_filter_in_bytes` – 布隆过滤器大小，以字节为单位。（由于压缩性能较好，可以在这里指定比较大的值，如 256 或 512）。
  - `number_of_hash_functions` – 布隆过滤器中使用哈希函数的个数。
  - `random_seed` – 哈希函数的随机种子。

- `tokenbf_v1(size_of_bloom_filter_in_bytes, number_of_hash_functions, random_seed)`
    跟 `ngrambf_v1` 类似，但是存储的是 token 而不是 ngrams。Token 是由非字母数字的符号分割的序列。

- `bloom_filter(bloom_filter([false_positive])` – 为指定的列存储布隆过滤器

    可选参数 `false_positive` 用来指定从布隆过滤器收到错误响应的几率。取值范围是 (0,1)，默认值：0.025。

    支持的数据类型：`Int*`、`UInt*`、`Float*`、`Enum`、`Date`、`DateTime`、`String`、`FixedString`、`Array`、`LowCardinality`、`Nullable`。

    以下函数可以利用这种索引：[equals](../../../sql-reference/functions/comparison-functions.md)、[notEquals](../../../sql-reference/functions/comparison-functions.md)、[in](../../../sql-reference/functions/in-functions.md)、[notIn](../../../sql-reference/functions/in-functions.md)、[has](../../../sql-reference/functions/array-functions.md)

``` sql
INDEX sample_index (u64 * length(s)) TYPE minmax GRANULARITY 4
INDEX sample_index2 (u64 * length(str), i32 + f64 * 100, date, str) TYPE set(100) GRANULARITY 4
INDEX sample_index3 (lower(str), str) TYPE ngrambf_v1(3, 256, 2, 0) GRANULARITY 4
```

#### 函数支持 {#functions-support}

WHERE 子句中的条件可以包含对某列数据进行运算的函数表达式，如果列是索引的一部分，ClickHouse 会在执行函数时尝试使用索引。不同函数所支持的索引种类不一。

`set` 索引会对所有函数生效，其他索引对函数的生效情况见下表。

| 函数 (操作符) / 索引                                         | primary key | minmax | ngrambf_v1 | tokenbf_v1 | bloom_filter |
| ------------------------------------------------------------ | ----------- | ------ | ---------- | ---------- | ------------ |
| [equals (=, ==)](../../../sql-reference/functions/comparison-functions.md#function-equals) | ✔           | ✔      | ✔          | ✔          | ✔            |
| [notEquals(!=, \<\>)](../../../sql-reference/functions/comparison-functions.md#function-notequals) | ✔           | ✔      | ✔          | ✔          | ✔            |
| [like](../../../sql-reference/functions/string-search-functions.md#function-like) | ✔           | ✔      | ✔          | ✔          | ✔            |
| [notLike](../../../sql-reference/functions/string-search-functions.md#function-notlike) | ✔           | ✔      | ✗          | ✗          | ✗            |
| [startsWith](../../../sql-reference/functions/string-functions.md#startswith) | ✔           | ✔      | ✔          | ✔          | ✗            |
| [endsWith](../../../sql-reference/functions/string-functions.md#endswith) | ✗           | ✗      | ✔          | ✔          | ✗            |
| [multiSearchAny](../../../sql-reference/functions/string-search-functions.md#function-multisearchany) | ✗           | ✗      | ✔          | ✗          | ✗            |
| [in](../../../sql-reference/functions/in-functions.md#in-functions) | ✔           | ✔      | ✔          | ✔          | ✔            |
| [notIn](../../../sql-reference/functions/in-functions.md#in-functions) | ✔           | ✔      | ✔          | ✔          | ✔            |
| [less (\<)](../../../sql-reference/functions/comparison-functions.md#function-less) | ✔           | ✔      | ✗          | ✗          | ✗            |
| [greater (\>)](../../../sql-reference/functions/comparison-functions.md#function-greater) | ✔           | ✔      | ✗          | ✗          | ✗            |
| [lessOrEquals (\<=)](../../../sql-reference/functions/comparison-functions.md#function-lessorequals) | ✔           | ✔      | ✗          | ✗          | ✗            |
| [greaterOrEquals (\>=)](../../../sql-reference/functions/comparison-functions.md#function-greaterorequals) | ✔           | ✔      | ✗          | ✗          | ✗            |
| [empty](../../../sql-reference/functions/array-functions.md#function-empty) | ✔           | ✔      | ✗          | ✗          | ✗            |
| [notEmpty](../../../sql-reference/functions/array-functions.md#function-notempty) | ✔           | ✔      | ✗          | ✗          | ✗            |
| hasToken                                                     | ✗           | ✗      | ✗          | ✔          | ✗            |

常量参数小于 ngram 大小的函数不能使用 `ngrambf_v1` 进行查询优化。

!!! note "注意"

布隆过滤器可能会包含不符合条件的匹配，所以 `ngrambf_v1`, `tokenbf_v1` 和 `bloom_filter` 索引不能用于结果返回为假的函数，例如：

- 可优化的场景
  - `s LIKE '%test%'`
  - `NOT s NOT LIKE '%test%'`
  - `s = 1`
  - `NOT s != 1`
  - `startsWith(s, 'test')`

- 不可优化的场景
  - `NOT s LIKE '%test%'`
  - `s NOT LIKE '%test%'`
  - `NOT s = 1`
  - `s != 1`
  - `NOT startsWith(s, 'test')`

### 投影 {#projections}

投影类似于物化视图，但它在数据片段级别上定义。用于在查询中提供自动的一致性保证。

#### 投影查询 {#projection-query}

投影基于投影查询进行定义。它的语法如下：
A projection query is what defines a projection. It has the following grammar:

`SELECT <COLUMN LIST EXPR> [GROUP BY] [ORDER BY]`

默认情况下，将会从父表查询数据。

#### 存储 {#projection-storage}

投影会被存储到数据片段的目录。它类似于索引，但包含了存储匿名 MergeTree 表的子目录。该表基于定义的投影查询进行计算。若查询包含 GROUP BY 子句，匿名表将自动使用 AggregatedMergeTree 作为引擎，并将所有聚集函数转换为 AggregateFunction。若查询包含 ORDER BY 子句，将把排序列作为 MergeTree 表的主键表达式。在数据片段合并的过程中，包含投影的数据片段会借助存储的合并功能进行合并。父表的校验和会包含投影表。其它维护性工作与数据跳过索引类似。
Projections are stored inside the part directory. It's similar to an index but contains a subdirectory that stores an anonymous MergeTree table's part. The table is induced by the definition query of the projection. If there is a GROUP BY clause, the underlying storage engine becomes AggregatedMergeTree, and all aggregate functions are converted to AggregateFunction. If there is an ORDER BY clause, the MergeTree table will use it as its primary key expression. During the merge process, the projection part will be merged via its storage's merge routine. The checksum of the parent table's part will combine the projection's part. Other maintenance jobs are similar to skip indices.

#### 查询分析 {#projection-query-analysis}

1. 检查投影是否能应答给定的查询，也就是说，查询投影表与查询父表都能返回相同的结果。
2. 选择最优匹配，也就是只需要读取最少颗粒。
3. 使用投影表的查询流水线会和不使用的有部分差异。如果投影表缺少了部分数据，可以让流水线实时进行投影。
4. Check if the projection can be used to answer the given query, that is, it generates the same answer as querying the base table.
5. Select the best feasible match, which contains the least granules to read.
6. The query pipeline which uses projections will be different from the one that uses the original parts. If the projection is absent in some parts, we can add the pipeline to "project" it on the fly.

## 并发数据访问 {#concurrent-data-access}

我们基于多版本机制实现并发数据访问。换言之，当一张表同时被读取和更新时，数据会从当前版本的数据片段中读取。没有冗长的锁。插入也不会阻碍读取。

对表的读操作是自动并行的。

## 列和表的 TTL {#table_engine-mergetree-ttl}

TTL 决定了值的生命周期，它既可以为整张表设置，也可以为每一列单独设置。表级别的 TTL 还可以指定数据在磁盘和卷上自动转移的逻辑。

TTL 表达式的计算结果必须是 [Date](../../../engines/table-engines/mergetree-family/mergetree.md) 或 [DateTime](../../../engines/table-engines/mergetree-family/mergetree.md) 类型的字段。

示例：

``` sql
TTL time_column
TTL time_column + interval
```

为了定义 `interval`, 需要使用[时间间隔](../../../engines/table-engines/mergetree-family/mergetree.md#operators-datetime)操作符。

``` sql
TTL date_time + INTERVAL 1 MONTH
TTL date_time + INTERVAL 15 HOUR
```

### 列 TTL {#mergetree-column-ttl}

当列中的值过期时, ClickHouse 会将它们替换成该列数据类型的默认值。如果数据片段中列的所有值均已过期，则 ClickHouse 会从文件系统中的数据片段中删除此列。

`TTL` 子句不能被用于主键字段。

**示例:**

创建表时指定 `TTL`

``` sql
CREATE TABLE example_table
(
    d DateTime,
    a Int TTL d + INTERVAL 1 MONTH,
    b Int TTL d + INTERVAL 1 MONTH,
    c String
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(d)
ORDER BY d;
```

为表中已存在的列字段添加 `TTL`

``` sql
ALTER TABLE example_table
    MODIFY COLUMN
    c String TTL d + INTERVAL 1 DAY;
```

修改列字段的 `TTL`

``` sql
ALTER TABLE example_table
    MODIFY COLUMN
    c String TTL d + INTERVAL 1 MONTH;
```

### 表 TTL {#mergetree-table-ttl}

表可以设置一个用于移除过期行的表达式，以及多个用于在磁盘或卷上自动转移数据片段的表达式。当表中的行过期时，ClickHouse 会删除所有对应的行。对于数据片段的转移特性，必须所有的行都满足转移条件。

``` sql
TTL expr
    [DELETE|TO DISK 'xxx'|TO VOLUME 'xxx'][, DELETE|TO DISK 'aaa'|TO VOLUME 'bbb'] ...
    [WHERE conditions]
    [GROUP BY key_expr [SET v1 = aggr_func(v1) [, v2 = aggr_func(v2) ...]] ]

```

TTL 规则的类型紧跟在每个 TTL 表达式后面，它会影响满足表达式时（到达指定时间时）应当执行的操作：

- `DELETE` - 删除过期的行（默认操作）;
- `TO DISK 'aaa'` - 将数据片段移动到磁盘 `aaa`;
- `TO VOLUME 'bbb'` - 将数据片段移动到卷 `bbb`.
- `GROUP BY` - 聚合过期的行

使用 `WHERE` 子句，您可以指定哪些过期的行会被删除或聚合（不适用于移动操作）。`GROUP BY` 表达式必须是表主键的前缀。如果某列不是 `GROUP BY` 表达式的一部分，也没有在 `SET` 子句中显式引用，结果中对应列的值是随机选择的（就好像使用了 `any` 聚集函数）。

**示例**:

创建时指定 TTL

``` sql
CREATE TABLE example_table
(
    d DateTime,
    a Int
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(d)
ORDER BY d
TTL d + INTERVAL 1 MONTH [DELETE],
    d + INTERVAL 1 WEEK TO VOLUME 'aaa',
    d + INTERVAL 2 WEEK TO DISK 'bbb';
```

修改表的 `TTL`

``` sql
ALTER TABLE example_table
    MODIFY TTL d + INTERVAL 1 DAY;
```

创建一张表，设置一个月后数据过期，这些过期的行中日期为星期一的删除：

``` sql
CREATE TABLE table_with_where
(
    d DateTime,
    a Int
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(d)
ORDER BY d
TTL d + INTERVAL 1 MONTH DELETE WHERE toDayOfWeek(d) = 1;
```

创建一张表，设置过期的列会被聚合。列 `x` 包含每组行中的最大值，`y` 为最小值，`d` 为可能任意值。

``` sql
CREATE TABLE table_for_aggregation
(
    d DateTime,
    k1 Int,
    k2 Int,
    x Int,
    y Int
)
ENGINE = MergeTree
ORDER BY (k1, k2)
TTL d + INTERVAL 1 MONTH GROUP BY k1, k2 SET x = max(x), y = min(y);
```

**删除数据**

ClickHouse 在数据片段合并时会删除掉过期的数据。

当 ClickHouse 发现数据过期时, 它将会执行一个计划外的合并。要控制这类合并的频率，您可以设置 `merge_with_ttl_timeout`。如果该值被设置的太低，它将引发大量计划外的合并，这可能会消耗大量资源。

如果在合并的过程中执行 `SELECT` 查询, 则可能会得到过期的数据。为了避免这种情况，可以在 `SELECT` 之前使用 [OPTIMIZE](../../../engines/table-engines/mergetree-family/mergetree.md#misc_operations-optimize) 查询。

## 使用多个块设备进行数据存储 {#table_engine-mergetree-multiple-volumes}

### 介绍 {#introduction}

MergeTree 系列表引擎可以将数据存储在多个块设备上。这对某些可以潜在被划分为“冷”“热”的表来说是很有用的。最新数据会被定期查询，但只需要占用很少的空间。另一方面，详尽的历史数据偶尔才会用到。如果有多块磁盘可用，那么“热”的数据可以放置在快速的磁盘上（比如 NVMe 固态硬盘或内存），“冷”的数据可以放在相对较慢的磁盘上（比如机械硬盘）。

数据片段是 `MergeTree` 引擎表的最小可移动单元。属于同一个数据片段的数据被存储在同一块磁盘上。数据片段会在后台自动的在磁盘间移动，也可以通过 [ALTER](../../../sql-reference/statements/alter.md#alter_move-partition) 查询来移动。

### 术语 {#terms}

- 磁盘 — 挂载到文件系统的块设备
- 默认磁盘 — 在服务器设置中通过 [path](../../../operations/server-configuration-parameters/settings.md#server_configuration_parameters-path) 参数指定的数据存储
- 卷 — 相同磁盘的有序列表（类似于 [JBOD](https://en.wikipedia.org/wiki/Non-RAID_drive_architectures)）
- 存储策略 — 卷的集合及它们之间的数据移动规则

以上名称的信息在 Clickhouse 中系统表 [system.storage_policies](https://clickhouse.tech/docs/zh/operations/system-tables/storage_policies/#system_tables-storage_policies) 和 [system.disks](https://clickhouse.tech/docs/zh/operations/system-tables/disks/#system_tables-disks) 体现。要应用存储策略，可以在建表时指定 `storage_policy` 配置。

### 配置 {#table_engine-mergetree-multiple-volumes_configure}

磁盘、卷和存储策略应当在主配置文件 `config.xml` 或 `config.d` 目录中配置文件中的 `<storage_configuration>` 标签内定义。

配置结构：

``` xml
<storage_configuration>
    <disks>
        <disk_name_1> <!-- 磁盘名 -->
            <path>/mnt/fast_ssd/clickhouse/</path>
        </disk_name_1>
        <disk_name_2>
            <path>/mnt/hdd1/clickhouse/</path>
            <keep_free_space_bytes>10485760</keep_free_space_bytes>
        </disk_name_2>
        <disk_name_3>
            <path>/mnt/hdd2/clickhouse/</path>
            <keep_free_space_bytes>10485760</keep_free_space_bytes>
        </disk_name_3>

        ...
    </disks>

    ...
</storage_configuration>
```

标签：

- `<disk_name_N>` — 磁盘名，名称必须与其他磁盘不同。
- `path` — 服务器存储数据 (`data` 和 `shadow` 目录) 的路径, 应当以 ‘/’ 结尾。
- `keep_free_space_bytes` — 需要保留的剩余磁盘空间。

可以以任意顺序定义磁盘。

存储策略配置：

``` xml
<storage_configuration>
    ...
    <policies>
        <policy_name_1>
            <volumes>
                <volume_name_1>
                    <disk>disk_name_from_disks_configuration</disk>
                    <max_data_part_size_bytes>1073741824</max_data_part_size_bytes>
                </volume_name_1>
                <volume_name_2>
                    <!-- 卷配置 -->
                </volume_name_2>
                <!-- 可以加入更多的卷 -->
            </volumes>
            <move_factor>0.2</move_factor>
        </policy_name_1>
        <policy_name_2>
            <!-- 策略配置 -->
        </policy_name_2>

        <!-- 可以加入更多策略 -->
    </policies>
    ...
</storage_configuration>
```

标签：

- `policy_name_N` — 策略名称，不能重复。
- `volume_name_N` — 卷名称，不能重复。
- `disk` — 卷中的磁盘。
- `max_data_part_size_bytes` — 卷中磁盘可存储的数据片段的大小上限。
- `move_factor` — 当可用空间少于这个因子时，数据将自动的向下一个卷（如果有的话）移动（默认值为 0.1）。
- `prefer_not_to_merge` - 禁止在这个卷中进行数据合并。该选项启用时，禁止对该卷的数据进行合并。主要用于慢速磁盘。

配置示例：

``` xml
<storage_configuration>
    ...
    <policies>
        <hdd_in_order> <!-- 策略名 -->
            <volumes>
                <single> <!-- 卷名 -->
                    <disk>disk1</disk>
                    <disk>disk2</disk>
                </single>
            </volumes>
        </hdd_in_order>

        <moving_from_ssd_to_hdd>
            <volumes>
                <hot>
                    <disk>fast_ssd</disk>
                    <max_data_part_size_bytes>1073741824</max_data_part_size_bytes>
                </hot>
                <cold>
                    <disk>disk1</disk>
                </cold>
            </volumes>
            <move_factor>0.2</move_factor>
        </moving_from_ssd_to_hdd>

        <small_jbod_with_external_no_merges>
            <volumes>
                <main>
                    <disk>jbod1</disk>
                </main>
                <external>
                    <disk>external</disk>
                    <prefer_not_to_merge>true</prefer_not_to_merge>
                </external>
            </volumes>
        </small_jbod_with_external_no_merges>
    </policies>
    ...
</storage_configuration>
```

在给出的例子中， `hdd_in_order` 策略实现了[轮询](https://zh.wikipedia.org/wiki/循环制)方法。因此这个策略只定义了一个卷（`single`），数据片段会以循环的顺序全部存储到它的磁盘上。当有多个类似的磁盘挂载到系统上，但没有配置 RAID 时，这种策略非常有用。但请注意，这些磁盘都不保证数据的可靠性，您可能需要用 3 份以上的副本机制来确保可靠性。

如果在系统中有不同类型的磁盘可用，可以使用 `moving_from_ssd_to_hdd` 策略。`hot` 卷由 SSD 磁盘（`fast_ssd`）组成，这个卷上可以存储的数据片段的最大大小为 1GB。所有大于 1GB 的数据片段都会被直接存储到 `cold` 卷上，`cold` 卷包含一个名为 `disk1` 的 HDD 磁盘。
同样，一旦 `fast_ssd` 使用了 80% 的容量，数据会通过后台进程向 `disk1` 进行转移。

存储策略中卷的枚举顺序是很重要的。因为当一个卷容量不足时，数据会向下一个卷转移。磁盘的枚举顺序同样重要，因为数据是依次存储在磁盘上的。

在创建表时，可以应用存储策略：

``` sql
CREATE TABLE table_with_non_default_policy (
    EventDate Date,
    OrderID UInt64,
    BannerID UInt64,
    SearchPhrase String
) ENGINE = MergeTree
ORDER BY (OrderID, BannerID)
PARTITION BY toYYYYMM(EventDate)
SETTINGS storage_policy = 'moving_from_ssd_to_hdd'
```

`default` 存储策略意味着只使用一个卷，这个卷只包含一个在 `<path>` 中定义的磁盘。您可以使用 [ALTER TABLE ... MODIFY SETTING] 来修改存储策略，新的存储策略应该包含所有以前的磁盘和卷，这些磁盘和卷都需要保持和之前相同的名称。

可以通过 [background_move_pool_size](../../../operations/settings/settings.md#background_move_pool_size) 设置调整执行后台转移任务的线程数。

### 详细说明 {#details}

对于 `MergeTree` 表，数据通过以下几种方式写入到磁盘当中：

- 插入（`INSERT` 查询）
- 后台合并和[数据变异](../../../sql-reference/statements/alter.md#alter-mutations)
- 复制另一份副本
- 使用 [ALTER TABLE … FREEZE PARTITION](../../../sql-reference/statements/alter.md#alter_freeze-partition) 冻结分区

除数据变异和冻结分区以外，数据按照以下逻辑存储到卷或磁盘上：

1. 按照定义顺序，选择拥有充足空间（`unreserved_space > current_part_size`）并且能够容纳指定数据片段（`max_data_part_size_bytes > current_part_size`）的第一个卷
2. 在这个数据卷内，从前一个数据存储磁盘开始，选择下一个能够容纳该数据片段（`unreserved_space - keep_free_space_bytes > current_part_size`）的磁盘。

数据变异和分区冻结是基于[硬链接](https://en.wikipedia.org/wiki/Hard_link)实现的。由于不支持在不同磁盘之间建立硬链接，在这种情况下，数据片段都会被存储到原来的那一块磁盘上。

在后台，数据片段基于剩余空间因数（`move_factor`参数）按照卷在配置文件中定义的顺序进行转移。
数据永远不会从最后一个卷转移到第一个卷。用户可以通过系统表 [system.part_log](../../../operations/system-tables/part_log.md#system_tables-part-log)（字段 `type = MOVE_PART`）和 [system.parts](../../../operations/system-tables/parts.md#system_tables-parts)（字段 `path` 和 `disk`）监控后台的移动情况。具体细节可以通过服务器日志查看。

用户可以通过 [ALTER TABLE … MOVE PART\|PARTITION … TO VOLUME\|DISK …](../../../sql-reference/statements/alter.md#alter_move-partition) 强制移动一个数据片段或分区到另外一个卷。该查询会遵守所有后台移动的限制，并在稍后自行启动移动作业，无需等待其它后台操作完成。如果没有足够的可用空间或不满足任何必须条件，用户会收到报错信息。

数据移动不会妨碍到数据复制。也就是说，同一张表的不同副本可以指定不同的存储策略。

在后台合并和数据变异之后，旧的数据片段会在一定时间后被移除 (`old_parts_lifetime`)。
在此期间，它们不能被移动到其它的卷或磁盘。也就是说，直到数据片段被完全移除之前，它们还是会算在磁盘占用空间里面。

## 使用 S3 进行数据存储 {#using-s3-data-storage}

`MergeTree`系列表引擎允许使用 [S3](https://aws.amazon.com/s3/) 存储数据，需要修改磁盘类型为 `S3`。

示例配置：

``` xml
<storage_configuration>
    ...
    <disks>
        <s3>
            <type>s3</type>
            <endpoint>https://storage.yandexcloud.net/my-bucket/root-path/</endpoint>
            <access_key_id>your_access_key_id</access_key_id>
            <secret_access_key>your_secret_access_key</secret_access_key>
            <region></region>
            <server_side_encryption_customer_key_base64>your_base64_encoded_customer_key</server_side_encryption_customer_key_base64>
            <proxy>
                <uri>http://proxy1</uri>
                <uri>http://proxy2</uri>
            </proxy>
            <connect_timeout_ms>10000</connect_timeout_ms>
            <request_timeout_ms>5000</request_timeout_ms>
            <retry_attempts>10</retry_attempts>
            <single_read_retries>4</single_read_retries>
            <min_bytes_for_seek>1000</min_bytes_for_seek>
            <metadata_path>/var/lib/clickhouse/disks/s3/</metadata_path>
            <cache_enabled>true</cache_enabled>
            <cache_path>/var/lib/clickhouse/disks/s3/cache/</cache_path>
            <skip_access_check>false</skip_access_check>
        </s3>
    </disks>
    ...
</storage_configuration>
```

必须的参数：

- `endpoint` - S3 的端点URL，以 `path` 或 [`virtual hosted` 风格](https://docs.aws.amazon.com/AmazonS3/latest/dev/VirtualHosting.html)书写。
- `access_key_id` - S3 的 Access Key ID。
- `secret_access_key` - S3 的 Secret Access Key。

可选参数：

- `region` - S3 的区域名称
- `use_environment_credentials` - 从 AWS_ACCESS_KEY_ID、AWS_SECRET_ACCESS_KEY 和 AWS_SESSION_TOKEN 环境变量中读取认证参数。默认值为 `false`。
- `use_insecure_imds_request` - 如果设置为 `true`，S3客户端在认证时会使用不安全的 IMDS 请求。默认值为`false`。
- `proxy` - 访问 S3 端点 URL 时的代理设置。每一个 `uri` 项的值都应该是合法的代理 URL。
- `connect_timeout_ms` - Socket 连接超时时间，默认值为 `10000`，即 10 秒。
- `request_timeout_ms` - 请求超时时间，默认值为 `5000`，即 5 秒。
- `retry_attempts` - 请求失败后的重试次数，默认值为 10。
- `single_read_retries` - 读过程中连接丢失后重试次数，默认值为 4。
- `min_bytes_for_seek` - 使用查找操作，而不是顺序读操作的最小字节数，默认值为 1000。
- `metadata_path` - 本地存放S3元数据文件的路径，默认值为 `/var/lib/clickhouse/disks/<disk_name>/`
- `cache_enabled` - 是否允许缓存标记和索引文件。默认值为 `true`。
- `cache_path` - 本地缓存标记和索引文件的路径。默认值为 `/var/lib/clickhouse/disks/<disk_name>/cache/`。
- `skip_access_check` - 如果为 `true`，Clickhouse 启动时不检查磁盘是否可用。默认为 `false`。
- `server_side_encryption_customer_key_base64` - 如果指定该项的值，请求时会加上为了访问 SSE-C 加密数据而必须的头信息。

S3 存储也可以设置冷热存储：
```xml
<storage_configuration>
    ...
    <disks>
        <s3>
            <type>s3</type>
            <endpoint>https://storage.yandexcloud.net/my-bucket/root-path/</endpoint>
            <access_key_id>your_access_key_id</access_key_id>
            <secret_access_key>your_secret_access_key</secret_access_key>
        </s3>
    </disks>
    <policies>
        <s3_main>
            <volumes>
                <main>
                    <disk>s3</disk>
                </main>
            </volumes>
        </s3_main>
        <s3_cold>
            <volumes>
                <main>
                    <disk>default</disk>
                </main>
                <external>
                    <disk>s3</disk>
                </external>
            </volumes>
            <move_factor>0.2</move_factor>
        </s3_cold>
    </policies>
    ...
</storage_configuration>
```

指定了 `cold` 选项后，本地磁盘剩余空间如果小于 `move_factor * disk_size`，或配置了 TTL 时，数据就会定时迁移至 S3 了。

[原始文章](https://clickhouse.tech/docs/en/operations/table_engines/mergetree/) <!--hide-->
