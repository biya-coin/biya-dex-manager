# DEX 管理后台：MPP/OLAP 数据库选型（ClickHouse vs Doris vs StarRocks 等）

本文面向“DEX 管理后台（订单/资金等海量列表）”的**数据存储与查询**需求，梳理常见 MPP/OLAP 数据库/引擎，并结合本项目数据特点给出**最优选型**与落地建议。

---

## 1. 背景与目标

管理后台的“订单管理/资金管理/资金明细”等页面存在：

- **数据量海量**：订单、成交、充提、余额快照、风控/杠杆事件等持续增长。
- **查询交互强**：列表页按多条件筛选、排序、分页，且需要秒级（或亚秒级）响应。
- **数据更新不可避免**：订单状态、成交进度、失败原因补充、余额对账口径等需要“更新/覆盖/幂等写”。

因此，不适合用单机/传统 OLTP（如 MySQL）承担主要查询压力；更适合采用**MPP 架构的列式分析型数据库（OLAP）**作为主存储与查询引擎。

---

## 2. 本项目数据特点（结合链上事件/接口梳理）

从链上事件与 Query 能力看，本项目落地通常会采用“链上为真源 + 链下索引/聚合”：

- **事实类数据（append-only 为主）**

  - 成交明细、交易流水、借贷利息分配、申购/赎回事件等：天然按时间追加写入。
  - 典型特征：写入吞吐大、按时间范围扫描/聚合频繁、压缩比高。

- **状态类数据（upsert/覆盖写为主）**

  - 订单“最新状态/累计成交量/成交额/失败原因”、余额对账口径（available/total/hold）、风控指标快照等。
  - 典型特征：同一主键会被多次写入（版本更新），需要**幂等 upsert**（按 block_height/tx_index/version 覆盖）。

- **维表/映射类数据（小表）**

  - market_id→ticker、denom→symbol、用户 Uxxx→subaccount_id 等：数据量小，但 join 频繁。

> 结论：这不是纯“日志分析”场景，而是 **append-only + upsert 混合**，并且有 join 与高并发点查/分页需求。

---

## 3. 典型查询模式（后台列表）

订单/资金列表通常具备：

- **时间范围过滤**（start/end）
- **多条件过滤**（用户/子账户、market、denom、状态、方向、订单类型等）
- **排序**（时间倒序、金额倒序）
- **分页**（高页数 offset 成本高，需要避免深分页）
- **联查/展示字段**（market_id→ticker、denom→symbol、用户映射等）

因此数据库需要：

- 列式存储 + 向量化执行 + MPP 并行
- 高并发点查/中等复杂 join
- 对 upsert/删除/版本覆盖有明确模型
- 支持物化视图/预聚合以减少实时计算开销

---

## 4. 常见 MPP/OLAP 数据库/引擎（OpenWebSearch 汇总）

下面这些是目前工程上常见的“MPP/分布式 OLAP”候选：

- **Apache Doris（MPP 实时分析型数据库）**

  - Unique Key 模型支持更新覆盖写（upsert），适合“最新状态表/宽表”。
  - 资料：Unique Key Table、Update Overview、MySQL 协议连接、Kafka Routine Load 等（见参考资料）。

- **StarRocks（分布式实时分析数据库）**

  - Primary Key 表支持 upsert/delete，适合实时更新。
  - 资料：Primary Key 表的 upsert/delete（见参考资料）。

- **ClickHouse（列式 OLAP，分布式）**

  - 查询性能极强、写入吞吐高；但 UPDATE/DELETE 属于 mutation，通常不建议高频。
  - 可用 ReplacingMergeTree 做“插入多版本 + 去重”，但一致性与实现复杂度更高（见参考资料）。

- **Greenplum（基于 PostgreSQL 的 MPP 数据仓库）**

  - 传统 MPP 数仓路线，SQL 能力强，生态成熟；但整体更偏“数仓/离线 + 重运维”，实时写入与低延迟高并发 OLAP 未必最优（见参考资料）。

- **Apache Impala（MPP SQL 引擎，面向 Hadoop 存储）**

  - 偏“查询引擎”而不是独立存储，通常依赖 HDFS/Hive/HBase；对本项目“实时 upsert + 在线服务化”落地成本高（见参考资料）。

- **Apache Pinot / Apache Druid（实时 OLAP 数据库）**

  - 更适合“指标看板/用户侧分析”类场景；复杂 join、强一致 upsert 并非强项，做后台订单/资金的任意组合筛选会受限（见参考资料）。

---

## 5. 评估维度（针对本项目）

| 维度              | 为什么重要                                      |
| ----------------- | ----------------------------------------------- |
| 写入吞吐/实时摄入 | 链上事件持续产生，落库是常态                    |
| Upsert/删除支持   | 订单/余额/状态表需要覆盖更新与幂等              |
| 查询延迟与并发    | 后台列表/导出/筛选需要稳定 SLA                  |
| Join/SQL 完整性   | 后台字段常需要关联维表与多条件过滤              |
| 物化视图/预聚合   | 降低每日统计、趋势图等开销                      |
| 运维复杂度与生态  | 团队落地速度、成本、稳定性                      |
| Go 接入成本       | 你们后端倾向 Go（gin/gorm），最好协议与生态成熟 |

