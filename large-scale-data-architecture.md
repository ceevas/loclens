# 大规模数据处理架构分析

> 项目路径: `/home/dd/dataana/data-analyzer`
> 技术栈: Tauri 2.x + Rust (Polars/DuckDB) + Vue 3

---

## 1. 当前架构评估

### 1.1 架构概述

```
┌─────────────────────────────────┐
│       Tauri Desktop App         │
│  ┌─────────┐    ┌────────────┐  │
│  │ Vue 3   │◄──►│ Rust Core  │  │
│  │ Frontend│    │ Polars/    │  │
│  │         │    │ DuckDB     │  │
│  └─────────┘    └────────────┘  │
└─────────────────────────────────┘
        │ IPC (本地进程通信)
        ▼
   ┌──────────┐
   │ 本地文件  │
   │ 数据库   │
   └──────────┘
```

**核心依赖:**

| 依赖 | 版本 | 用途 |
|------|------|------|
| polars | 0.44 | DataFrame 计算引擎 (lazy + streaming + parquet + sql) |
| duckdb | 1.1.1 | 嵌入式 OLAP 查询 |
| sqlx | 0.8 | 数据库连接 (SQLite/MySQL/PostgreSQL) |
| tokio | 1.x | 异步运行时 |

### 1.2 当前瓶颈

**内存硬上限:**
- 默认 `max_memory_mb: 2048` (2GB)
- 所有 DataFrame 缓存在 `AppState` HashMap 中驻留内存
- 查询结果强制 `.collect()` 全量物化

**查询模式问题:**
- `polars.rs`: 所有 filter/group_by/join 最终都 collect 到内存
- `streaming_query`: 虽然名义上分页，但底层仍是全量 collect 后 `.slice()`
- `csv.rs`: `count_rows()` 全量读取整个 CSV 文件

**数据源限制:**
- 仅支持本地文件 (CSV/JSON/Parquet/Excel/SAS)
- 数据库查询结果全量拉取到本地内存
- 无远程数据源、无对象存储集成

---

## 2. 局域网分布式架构设计

### 2.1 总体架构

```
┌──────────────────────────────────────┐
│       Tauri Desktop (Client)         │
│  Vue 3 Frontend ─ IPC ─ Rust Commands│
└───────────────┬──────────────────────┘
                │ HTTP/gRPC (局域网 192.168.1.x)
                ▼
┌──────────────────────────────────────┐
│          Coordinator (192.168.1.12)   │
│  ┌──────────┐  ┌──────────┐         │
│  │ Query    │→ │ Task     │         │
│  │ Planner  │  │ Scheduler│         │
│  └──────────┘  └──────────┘         │
│  ┌──────────┐  ┌──────────┐         │
│  │ Result   │  │ Node     │         │
│  │Aggregator│  │ Manager  │         │
│  └──────────┘  └──────────┘         │
│                                      │
│  共享存储: NFS / MinIO / Local FS    │
└──────┬──────────┬──────────┬─────────┘
       │ gRPC     │ gRPC     │ gRPC
  ┌────▼────┐ ┌───▼────┐ ┌───▼────┐
  │Worker-1 │ │ Worker │ │ Worker │
  │(本机0) │ │   2    │ │   N    │
  │62GB/40C│ │        │ │        │
  └────────┘ └────────┘ └────────┘
       │          │          │
  ┌────▼────┐ ┌───▼────┐ ┌───▼────┐
  │Polars   │ │Polars  │ │Polars  │
  │partition│ │partition│ │partition│
  │   #0   │ │   #1   │ │   #N   │
  └────────┘ └────────┘ └────────┘
```

### 2.2 分层设计

**Layer 1 - 存储层:**
- 数据分片存储在局域网共享存储 (NFS/SMB/MinIO)
- 每个 Worker 本地缓存负责的分片
- Parquet 格式 + 分区 (按日期/类别)，支持谓词下推

**Layer 2 - 协调层 (Coordinator):**
- Rust + tonic (gRPC) 实现
- 功能：查询规划 → 任务拆分 → 分发执行 → 结果聚合 → 返回客户端

**Layer 3 - 计算层 (Worker):**
- 每个 Worker 是 Polars LazyFrame 执行器
- 通过 gRPC 接收任务，返回 Arrow IPC 格式中间结果
- 局域网内 gRPC/HTTP2 通信

**Layer 4 - 展示层 (Tauri Desktop):**
- 前端 Vue 3 + ECharts 基本不变
- 后端命令层改为异步查询 Coordinator
- 大数据查询用流式推送 (WebSocket/SSE)

