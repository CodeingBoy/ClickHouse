# 什么是 ClickHouse？ {#what-is-clickhouse}

ClickHouse 是一个用于联机分析（OLAP）的列式数据库管理系统（DBMS）。

在传统的行式数据库系统中，数据按如下顺序存储：

| Row | WatchID     | JavaEnable | Title              | GoodEvent | EventTime           |
|-----|-------------|------------|--------------------|-----------|---------------------|
| #0 | 89354350662 | 1          | Investor Relations | 1         | 2016-05-18 05:19:20 |
| #1 | 90329509958 | 0          | Contact us         | 1         | 2016-05-18 08:10:20 |
| #2 | 89953706054 | 1          | Mission            | 1         | 2016-05-18 07:38:00 |
| #N | …           | …          | …                  | …         | …                   |

位于同一行中的数据总是被物理地的存储在一起。

常见的行式数据库系统有：MySQL、Postgres 和 MS SQL Server。

而在列式数据库系统中，数据按如下的顺序存储：

| Row:        | #0                 | #1                 | #2                 | #N |
|-------------|---------------------|---------------------|---------------------|-----|
| WatchID:    | 89354350662         | 90329509958         | 89953706054         | …   |
| JavaEnable: | 1                   | 0                   | 1                   | …   |
| Title:      | Investor Relations  | Contact us          | Mission             | …   |
| GoodEvent:  | 1                   | 1                   | 1                   | …   |
| EventTime:  | 2016-05-18 05:19:20 | 2016-05-18 08:10:20 | 2016-05-18 07:38:00 | …   |

这些示例只显示了数据的排列顺序。来自不同列的值被单独存储，来自同一列的数据被存储在一起。

常见的列式数据库有：Vertica、 Paraccel (Actian Matrix，Amazon Redshift)、 Sybase IQ、 Exasol、 Infobright、 InfiniDB、 MonetDB (VectorWise， Actian Vector)、 LucidDB、 SAP HANA、 Google Dremel、 Google PowerDrill、 Druid、 kdb+。

不同的数据存储方式适用不同的业务场景。数据访问场景包含以下因素：查询的种类、频次以及比例；每种查询将读取多少数据（以行、列或字节计）；数据读取和更新的关系；工作数据集的大小以及本地访问的频率；事务的利用程度以及隔离级别；数据副本机制和完整性要求；对每种查询的延迟及吞吐量要求等等。

系统承受的负载越高，就越凸显在使用场景下的定制化的重要性，也就越要求定制化的精细和深入。没有一个系统能够同时适用所有不同的业务场景。如果一个适应灵活的系统，在高负载的情况下，要么降低性能两头不讨好，要么对少数特定的场景做出优化。

## OLAP 场景的关键特征 {#key-properties-of-olap-scenario}

-   绝大多数是读请求
-   数据的更新以批次进行（\> 1000 ）更新，不会出现单行更新，或者根本就不会更新数据
-   数据添加到数据库后不能修改
-   读取大量的行，却只需要其中的小部分列
-   每个表包含着大量的列，即宽表
-   查询相对较少（通常服务器每秒的查询数不多于数百次）
-   对简单的查询，要求能在 50 ms 左右给出响应
-   列中的数据相对较小，基本是数字和短字符串（例如长度为 60 个字节的 URL）
-   处理单个查询时需要高吞吐量（每台服务器每秒可访问数十亿行）
-   事务不是必需的
-   对数据一致性要求低
-   每个查询有一个大表。其它的表都比较小
-   查询结果明显少于源数据。也就是数据会经过过滤或聚合，因此结果可以放入单台服务器的内存中

很容易可以看出，OLAP 场景与其他通常业务场景（如 OLTP 或键值查询）有很大的不同，因此想要使用 OLTP 或键值数据库去高效地处理分析查询场景，并不是非常完美的方案。如果你尝试使用 MongoDB 或 Redis 来处理分析查询，性能上会劣于使用 OLAP 数据库。

## 列式数据库更适合 OLAP 场景的原因 {#why-column-oriented-databases-work-better-in-the-olap-scenario}

列式数据库更适用于 OLAP 场景。对于大多数查询而言，处理速度至少提高了 100 倍。下面利用图片直观解释了原因：

**行式数据库**

![Row oriented](images/row-oriented.gif#)

**列式数据库**

![Column oriented](images/column-oriented.gif#)

看出差别了吗？

### 输入/输出 {#inputoutput}

1.  对于分析类查询，通常只需要读取表的一小部分列。在列式数据库中，你可以只读取你需要的数据。例如，如果只读取 100 列中的 5 列，至少能够帮你节省 20 倍的 I/O 消耗。
2.  由于数据是批量读取的，数据压缩变得更加简单。并且按列存储的数据也提升了数据的压缩率。进一步降低了 I/O 传输的大小。
3.  由于 I/O 数据减小，系统能够缓存更多的数据。

例如，查询“统计每个广告平台的记录数量”需要读取“广告平台ID”这一列，它在未压缩的情况下需要 1 个字节进行存储。如果大部分流量并非来自广告平台，那么这一列至少可以以十倍的压缩率进行压缩。当采用快速压缩算法，单台服务器每秒可以至少解压出好几 GB 的数据。换句话说，这个查询可以在单台服务器上以每秒约几十亿行的速度进行处理。实际上，这已经是当前实现了的速度。

### CPU {#cpu}

由于查询执行过程中需要处理大量的行，在整个向量上执行所有操作会比在每一行上执行更加高效。甚至还能实现一个几乎没有调用成本的查询引擎。否则在中低端的磁盘系统上，查询引擎都将不可避免地停止 CPU 进行等待。所以数据不仅需要按列存储，还需要尽可能地按列执行。

为了达成这个目的，我们采取了两种方法：

1.  向量引擎：所有的操作都面向向量而不是单个值。也就意味着，不需要频繁调用多个操作，且调用的成本可以基本忽略不计。操作地代码包含一个经过优化的内部循环。

2.  代码生成：为查询生成代码，包含了查询需要的所有操作。

这些方法不适用一个通用型的数据库，这是因为对简单查询，这些措施收效不大。但是也有例外，例如，MemSQL 使用代码生成来减少处理 SQL 查询的延迟（注意，分析型数据库更关注的是吞吐量的优化，而不是查询延迟)。

为了提高 CPU 效率，查询语言必须是声明式的（SQL 或 MDX），或者至少一个向量 (J，K)。查询应该只包含隐式循环，以允许数据库进行优化。

[来源文章](https://clickhouse.tech/docs/zh/) <!--hide-->
