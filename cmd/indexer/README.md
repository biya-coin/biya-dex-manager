## indexer（链上采集 → Kafka）

### 做什么

- **扫链/订阅**：按高度扫描区块（或订阅），获取链上事件与必要的 Query 数据。
- **解码与归一化**：将链上事件/接口返回转换为统一的“事实域消息”（market/order/trade/balance/leverage）。
- **写入 Kafka**：按事实域分 Topic 写入，附带元信息（`block_height`、`block_time`、`tx_hash`、`event_id` 等）。
- **维护进度**：持久化扫描 checkpoint，保证可重启续扫与回放。

### 不做什么

- **不落库**：不直接写 MySQL/MongoDB。
- **不做业务聚合**：不在采集侧做页面聚合指标计算（只做规范化与分发）。


