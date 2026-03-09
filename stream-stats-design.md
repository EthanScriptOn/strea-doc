# 离线流式统计计算模块 — 架构概览

**版本：** v1.1　　**日期：** 2026-03-09

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

    subgraph OUTPUT["结果输出（双路并行）"]
        ES["**Sink → Elasticsearch**
        窗口聚合结果
        秒级实时查询 / 监控看板"]

        DW["**Sink → 数仓**
        窗口聚合结果 + 原始明细 ETL
        用于二次加工分析"]
    end

    DB --> DBZ --> MQ --> A
    A -- "归档原始消息" --> PG_ARCHIVE
    D -- "窗口结果" --> ES
    D -- "窗口结果" --> DW
    PG_ARCHIVE -- "定时 ETL" --> DW
```

---

## 数仓二次加工层

```mermaid
flowchart LR
    DW_IN["来自流式模块的
    窗口聚合结果
    +
    PostgreSQL 原始明细 ETL"]

    ODS["**ODS 层**
    原始数据原样入库
    保留完整历史明细"]

    DWD["**DWD 层**
    清洗 + 去重
    关联用户 / 商品 / 地区等维度表
    构建宽事实表"]

    DWS["**DWS 层**
    多维度聚合汇总
    按渠道 / 地区 / SKU
    按天 / 周 / 月"]

    ADS["**ADS 层**
    最终应用指标
    漏斗分析 / 用户留存
    GMV / DAU / 转化率
    BI 报表 / 对外 API"]

    DW_IN --> ODS --> DWD --> DWS --> ADS
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
| **Sink ES** | 将窗口结果写入 ES，供实时监控使用 |
| **Sink DW** | 将窗口结果写入数仓 ODS 层，供二次加工 |
| **PostgreSQL 归档** | 存储原始事件，同时作为 Replay 和数仓 ETL 的数据来源 |
| **数仓 ODS→ADS** | 原始贴源→清洗关联→多维汇总→最终指标，支持复杂分析 |
| **Replayer** | 从归档读取历史数据，重走计算流程，结果写入隔离存储 |