### 2.3 数据分片策略

```
共享存储/
├── parquet/
│   ├── table=sales/
│   │   ├── year=2024/month=01/part-0.parquet  → Worker-1
│   │   ├── year=2024/month=02/part-0.parquet  → Worker-1
│   │   ├── year=2024/month=03/part-0.parquet  → Worker-2
│   │   └── year=2024/month=04/part-0.parquet  → Worker-2
│   ├── table=users/
│   │   ├── hash=0/part-0.parquet               → Worker-1
│   │   ├── hash=1/part-0.parquet               → Worker-2
│   │   └── hash=2/part-0.parquet               → Worker-3
│   └── table=logs/
│       ├── day=2024-01-01/part-0.parquet       → round-robin
│       └── day=2024-01-02/part-0.parquet
└── metadata/
    └── catalog.json    # 表名 → 分区 → 分片映射
```

### 2.4 查询执行流程

```
1. 前端发 SQL: "SELECT region, SUM(sales) FROM orders GROUP BY region"

2. Coordinator 解析:
   - 识别 table=orders, 共 4 个分片分布在 2 个 Worker
   - 操作可并行: SUM 支持 partial + global merge

3. 下发 Task 到每个 Worker:
   Worker-1: SUM(sales) GROUP BY region  (只读 month=01,02 分片)
   Worker-2: SUM(sales) GROUP BY region  (只读 month=03,04 分片)

4. Worker 返回部分聚合结果 (Arrow IPC):
   Worker-1: [(华东, 100w), (华南, 80w), ...]
   Worker-2: [(华东, 150w), (华南, 90w), ...]

5. Coordinator 做全局聚合:
   [(华东, 250w), (华南, 170w), ...]

6. 返回给前端 ECharts 渲染
```

### 2.5 模块结构

```
distributed/
├── coordinator/
│   ├── Cargo.toml
│   └── src/
│       ├── main.rs           # HTTP 服务入口
│       ├── query_planner.rs  # SQL → DistributedPlan
│       ├── task_runner.rs    # MapReduce 调度
│       ├── result_merge.rs   # 结果合并 (repartition/shuffle)
│       ├── nodes.rs          # Worker 注册/心跳/健康检查
│       └── proto/
│           └── task.proto    # gRPC 定义
├── worker/
│   ├── Cargo.toml
│   └── src/
│       ├── main.rs           # gRPC 服务入口
│       ├── executor.rs       # 执行 Polars 操作
│       ├── partition.rs      # 本地数据分片管理
│       └── proto/
│           └── task.proto    # 复用 Coordinator 的 proto
└── tauri/
    └── commands/
        └── distributed.rs    # Tauri 客户端命令
```

### 2.6 Proto 协议定义

```protobuf
// 查询请求
message QueryRequest {
  string sql = 1;
  repeated string source_ids = 2;
  int32 limit = 3;
}

// 分发到 Worker 的任务
message Task {
  string task_id = 1;
  string partition_id = 2;
  repeated ScanOperator scans = 3;
  repeated FilterOperator filters = 4;
  repeated ProjectOperator projections = 5;
  AggregationPlan aggregation = 6;
}

// Worker 返回 Arrow IPC 批次
message TaskResult {
  string task_id = 1;
  bytes arrow_batch = 2;   // Arrow IPC 二进制
  int64 row_count = 3;
  int64 elapsed_ms = 4;
  Status status = 5;
}

enum Status { OK, ERROR, TIMEOUT }
```

---

## 3. 数据处理量级评估

### 3.1 当前机器配置

```
主机: 192.168.1.12
内存: 62 GB (可用 ~47 GB)
CPU: 40 核
磁盘: 1.1 TB NVMe (剩余 133 GB)
网络: 千兆局域网
```

### 3.2 不同节点规模的处理能力

**场景 A: 小型局域网 (3-5 台机器，千兆网卡)**

| 指标 | 数值 |
|------|------|
| 总内存 | 186 - 310 GB |
| 总 CPU 核心 | 120 - 200 |
| 并行读带宽 | 5 × 500MB/s = 2.5 GB/s |
| 聚合网络带宽 | 125 MB/s（千兆瓶颈）|

| 数据量 | 全表扫描 | 聚合 Shuffle |
|--------|---------|-------------|
| 1 TB | 6.7 分钟 | 1.3 分钟 (10GB) |
| 5 TB | 33 分钟 | 6.7 分钟 (50GB) |

**场景 B: 中型局域网 (10-20 台，千兆网卡)**

