# 离线流式统计计算模块 — 架构概览

**版本：** v1.2　　**日期：** 2026-03-09

---

## 整体数据流

```mermaid
flowchart TD
    DB[("`业务数据库
    MongoDB / MySQL`")]

    DBZ["**Debezium CDC**
    监听数据库变更，封装成标准消息"]

    MQ["**RabbitMQ**
    独立队列接收 CDC 消息，与业务消费者隔离"]

    subgraph STREAM["流式统计模块"]
        direction TB
        A["**① Source**
        消费 RabbitMQ 消息
        同步归档原始消息到 PostgreSQL"]

        B["**② Normalize**
        将 Debezium JSON 解析为
        统一的 Event 结构体"]

        C["**③ Window Engine**
        按事件时间分配到窗口
        支持乱序容忍（Watermark）
        滚动窗口 / 滑动窗口"]

        D["**④ Aggregation**
        每个窗口独立计算
        COUNT / SUM / AVG / UV / P99"]

        E["**⑤ Checkpoint**
        定期将窗口计算状态存入 PostgreSQL
        服务重启后可从此恢复，无需重算"]

        A --> B --> C --> D --> E
    end

    PG_ARCHIVE[("**PostgreSQL**
    原始事件归档
    用于 Replay 和数仓 ETL")]

    ES[("**Elasticsearch**
    窗口聚合结果
    秒级实时查询 / 监控看板")]

    subgraph DW["数仓二次加工层（ClickHouse / Doris）"]
        direction TB
        ODS["**ODS 层**
        原始数据原样入库
        保留完整历史明细"]

        DWD["**DWD 层**
        清洗 + 去重
        关联用户 / 商品 / 地区等维度表"]

        DWS["**DWS 层**
        多维度聚合汇总
        按渠道 / 地区 / SKU / 天 / 周 / 月"]

        ADS["**ADS 层**
        最终应用指标
        漏斗 / 留存 / GMV / DAU / BI 报表"]

        ODS --> DWD --> DWS --> ADS
    end

    DB --> DBZ --> MQ --> A
    A -- "归档原始消息" --> PG_ARCHIVE
    D -- "实时窗口结果" --> ES
    D -- "窗口结果写入 ODS" --> ODS
    PG_ARCHIVE -- "定时 ETL 补充明细" --> ODS
```

---

## Replay（历史重算）

```mermaid
flowchart LR
    PG[("PostgreSQL
    原始事件归档")]

    RP["**Replayer**
    按时间范围从归档读取历史事件
    绕过 RabbitMQ，重新投入 Pipeline"]

    ISO["**隔离输出**
    ES：独立 index 后缀 _replay
    数仓：独立表名后缀 _replay
    不覆盖任何线上数据"]

    PG --> RP --> ISO
```

---

## 组件一览

| 组件 | 职责 |
|------|------|
| **Debezium CDC** | 监听业务数据库变更，产生标准化消息 |
| **RabbitMQ** | 消息队列，独立队列保证与业务隔离 |
| **Source** | 消费 MQ 消息，归档原始数据，投递到处理管道 |
| **Normalize** | 解析 Debezium 格式，输出统一 Event 结构 |
| **Window Engine** | 按事件时间分窗口，处理乱序，触发计算 |
| **Aggregation** | 在每个窗口内计算 COUNT / SUM / AVG / UV / P99 |
| **Checkpoint** | 定期保存计算状态到 PG，支持故障恢复 |
| **Elasticsearch** | 存储窗口结果，供实时监控 / 看板查询 |
| **PostgreSQL 归档** | 存储原始事件，同时作为 Replay 和数仓 ETL 的数据来源 |
| **数仓 ODS 层** | 原始数据贴源入库，保留完整明细 |
| **数仓 DWD 层** | 清洗去重，关联用户 / 商品 / 地区维度表 |
| **数仓 DWS 层** | 多维度聚合汇总宽表 |
| **数仓 ADS 层** | 最终业务指标，对接 BI 报表 / 对外 API |
| **Replayer** | 从归档读取历史数据，重走计算流程，结果写入隔离存储 |
