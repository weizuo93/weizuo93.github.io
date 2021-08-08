---
layout: post
title: Apache Doris 查询原理解析
categories: [Apache Doris]
description: Apache Doris 查询原理解析。
keywords: Apache Doris、SQL解析、执行计划
---

## Apache Doris 查询原理解析

### 1. 引言

Apache Doris的查询需要经过查询接收、SQL解析、单节点执行计划的生成、分布式执行计划的生成、执行计划的下发、查询计划的执行、查询结果的返回等步骤，其中查询接收、SQL解析、单节点执行计划的生成、分布式执行计划的生成、执行计划的下发以及查询结果的返回由FE负责，查询计划的执行由BE负责。

### 2. 查询接收

Apache Doris 兼容 Mysql 协议，用户可以通过 Mysql 客户端和其他支持 Mysql 协议的工具向 Doris 发送查询请求。MysqlServer负责接收客户端发送来的Mysql连接请求，每个连接请求都被封装成一个ConnectContext对象，并被提交给ConnectScheduler。ConnectScheduler 会维护一个线程池，每个 ConnectContext 会在线程池中由一个 ConnectProcessor 线程处理。

### 3. SQL解析

#### 3.1 SQL解析

ConnectProcessor 会首先对查询 SQL 进行解析，采用FLEX进行词法分析，采用cup进行语法分析，生成抽象语法树（AST，Abstract Syntax Tree ）。

不同类型的SQL（select, insert, show, set, alter table, create table等）经过解析后生成不同的AST（SelectStmt, InsertStmt, ShowStmt, SetStmt, AlterStmt, AlterTableStmt, CreateTableStmt等，但他们都继承自StatementBase）。SelectStmt结构包含了SelectList，FromClause，WhereClause，GroupByClause，SortInfo等结构，这些结构又包含了更基础的一些数据结构，如WhereClause包含了BetweenPredicate（between表达式）, BinaryPredicate（二元表达式）， CompoundPredicate（and or组合表达式）, InPredicate（in表达式）等。

#### 3.2 语法和语义分析

SQL语句被解析成AST之后，会被交给 StmtExecutor 。StmtExecutor 会首先对 AST 进行语法和语义分析，大概会做下面的事情：
* 检查并绑定 Cluster, Database, Table, Column 等元信息。
* SQL 的合法性检查：窗口函数不能 DISTINCT，HLL 和 Bitmap 列不能 sum, count, where 中不能有 grouping 操作等。
* SQL 重写：比如将 select * 扩展成 select 所有列，count distinct 查询重写等。
* Table与Column别名处理。
* 为Tuple, Slot, Expr等分配唯一ID。
* 函数参数的合法性检测。
* 表达式替换。
* 类型检查，类型转换（BIGINT 和 DECIMAL 比较，BIGINT 类型需要 Cast 成 DECIMAL）。

#### 3.3 查询重写

StmtExecutor 在对 AST 进行语法和语义分析后，会让 ExprRewriter 根据 ExprRewriteRule 进行一次 Rewrite。目前 Doris 的重写规则比较简单，主要是进行了常量表达式的化简和谓词的简单处理。 常量表达式的化简是指 `1 + 1 + 1` 重写成 `3`，`1 > 2` 重写成 `Flase` 等。如果重写后，有部分节点被成功改写，比如， `1 > 2` 被改写成 `Flase`，那么就会再触发一次语法和语义分析的过程。对于有子查询的 SQL，StmtRewriter 会进行重写，比如将 where in, where exists 重写成 semi join, where not in, where not exists 重写成 anti join。

### 4. 单节点执行计划

单机 Plan 由 SingleNodePlanner 执行，输入是 AST，输出是单机物理执行 Plan, Plan 中每个节点是一个 PlanNode。SingleNodePlanner 核心任务就是根据 AST 生成 OlapScanNode, AggregationNode, HashJoinNode, SortNode, UnionNode 等。

单机的逻辑执行计划中，主要做以下的工作:
* Slot物化：指确定一个表达式对应的列需要 Scan 和计算，比如聚合节点的聚合函数表达式和 Group By 表达式需要进行物化。
* 投影下推：BE 在 Scan 时会 Scan 哪些列。
* 谓词下推：在满足语义正确的前提下将过滤条件尽可能下推到 Scan 节点。
* 分区，分桶裁剪：比如建表时按照 UserId 分桶，每个分区 100 个分桶，那么当不包含 or 的 Filter 条件包含 UserId ==xxx 时，Doris 就只会将查询发送 100 个分桶中的一个发送给 BE，可以大大减少不必要的数据读取。
* Join Reorder：对于 Inner Join, Doris 会根据行数调整表的顺序，将大表放在前面。
* 将Sort + Limit 优化成 TopN。
* 物化视图选择：根据查询需要的列，过滤、排序和 Join 的列，行数、列数等因素选择最佳的 物化视图。