| 指标 | 数值 |
|------|------|
| 总内存 | 620 GB - 1.2 TB |
| 总 CPU 核心 | 400 - 800 |
| 聚合网络带宽 | 125 MB/s（仍为千兆瓶颈）|

| 数据量 | 全表扫描 | 聚合 Shuffle |
|--------|---------|-------------|
| 10 TB | 33 分钟 | 13 分钟 (500GB) |
| 20 TB | 33 分钟 | 27 分钟 (1TB) |

**场景 C: 万兆网络升级**

| 对比项 | 千兆 | 万兆 |
|--------|------|------|
| Shuffle 带宽 | 125 MB/s | 1250 MB/s (10倍) |
| 500GB 聚合耗时 | 67 分钟 | 6.7 分钟 |
| 5TB 聚合耗时 | 11 小时 | 67 分钟 |

### 3.3 数据量级对照表

| 数据量 | 千兆 3-5 节点 | 千兆 10-20 节点 | 万兆 10-20 节点 | PB 级参考 |
|--------|-------------|-----------------|-----------------|----------|
| 100 GB | 秒级 | 秒级 | 秒级 | - |
| 1 TB | 分钟级 | 秒级 | 秒级 | - |
| 10 TB | 不可行 | 分钟级 | 分钟级 | 1/100 |
| 100 TB | 不可行 | 勉强 | 可行 | 1/10 |
| 1 PB | 不可行 | 不可行 | 不可行 | 需 200+ 节点 |
| 10 PB | 不可行 | 不可行 | 不可行 | 云原生/CDH |

### 3.4 估算公式

```
全表扫描耗时 = 总数据量(GiB) / (节点数 × 单盘读速(MiB/s))

聚合 Shuffle 耗时 = 结果集大小(GiB) / 网络带宽(MiB/s)

内存可用量 = Σ(各节点可用内存) × 0.7
              ↑ 预留 30% 给 OS 和网络缓冲区

单节点最大分区 = 可用内存 × Parquet 压缩率(3-5倍)
```

---

## 4. 核心瓶颈因素分析

### 4.1 瓶颈排序（从高到低）

```
第 1 名: 内存 (单节点生死线)
第 2 名: 网络带宽 (Shuffle 效率天花板)
第 3 名: 磁盘 I/O (并行读取后可扩展)
第 4 名: CPU (40 核 + SIMD 充分余量)
```

### 4.2 磁盘 I/O 分析

**特性: 分布式下可线性扩展**

| 介质 | 单盘读取速度 | 1TB 扫描时间 (单节点) | 10TB 扫描时间 (20 节点) |
|------|-------------|----------------------|------------------------|
| HDD | ~150 MB/s | 1 小时 54 分 | 57 分钟 |
| SATA SSD | ~500 MB/s | 35 分钟 | 17 分钟 |
| NVMe SSD | ~2000 MB/s | 8.5 分钟 | 4.2 分钟 |

**结论:** N 个 Worker 并行读盘，I/O 不是瓶颈。

### 4.3 网络带宽分析

**Shuffle 场景 (Workers 回传结果给 Coordinator):**

| 网络类型 | 带宽 | 1GB 传输 | 10GB 传输 | 100GB 传输 |
|---------|------|---------|----------|----------|
| 百兆 | 12.5 MB/s | 80 秒 | 13 分钟 | 2 小时 |
| 千兆 | 125 MB/s | 8 秒 | 1.3 分钟 | 13 分钟 |
| 万兆 | 1250 MB/s | 0.8 秒 | 8 秒 | 1.3 分钟 |

**真实查询示例:**

```sql
-- 按地区统计销售额
SELECT region, SUM(sales) FROM orders GROUP BY region

-- 3 个 Worker, 每个返回局部聚合结果
Worker-1: 100MB
Worker-2: 120MB
Worker-3: 80MB
总计: 300MB
```

| 网络 | 聚合 300MB 耗时 | 可接受返回集上限 |
|------|---------------|----------------|
| 百兆 | 4 分钟 | ~300 MB |
| 千兆 | 24 秒 | ~3 GB |
| 万兆 | 2.4 秒 | ~30 GB |

**结论:** 千兆网络下，Shuffle 数据超过几个 GB，查询明显变慢。升级万兆可提升 10 倍。

### 4.4 内存分析

**生死线特性: OOM = 直接崩溃，没有任何商量余地**

