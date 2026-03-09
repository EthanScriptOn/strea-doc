# 离线流式统计计算模块 — 技术设计方案

**版本：** v1.0
**日期：** 2026-03-09
**状态：** 待评审

---

## 一、背景与目标

当前系统通过 Debezium CDC 将数据库变更事件投递至 RabbitMQ，但缺乏对这些事件的实时统计能力。本方案设计一套**离线流式统计计算模块**，在不影响主链路的前提下，对 CDC 事件进行窗口聚合计算，并将结果写入 Elasticsearch，支撑后续的数据分析与监控需求。

**核心目标：**

- 对 Debezium CDC 事件进行实时窗口聚合（COUNT / SUM / AVG / UV / Percentile）
- 支持滚动窗口（Tumbling）和滑动窗口（Sliding）并行运行
- 支持乱序事件处理（Watermark 机制）
- 支持故障重启后从 Checkpoint 恢复，避免重算
- 支持历史数据 Replay，不影响线上统计结果
- 模块完全解耦，通过配置开关控制启停

---

## 二、整体架构

### 主链路（在线实时）

```mermaid
flowchart TD
    DB[("`**业务数据库**
    MongoDB / MySQL`")]
    DBZ["Debezium CDC
    ─────────────
    监听数据库 binlog / oplog
    将变更封装为标准 JSON 消息"]
    MQ["**RabbitMQ**
    ─────────────
    独立 Exchange + Queue
    Consumer Tag 隔离，不影响其他消费者"]

    subgraph SRC["① Source　　rabbitmq.go"]
        S1["消费消息，自动 ACK
        断线后按 reconnect-delay 重连
        prefetch-count 控制预取量"]
    end

    PG_RAW[("**PostgreSQL**
    stream_raw_events
    ─────────────
    原始消息归档
    支持按时间范围查询重放")]

    subgraph NORM["② Normalize　　debezium.go"]
        N1["解析 Debezium JSON Envelope
        提取 op / event_time / before / after
        输出统一 Event Struct"]
    end

    subgraph WIN["③ Window Engine　　window/engine.go"]
        W1["推进 Watermark
        容忍乱序延迟 watermark-lag=10s
        超时事件丢弃并记录日志"]
        W2["分配事件到窗口
        Tumbling（不重叠）
        Sliding（滑动重叠）
        多窗口并行运行"]
        W3["触发 Flush
        窗口到期 or Watermark 超过窗口终点"]
        W1 --> W2 --> W3
    end

    subgraph AGG["④ Aggregation　　agg/registry.go"]
        A1["每个窗口独立计算
        COUNT / SUM / AVG
        MIN / MAX / UV / P50·P95·P99"]
    end

    subgraph CP["⑤ Checkpoint　　checkpoint/postgres.go"]
        C1["每 30s 或窗口关闭时
        序列化聚合状态写入 PG
        重启后从此恢复，无需重算"]
    end

    subgraph SINK["⑥ Sink　　sink/es.go"]
        E1["攒批写入 Elasticsearch
        flush-batch=100 条
        或每 flush-interval=5s 强制刷写"]
    end

    PG_CK[("**PostgreSQL**
    stream_window_checkpoints
    ─────────────
    窗口状态快照
    含聚合器序列化状态")]
    ES[("**Elasticsearch**
    stream_stats_{pipeline}
    ─────────────
    最终统计结果
    供查询 / 监控使用")]

    DB --> DBZ --> MQ --> S1
    S1 -- "归档原始消息 Raw bytes" --> PG_RAW
    S1 -- "投递到 Pipeline Channel" --> N1
    N1 --> W1
    W3 --> A1
    A1 -- "定期保存窗口状态" --> C1
    A1 -- "窗口结果 WindowResult" --> E1
    C1 --> PG_CK
    E1 --> ES
```

### Replay 链路（历史重算，与线上完全隔离）

```mermaid
flowchart LR
    PG_RAW[("PostgreSQL
    stream_raw_events")]

    subgraph RP["Replay　　replay/replayer.go"]
        R1["按 From/To 时间范围
        分批查询归档事件
        BatchSize 控制每批数量"]
        R2["重走 Normalize
        → Window → Agg
        Pipeline 带 replay 标记"]
        R1 --> R2
    end

    PG_CK_R[("PostgreSQL
    stream_window_checkpoints
    ─────────────
    window_id 加 replay_ 前缀
    不覆盖线上 Checkpoint")]
    ES_R[("Elasticsearch
    stream_stats_{pipeline}_replay
    ─────────────
    独立 index，不污染线上数据
    验证完成后可人工切换")]

    PG_RAW --> R1
    R2 -- "隔离写入" --> PG_CK_R
    R2 -- "隔离写入" --> ES_R
```