### 5. 分布式执行计划

有了单机的 Plan 之后，DistributedPlanner 就会根据单节点执行计划的 PlanNode 树，生成 PlanFragment 树。分布式化的目标是最小化数据移动和最大化本地 Scan。

Plan 分布式化的方法是增加 ExchangeNode，执行计划树会以 ExchangeNode 为边界拆分为 PlanFragment。ExchangeNode 的作用是实现不同 BE 之间的数据交换，类似于 Spark 和 MR 中的 Shuffle。各个 Fragment 的数据流转和最终的结果发送依赖DataSink。比如 DataStreamSink 会将一个 Fragment 的数据发送到另一个 Fragment 的 ExchangeNode，ResultSink 会将查询的结果集发送到 FE。每个 PlanFragment 可以在一个 BE 节点生成 1 个或多个执行实例，不同执行实例处理不同的数据集，通过并发来提升查询性能。

DistributedPlanner 中最主要的工作是决定 Join 的分布式执行策略（Shuffle Join，Broadcast Join，Colocate Join）和增加 Aggregation 的 Merge 阶段。决定 Join 的分布式执行策略的逻辑如下：
* 如果两种表是 Colocate Join 表，且Join 的 Key 和分桶的 Key 一致，同时两张表没有在做数据 balance，则会执行 Colocate Join。
* 如果 Join 的右表比较少，集群节点数较少，执行 Broadcast Join 成本较低，则会选择 Broadcast Join；否则，会选择 Shuffle Join。

### 6. 执行计划的调度

生成了逻辑执行计划后，Coordinator会负责生成Fragment的执行实例并进行调度管理，要保证生成正确的物理执行计划需要做以下几件事:
* 将哪PlanFragment下发到哪个 BE 执行。
* 如何选择待查询的Tablet的副本，这个过程中需要尽可能地保证每台BE节点的负载均衡。
* 如何进行多实例并发。

在生成单节点执行计划过程中，会先进行分区和分桶裁剪，得到需要访问的 Tablet 列表。需要对每个 Tablet选择版本匹配的、健康的、所在的 BE 状态正常的副本进行查询，在决定每个 Tablet 最终选择哪个副本查询是随机的，不过 会尽可能保证每个 BE 的请求均衡。假如我们有 10 个 BE，10 个 tablet，最终调度的结果理论上就是每个 BE 负责 1 个 tablet 的 Scan。具体逻辑在computeScanRangeAssignmentByScheduler。

当包含 Scan 的 PlanFragment 由哪些 BE 节点执行确定后，其他的 PlanFragment 实例也会在 Scan 的 BE 节点上执行，不过具体选择哪个 BE 是随机选取的。当每个 PlanFragment 实例的 BE 节点确定后，每个 DataSink 的目标 BE 节点自然也就确定了。

多实例并发执行的话，是数据并行的方式，假如我们有 10 个 tablet，并行度设置为 5 的话，那么 Scan 所在的 PlanFragment，每个 BE 上我们可以生成 5 个执行实例，每个执行实例会分别 Scan 2 个 tablet。

当我们知道每个 PlanFragment 需要生成多少个执行实例，每个执行实例在哪个 BE 执行后，FE 就会将 PlanFragment 执行相关的参数通过 Thrift 的方式发送给 BE。

### 7. 查询计划的执行

BE 的 BackendService 会接收 FE 的 查询请求，让 FragmentMgr 进行处理。 FragmentMgr::exec_plan_fragment 会启动一个线程由 PlanFragmentExecutor 具体执行一个 plan fragment。PlanFragmentExecutor 会根据 plan fragment 创建一个 ExecNode 树，FE 每个 PlanNode 都会对应 ExecNode 的一个子类。不同的 operator 之间以 RowBatch 的方式传输数据。

`PlanFragmentExecutor::get_next_internal` 会驱动整个 ExecNode 树的执行，会自顶向下调用每个 ExecNode 的 get_next 方法，最终数据会从 ScanNode 节点产生，向上层节点传递，每个节点都会按照自己的逻辑处理 RowBatch。 PlanFragmentExecutor 在拿到每个 RowBatch 后，如果是中间结果，就会将数据传输给其他 BE 节点，如果是最终结果，就会将数据传输给 FE 节点。


（参考：https://blog.bcmeng.com/post/apache-doris-query.html）