---

## 6. 核心对比（面向“订单/资金海量列表 + 实时更新”）

| 候选                     | 优势                                                                                            | 关键短板/风险                                                                                               | 对本项目适配度                   |
| ------------------------ | ----------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- | -------------------------------- |
| **Apache Doris（推荐）** | MPP 实时 OLAP；**Unique Key upsert**；MySQL 协议生态；Kafka Routine Load；物化视图/增量刷新能力 | 需要合理建模（分区/分桶/索引/预聚合），避免把 OLTP 用法硬套                                                 | **高**：能同时覆盖事实表与状态表 |
| StarRocks（可选备选）    | Primary Key upsert/delete；高并发、物化视图能力强                                               | 与 Doris 类似的建模要求；团队需评估生态/运维经验                                                            | **高**：与 Doris 同级别备选      |
| ClickHouse               | 极强查询性能与写入；适合日志/事件流事实表                                                       | **UPDATE/DELETE（mutation）不适合高频**；ReplacingMergeTree 去重属于“插入多版本+后台合并”，实现与一致性复杂 | **中**：事实表很强，状态表会复杂 |
| Greenplum                | PostgreSQL 生态 + MPP；SQL 强                                                                   | 更偏数仓路线，实时导入/低延迟高并发需要较强运维与调优；成本可能更高                                         | **中**                           |
| Impala                   | Hadoop 上交互式 SQL                                                                             | 依赖 HDFS/Hive；更像查询引擎；不适合本项目“一体化实时库”                                                    | **低**                           |
| Pinot/Druid              | 实时指标分析强                                                                                  | join/更新/任意筛选能力受限；更像“指标服务”而非后台任意组合查询                                              | **低~中**（仅指标场景）          |

---

## 7. 选型结论：推荐 **Apache Doris**

综合“海量数据 + 高频筛选分页 + 状态 upsert + Go 服务化接入”的要求，**推荐选型 Apache Doris 作为本项目主数据库**。

### 7.1 关键理由（对应项目痛点）

1. **状态表的幂等更新更自然**订单/余额/风控指标等本质是“同一主键多次写入”，Doris 的 Unique Key/更新机制更贴合这类模型（比 ClickHouse mutation 更省心）。
2. **更适合做“在线后台查询”**目标是让管理后台列表/筛选在海量数据下仍保持稳定延迟与并发，而不是只做离线数仓。
3. **接入与生态成本更低**MySQL 协议与标准 SQL 生态，有利于 Go 服务端快速落地（你们后续还要写接口/SDK/权限等）。
4. **事实表 + 状态表一库覆盖，减少多数据库架构复杂度**
   在一个库内同时放“事件事实表（append）”与“最新状态表（upsert）”，再用物化视图做预聚合，比较符合你们“一个后台服务”的工程诉求。

> 备选：如果团队已经有 StarRocks 深度运维经验，也可以选 StarRocks；两者定位非常接近，最终可按团队经验与压测结果拍板。

---

## 8. 落地建议（按本项目数据形态）

### 8.1 表模型建议（避免把 OLTP 用法直接搬过来）

- **事实表（append-only）**：使用可追加写入模型（如 Doris Duplicate Key 或明细事实表），按 `dt/小时` 分区，按 `subaccount_id/market_id` 分桶。

  - 例：成交明细、充提明细、申购赎回事件、利息分配事件等。

- **状态表（upsert）**：使用 Unique Key（主键=业务唯一标识），按 `version` 覆盖更新。

  - 例：`order_latest(order_hash)`、`balance_latest(subaccount_id, denom)`、`withdraw_latest(tx_hash)` 等。
  - version 推荐用：`block_height` + `tx_index`（或单调递增的写入序号）。

- **维表**：market/token/user 映射小表可以放 Doris 同库，避免跨库 join。

### 8.2 写入链路建议（事件索引 → Doris）

- 推荐链路：**链上事件索引服务 → Kafka → Doris Routine Load**（持续消费）
  - Kafka 持久化便于重放与容灾；Routine Load 适合持续导入。
- 对状态表写入：建议由索引服务在入库前做**幂等去重/版本控制**（或在 Doris 侧用 Unique Key 覆盖）。

### 8.3 Go 后端如何接入 Doris（推荐方案）

本项目后端采用 Go 语言时，Doris 的主流接入方式是 **MySQL 协议（查询/管理）** + **HTTP 导入（Stream Load）** + **Kafka 持续导入（Routine Load）**，因此即使不依赖专门的 Doris Go SDK，也可以直接落地。

#### 8.3.1 查询/管理（推荐）：`database/sql` + MySQL Driver

- Doris 采用 MySQL 网络协议（兼容 MySQL 生态客户端）。这意味着 Go 侧可以直接用 `database/sql` + `github.com/go-sql-driver/mysql` 连接 FE 的 MySQL 端口（默认 9030）做查询与管理 SQL。
- 工程建议：
  - 以 **手写 SQL/SQL Builder** 为主（复杂筛选、动态 where、聚合查询更可控）。
  - 明确**不把 Doris 当 OLTP**：避免高频小事务式更新；尽量批量写/导入。
  - 分页建议使用 **seek/cursor 分页**（避免大 offset）。

