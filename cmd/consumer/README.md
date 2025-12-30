## consumer（Kafka → MySQL/MongoDB）

### 做什么

- **消费 Kafka**：按事实域订阅/消费 Topic（market/order/trade/balance/leverage）。
- **幂等落库**：
  - **MySQL**：主键聚合型表用 `INSERT ... ON DUPLICATE KEY UPDATE`。
  - **MongoDB**：明细集合用 `updateOne(..., upsert=true)`。
- **防倒灌**：用 `block_height`（或等价 version）阻止旧消息覆盖新状态。
- **异常隔离**：支持 DLQ、重放/补数据（replay/backfill）等运维手段（后续实现）。

### 不做什么

- **不提供 HTTP**：不对前端暴露接口。
- **不扫链**：只处理来自 Kafka 的消息。