---

## 三、目录结构

```
internal/stream/
├── stream.go             # Manager：Start / Stop 入口
├── config.go             # 所有配置结构体
├── event.go              # Event / WindowResult 核心类型
├── pipeline.go           # Pipeline 编排（source → stages → sink）
├── source/
│   └── rabbitmq.go       # RabbitMQ 消费者 + 原始事件归档
├── normalize/
│   └── debezium.go       # Debezium CDC JSON → Event 标准化
├── window/
│   ├── window.go         # Window / WindowState 接口定义
│   ├── engine.go         # 窗口生命周期管理（创建 / flush / 关闭）
│   ├── tumbling.go       # 滚动窗口实现
│   └── sliding.go        # 滑动窗口实现
├── agg/
│   ├── agg.go            # Aggregator 接口
│   ├── registry.go       # 聚合器插件注册表
│   └── builtin.go        # COUNT / SUM / AVG / MIN / MAX / UV / Percentile
├── checkpoint/
│   └── postgres.go       # 窗口状态持久化到 PostgreSQL
├── replay/
│   └── replayer.go       # 从 PG 归档重放历史事件
└── sink/
    └── es.go             # 写入 Elasticsearch
```

---

## 四、核心数据结构

### 4.1 统一事件结构（Event）

```go
type Event struct {
    ID        string
    Source    string         // Debezium topic，如 mongo.orders
    Op        string         // insert / update / delete / read
    EventTime time.Time      // 事件时间（用于窗口分配）
    Before    map[string]any
    After     map[string]any
    Raw       []byte         // 原始消息（用于归档）
}
```

### 4.2 聚合器接口（Aggregator）

采用插件化设计，每种聚合类型实现统一接口，便于扩展：

```go
type Aggregator interface {
    Name()           string
    Add(e Event)     error
    Result()         any
    MarshalState()   ([]byte, error)  // 序列化，用于 Checkpoint
    UnmarshalState([]byte) error      // 反序列化，用于恢复
    Reset()
    Clone()          Aggregator
}
```

### 4.3 窗口状态（WindowState）

```go
type WindowState struct {
    WindowID     string
    PipelineName string
    WindowStart  time.Time
    WindowEnd    time.Time
    AggStates    map[string]json.RawMessage  // aggregator name → 序列化状态
    EventCount   int64
    Flushed      bool
}
```

### 4.4 函数式 Pipeline Stage

```go
type Stage    func(in <-chan Event) <-chan Event
type Pipeline struct {
    source Source
    stages []Stage
    sink   Sink
}
```

---

## 五、时间窗口策略

| 类型 | 说明 | 配置示例 |
|------|------|---------|
| **Tumbling（滚动）** | 固定不重叠，`[0,1min),[1,2min)` | `size: 1m` |
| **Sliding（滑动）** | 窗口重叠，`[0,5min),[1,6min)` | `size: 5m, slide: 1m` |
| **多窗口并行** | 同一 Pipeline 可同时配置多个窗口 | 见 YAML 配置 |

**乱序事件处理：** 使用 Watermark 机制，配置最大容忍延迟（`watermark-lag`），超过水位的延迟事件会被丢弃并记录日志。

---

## 六、内置聚合器

| 类型 | 说明 | 状态可序列化 |
|------|------|:----------:|
| `count` | 事件总数 | ✓ |
| `sum` | 数值字段求和 | ✓ |
| `avg` | 均值（保存 sum + count） | ✓ |
| `min` / `max` | 最小 / 最大值 | ✓ |
| `uv` | 去重计数（精确 map，可扩展为 HyperLogLog） | ✓ |
| `percentile` | P50 / P95 / P99（Reservoir Sampling） | ✓ |

---

## 七、数据库表结构

### 7.1 原始事件归档（支持 Replay）

```sql
CREATE TABLE stream_raw_events (
    id          BIGSERIAL PRIMARY KEY,
    event_id    TEXT NOT NULL UNIQUE,
    source      TEXT NOT NULL,
    op          TEXT NOT NULL,
    event_time  TIMESTAMPTZ NOT NULL,
    payload     JSONB NOT NULL,
    archived_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX ON stream_raw_events (event_time);
CREATE INDEX ON stream_raw_events (source, event_time);
```

### 7.2 窗口状态 Checkpoint

