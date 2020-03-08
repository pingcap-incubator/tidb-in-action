# TiDB 整体架构

当今时代，我们的数据越来越庞大，应用的规模也越来越复杂，在这个背景之下，传统的单机数据库已经在很多场景下表现得力不从心，为了解决海量数据平台的扩展性的问题，TiDB 应运而生。
TiDB 是当今开源 NewSQL 数据库领域的代表产品之一，相比传统的单机数据库，TiDB 有以下的一些优势：

* 纯分布式架构，拥有良好的扩展性，支持弹性的扩缩容
* 支持 SQL，对外暴露 MySQL 的网络协议，并兼容大多数 MySQL 的语法，大多数场景下是可以直接替换 MySQL
* 默认支持高可用，在部分节点失效的情况下，数据库本身自动进行数据修复和故障转移，对业务透明
* 支持 ACID 事务，对于一些有强一致需求的场景友好，例如：银行转账
* 有丰富的工具链生态，覆盖数据迁移，同步，备份等场景

本书会专注于 TiDB 4.0 的实操与最佳实践，详细介绍 TiDB 的使用和一些相关的原理。

宏观上的 TiDB 最初的设计受到 Google 内部开发的知名分布式数据库 Spanner 和 F1 的启发，在内核设计上将整体的架构拆分成多个大的模块，大的模块之间互相通信，组成完整的 TiDB 系统。大的架构如下：

![1.png](/res/session1/chapter1/tidb-architecture/1.png)

这三个大模块相互通信，每个模块都是分布式的架构，在 TiDB 中，对应的这几个模块叫做：

![2.png](/res/session1/chapter1/tidb-architecture/2.png)

TiDB (tidb-server, [https://github.com/pingcap/tidb](https://github.com/pingcap/tidb)): SQL 层，对外暴露 MySQL 协议的链接 endpoint，负责接受客户端的链接，包含完整的 SQL 解析，优化，生成分布式执行计划。TiDB 层本身是无状态的，实践中可以启动多个 TiDB 实例，客户端的链接可以均匀的分摊在多个 TiDB 实例上以达到负载均衡的效果。tidb-server 本身并不存储数据，只是解析 SQL，将实际的数据读取请求转发给底层的存储层 TiKV。

TiKV (tikv-server, [https://github.com/pingcap/tikv](https://github.com/pingcap/tikv)) : 分布式 KV 存储，类似 NoSQL 数据库，作为 TiDB 的默认分布式存储引擎，支持完全弹性的扩容和缩容，数据分布在多个 tikv 存储节点中，系统会动态且自动的进行均衡，绝大多数情况下不需要人工介入，与普通的 NoSQL 系统不一样的是，TiKV 的 API 原生支持在 KV 键值对层面的分布式事务，默认提供了 SI （Snapshot Isolation）的隔离级别，这也是 TiDB 在 SQL 层面支持分布式事务的核心，上面提到的 TiDB SQL 层做完 SQL 解析后，会将 SQL 的执行计划变成实际的 TiKV 的 KV API 调用。所以实际上数据都是存储在 TiKV 中。另外，TiKV 中的数据都会自动维护多副本（默认为 3），天然支持高可用和自动故障转移。TiFlash 是一类特殊的存储节点：TiFlash，和普通 TiKV 节点不一样的是，在 TiFlash 内部数据是以列式的形式存储，主要的功能是为分析型的场景加速。后面的章节会详细介绍。

Placement Driver (简称 PD，[https://github.com/pingcap/pd](https://github.com/pingcap/pd)): 整个 TiDB 集群的元信息管理模块，负责存储每个 TiKV 节点实时的数据分布情况，整体集群拓扑结构，提供 Dashboard 管控界面，并为分布式事务分配事务 ID。PD 不仅仅是单纯的元信息存储，同时 PD 会根据实时的数据分布状态，下发数据调度命令给具体的 TiKV 节点，可以说是整个集群的「大脑」，另外 PD 本身也是由至少 3 个对等节点构成，拥有高可用的能力。