#### 8.3.2 ORM（可用但建议“弱化”）：GORM / sqlx / sqlc

由于 Doris 兼容 MySQL 协议，原则上可以通过 MySQL driver 让 GORM 工作；但建议定位为：

- **GORM 主要用于连接管理 + 少量简单 CRUD（维表/配置表）**
- 订单/资金等核心列表查询建议仍以 **Raw SQL** 为主（GORM 的自动迁移/复杂关联/事务语义在分析型数据库里往往不匹配）

备选工具：

- `sqlx`：对 `database/sql` 的增强（结构体扫描更方便）。
- `sqlc`：从 SQL 生成类型安全的 Go 代码（适合固定查询的仓储层）。

#### 8.3.3 写入/导入（强烈推荐）：Stream Load / Routine Load

本项目的“链上事件索引 → Doris”写入建议与“查询服务”解耦：

- **主写入链路（推荐）**：索引服务写 Kafka → Doris **Routine Load** 持续消费（Exactly-Once 语义）
- **批量导入/回补（推荐）**：Go 服务或离线任务通过 Doris **Stream Load（HTTP）** 做大批量写入
  - 适合回放链上历史、补数据、导入离线文件等。

> 小提示：对于“状态表 upsert”，更推荐由索引层统一做版本控制（例如以 `block_height/tx_index` 作为 version），写入 Doris Unique Key 表实现幂等覆盖。

### 8.4 查询与分页建议（非常重要）

在海量数据下，**避免深分页 OFFSET**。推荐使用“游标/seek 分页”：

- 用 `(create_time, id)` 作为游标：`WHERE (create_time, id) < (?, ?)` + `ORDER BY create_time DESC, id DESC LIMIT N`
- Doris/ClickHouse 这类列式 OLAP 在大 offset 下都会变慢，seek 分页能显著改善稳定性。

### 8.5 预聚合与加速

- 对“首页趋势/统计”等聚合类查询，建议做：
  - Doris 物化视图（按日/小时聚合）
  - 或离线/定时任务落宽表（ADS 层）

---

## 9. 风险与应对

- **风险：把 OLAP 当 OLTP 用（大量小事务/高频更新）**

  - 应对：事件明细 append；状态表 upsert（批量写）；减少单行频繁 update。

- **风险：查询慢主要来自深分页与高维过滤**

  - 应对：seek 分页；合理分区/分桶；必要时给高频过滤字段加合适索引/排序键；对热查询做缓存（Redis）。

- **风险：口径一致性（链上为真源）**

  - 应对：保留原始事件事实表；状态表由事实表派生；支持重放修复；关键字段记录 block_height/tx_hash 便于追溯。

---

## 10. 参考资料（OpenWebSearch）

- Apache Doris Unique Key Table：[https://doris.apache.org/docs/3.x/table-design/data-model/unique](https://doris.apache.org/docs/3.x/table-design/data-model/unique)
- Apache Doris Data Update Overview：[https://doris.apache.org/docs/3.x/data-operate/update/update-overview](https://doris.apache.org/docs/3.x/data-operate/update/update-overview)
- Apache Doris Routine Load（Kafka）：[https://doris.apache.org/docs/3.x/data-operate/import/import-way/routine-load-manual](https://doris.apache.org/docs/3.x/data-operate/import/import-way/routine-load-manual)
- Apache Doris MySQL Protocol 连接：[https://doris.apache.org/docs/4.x/db-connect/database-connect](https://doris.apache.org/docs/4.x/db-connect/database-connect)
- Apache Doris Stream Load（HTTP 导入）：[https://doris.apache.org/docs/4.x/data-operate/import/import-way/stream-load-manual](https://doris.apache.org/docs/4.x/data-operate/import/import-way/stream-load-manual)
- ClickHouse ReplacingMergeTree：[https://clickhouse.com/docs/guides/replacing-merge-tree](https://clickhouse.com/docs/guides/replacing-merge-tree)
- ClickHouse Mutations（UPDATE/DELETE）：[https://clickhouse.com/docs/guides/developer/mutations](https://clickhouse.com/docs/guides/developer/mutations)
- StarRocks Primary Key 表变更写入（Upsert/Delete）：[https://docs.starrocks.io/docs/loading/Load_to_Primary_Key_tables/](https://docs.starrocks.io/docs/loading/Load_to_Primary_Key_tables/)
- Apache Impala Overview：[https://impala.apache.org/overview.html](https://impala.apache.org/overview.html)
- Greenplum Architecture Overview（Broadcom/VMware 文档）：[https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/admin_guide-intro-arch_overview.html](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/admin_guide-intro-arch_overview.html)
- Apache Pinot 官网：[https://pinot.apache.org/](https://pinot.apache.org/)
- Apache Druid Data Updates：[https://druid.apache.org/docs/latest/data-management/update/](https://druid.apache.org/docs/latest/data-management/update/)