```sql
CREATE TABLE stream_window_checkpoints (
    window_id     TEXT PRIMARY KEY,
    pipeline_name TEXT NOT NULL,
    window_start  TIMESTAMPTZ NOT NULL,
    window_end    TIMESTAMPTZ NOT NULL,
    agg_states    JSONB NOT NULL,
    event_count   BIGINT DEFAULT 0,
    flushed       BOOLEAN DEFAULT FALSE,
    updated_at    TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 八、配置示例（YAML）

```yaml
stream:
  enabled: false
  rabbitmq:
    url: "amqp://guest:guest@localhost:5672/"
    exchange: "debezium.events"
    queue: "stream.offline.stats"
    consumer-tag: "offline-stream-consumer"
    prefetch-count: 100
    reconnect-delay: "5s"
  window:
    watermark-lag: "10s"    # 容忍的最大事件乱序延迟
    tick-interval: "1s"     # 窗口引擎检查频率
  pipelines:
    - name: "order_stats"
      source-filter: "mongo.orders"
      windows:
        - type: tumbling
          size: "1m"
        - type: tumbling
          size: "1h"
        - type: sliding
          size: "5m"
          slide: "1m"
      aggregators:
        - type: count
        - type: sum
          field: "amount"
        - type: avg
          field: "amount"
        - type: uv
          field: "user_id"
        - type: percentile
          field: "latency_ms"
          percentiles: [50, 95, 99]
  checkpoint:
    enabled: true
    interval: "30s"
  sink:
    es:
      addresses: ["http://localhost:9200"]
      index-prefix: "stream_stats"    # 最终写 stream_stats_order_stats
      flush-batch: 100
      flush-interval: "5s"
```

---

## 九、Replay 机制

历史重算完全绕过 RabbitMQ，直接从 PostgreSQL 归档读取，与线上统计**完全隔离**：

```
流程：
1. 按 event_time 范围从 stream_raw_events 分批查询历史事件
2. 重新 normalize → 投入独立 replay pipeline（带 replay 标记）
3. Checkpoint 写入带 replay_ 前缀的 window_id，不覆盖线上数据
4. 结果写入带 _replay 后缀的 ES index（如 stream_stats_order_stats_replay）
```

Replay 参数：

```go
type ReplayOptions struct {
    PipelineName string
    From         time.Time
    To           time.Time
    BatchSize    int
}
```

---

## 十、关键设计决策

| 问题 | 决策 | 原因 |
|------|------|------|
| **乱序事件** | Watermark 机制 | Debezium / 网络延迟会导致事件乱序到达 |
| **背压控制** | 有界 channel buffer + 丢弃策略 | 统计数据允许少量丢失，绝不阻塞主链路 |
| **UV 精度** | 默认精确 map，大规模可换 HyperLogLog | 先保证正确性，再按需优化 |
| **Checkpoint** | 定时 + 窗口关闭时强制保存 | 重启可恢复，避免全量重算 |
| **Replay 隔离** | 独立 index 前缀 + 独立 window_id | 不覆盖线上统计结果，可安全验证 |
| **模块解耦** | `stream.enabled: false` 默认关闭 | 不影响现有功能，按需启用 |

---

## 十一、新增外部依赖

| 依赖包 | 版本 | 用途 |
|--------|------|------|
| `github.com/rabbitmq/amqp091-go` | v1.10.0 | RabbitMQ 消费者 |
| `github.com/elastic/go-elasticsearch/v8` | v8.14.0 | 写入 Elasticsearch |

项目已有 `github.com/jackc/pgx/v5`，无需额外添加 PostgreSQL 驱动。

---

## 十二、实施计划

| 阶段 | 内容 | 优先级 |
|------|------|:------:|
| **P0** | `event.go` + `config.go` — 基础类型定义 | 高 |
| **P0** | `source/rabbitmq.go` — 消费 + 归档 | 高 |
| **P0** | `normalize/debezium.go` — 事件标准化 | 高 |
| **P1** | `agg/` — 聚合器（count / sum / avg 优先） | 高 |
| **P1** | `window/engine.go` — 窗口引擎（核心） | 高 |
| **P2** | `checkpoint/postgres.go` — 状态持久化 | 中 |
| **P2** | `sink/es.go` — 写入 Elasticsearch | 中 |
| **P2** | `pipeline.go` + `stream.go` — 组装整体 | 中 |
| **P3** | `replay/replayer.go` — 历史重算 | 低 |

---

*文档路径：`docs/stream-stats-design.md`*
