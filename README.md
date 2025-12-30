## biya-dex-manager

DEX 管理后台后端工程骨架（单仓库多进程：`indexer` / `consumer` / `api`）。

### 目录结构

- **`docs/`**：技术方案、PRD、原型、链上事件梳理等文档
- **`cmd/indexer/`**：采集服务（扫链/拉取 Query → 事件解码与归一化 → 写入 Kafka）
- **`cmd/consumer/`**：消费服务（消费 Kafka → 幂等处理 → 落库 MySQL/MongoDB）
- **`cmd/api/`**：API 服务（Gin HTTP/JSON 对外接口 + 内置后台任务执行器，如导出/缓存刷新）
- **`internal/`**：各进程的内部实现代码（后续补充；当前仅目录占位）

### 说明

- **当前状态**：仅初始化工程目录与说明文件，不包含任何业务实现代码（不生成 `.go` 业务文件）。
- **命名对齐**：进程命名与技术方案文档 4.1 保持一致：`cmd/indexer`、`cmd/consumer`、`cmd/api`。