| 操作 | 内存公式 | 10GB 数据所需内存 |
|------|---------|----------------|
| 全表扫描 | 原始大小 × 压缩率倒数 | 10-20 GB |
| GROUP BY | 结果集大小 (远小于原始) | 100MB-1GB |
| JOIN | 左表+右表+哈希表 (≈ 2-3x) | 20-30 GB |
| 排序 | 数据大小 × 2 | 20 GB |
| 窗口函数 | 窗口大小 × 列数 | 可能极大 |

**本机实际可用内存:**

```
62 GB 物理内存
- OS 预留: 4-6 GB
- Docker 容器: 5-8 GB
- Tauri 前端: 1-2 GB
- Coordinator: 2-4 GB
───────────────────────
Polars Worker 可用: ≈ 40-45 GB
```

**关键约束:**
- 可用 40GB 内存，配合 Parquet 5-10 倍压缩率
- 可加载 200-400GB Parquet 数据的单分区
- JOIN 操作两表合计不能超过 40GB
- **内存不可横向扩展**，单分区大小受限于单节点内存

### 4.5 CPU 分析

**基本不是瓶颈:**

```
聚合 1000 万行:     ~0.5 秒
过滤 + 投影:        ~0.2 秒
排序 500 万行:      ~1 秒
```

Polars 本身已充分利用 SIMD 和多线程，40 核的计算能力远超 IO 和网络瓶颈。CPU 成为瓶颈的唯一场景:
- 复杂多表 JOIN
- 大量正则表达式匹配
- 自定义 UDF 函数

---

## 5. 实现路线图

### Phase 1: 基础框架 (1-2 周)

- [ ] Proto 定义 + gRPC codegen
- [ ] Worker 服务：注册、心跳、执行 scan + filter
- [ ] Coordinator 服务：节点管理、任务下发、结果收集
- [ ] Tauri 命令层对接 Coordinator

### Phase 2: SQL 查询支持 (2-3 周)

- [ ] Coordinator 集成 DuckDB 做 query planning
- [ ] 支持 GROUP BY / JOIN / ORDER BY 分发
- [ ] Arrow IPC 序列化/反序列化
- [ ] 结果合并 (partial + global aggregation)

### Phase 3: 运维能力 (1-2 周)

- [ ] Docker Compose 一键部署
- [ ] Web UI 查看集群状态、数据分布
- [ ] 自动分片管理工具
- [ ] 负载均衡

### Phase 4: 高级特性 (可选)

- [ ] 数据自动重平衡
- [ ] 增量数据接入
- [ ] 安全认证 (API Key)
- [ ] Checkpoint 容错恢复

---

## 6. 成本估算

| 阶段 | 节点 | 可处理量级 | 硬件成本 |
|------|------|-----------|---------|
| 单机 | 1 台 | 5-10 GB | 已有 |
| 双节点 | 2 台 | 20-50 GB | ~5,000 元 |
| 小型集群 | 5 台 (千兆) | 200-500 GB | ~20,000 元 |
| 中型集群 | 5 台 (万兆) | 1-5 TB | ~50,000 元 |
| 大型集群 | 200+ 台 | PB 级 | ~500,000+ 元 |

如果数据真正达到 PB 级且不需要自建硬件，直接上云 (阿里云 MaxCompute / AWS Athena) 更划算——按查询付费，单次 PB 级扫描约几元钱。

---

## 附录

### A. 当前项目关键文件

| 文件 | 说明 |
|------|------|
| `src-tauri/Cargo.toml` | Polars 0.44, DuckDB 1.1.1, sqlx 0.8 |
| `src/config.rs` | max_memory_mb 默认 2048 MB |
| `src/state.rs` | AppState 存储所有 DataFrame (内存) |
| `src/commands/polars.rs` | 查询命令，全量 collect |
| `src/commands/stream.rs` | 本地流式管道，非分布式 |
| `src/engine/io/parquet.rs` | Parquet 读取，支持列选择/谓词下推 |
| `src/engine/io/csv.rs` | CSV 读取，count_rows 全量加载 |
| `src/commands/memory_monitor.rs` | 内存监控，100MB 增长阈值 |

### B. 优化优先级

```
高优先级 (直接提升单机能力):
  1. 所有 .collect() 改为 .collect_streaming()
  2. 启用 Polars disk spilling
  3. CSV count_rows 改为基于文件行数估算
  4. 配置 max_memory_mb 合理值 (如 32768)

中优先级 (分布式基础):
  5. 增加远程数据源 (S3/Trino/ClickHouse)
  6. 虚拟表抽象层
  7. 查询结果强制限流

高成本 (架构重构):
  8. Coordinator + Worker gRPC 服务
  9. 数据分片与目录管理
  10. Shuffle 引擎
```
