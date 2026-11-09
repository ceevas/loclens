# loclens
LocLens is a local-first data analytics and collaboration workspace built with Rust, Tauri, and Vue. It supports CSV/Excel/Parquet import/export, statistical analysis, interactive visualization, real-time streams, API/CLI automation, and tiered Community/Pro/Team/Enterprise capabilities.

## 中文摘要

LocLens 是一个基于 **Rust + Tauri + Vue** 构建的**本地优先**数据分析与协作工作台。数据不出本机即可完成导入、清洗、统计、可视化、报告导出与自动化调度；需要协作时可升级到团队与企业形态。

核心能力：CSV / Excel / Parquet / JSON 数据导入导出 · 描述性统计与相关性分析 · 交互式图表探索 · 实时数据流处理 · CLI / REST API / 插件 SDK · 定时任务与批处理管道 · 团队协作与数据治理 · 审计与合规。

提供 **Community / Pro / Team / Enterprise** 四个分层，完整对比见 [功能矩阵](docs/feature-matrix.md)。

---

## Table of Contents

- [Why LocLens](#-why-loclens)
- [Features](#-features)
- [Design Principles](#-design-principles)
- [下载与安装](#-下载与安装)
- [快速开始](#-快速开始)
- [系统要求](#-系统要求)
- [Feature Tiers](#-feature-tiers)
- [Documentation](#-documentation)
- [版本与更新](#-版本与更新)
- [支持](#-支持)
- [License](#-license)

---

## ✨ Why LocLens

| 痛点 | LocLens 的做法 |
|------|----------------|
| 分析工具要求数据上云 | **本地优先**：数据默认留在本机，本地数据加密可选/默认开启 |
| 大文件导入崩溃、编码乱码 | **导入鲁棒性**：编码检测、分隔符推断、流式读取、错误行报告与跳过策略 |
| 分析完导出报告失真 | **导出保真度契约**：Excel 公式/图表/条件格式、PDF 矢量、PNG 分辨率明确承诺 |
| 脚本能跑，但没人能维护 | **自动化与集成**：CLI、带 OpenAPI 契约的 REST API、插件 SDK、Cron 风格调度器 |
| 团队协作靠传文件 | **Team 层**：本机多账号 + 文件锁冲突四段式解决 + 审计日志检索 |
| 企业采购卡在合规 | **Enterprise 层**：SSO/SCIM、RBAC、SIEM 集成、气隙部署、SOC 2 / ISO 27001 证据包 |

---

## 🚀 Features

### 数据分析
- 支持 **CSV / Excel / JSON / Parquet / Feather** 的读写与互转
- 描述性统计（均值、中位数、四分位等）、相关性矩阵、单变量分布、双/多变量对比
- 交互式图表探索：筛选、排序、聚合
- 优化与推荐引擎（Python Bridge）

### 数据输出与交付
- CSV / PNG / Excel（.xlsx 格式化）/ Parquet / Feather 导出
- HTML、PDF（矢量）、PPT（.pptx 自动化模板）报告生成
- 批量与定时导出、品牌水印移除、自定义 Logo 与配色

### 自动化与集成
- CLI 命令行接口、本地 REST API（OpenAPI + 版本化 + 弃用策略）
- Python / JS 插件 SDK（读写权限分级 + 签名验证）
- Cron 风格调度器：失败重试、任务持久化、崩溃恢复、错过执行补偿
- 多步 ETL 管道、Webhook 进出站（HMAC / JWT 鉴权）、邮件 / IM 推送
- 外部数据库连接：PostgreSQL / MySQL / MSSQL / Oracle / Snowflake

### 实时数据流
- 并发流：1 → 5 → 20 → 128（Community → Enterprise）
- 连接器：本地 Socket/HTTP、Kafka / MQTT / RabbitMQ、Kinesis / Pub/Sub / CDC
- 窗口计算（滚动 / 滑落 / 会话 / 全局）、持久化状态、Checkpoint
- 冷热分层存储、告警规则与多通道路由

### 协作与治理
- 本机多账号独立工作区、LAN 共享工作区（文件锁模式）
- 角色定义与细粒度 RBAC、多级审批链、团队模板库
- 数据目录 / 血缘与影响分析 / 业务术语表 / 数据质量规则（Enterprise）
- 审计日志（哈希链防篡改）、检索过滤、SIEM 集成

### 部署与安全
- 桌面安装包（Windows / macOS / Linux），支持静默安装 / MSI / PKG / GPO / SCCM / Intune
- 离线激活（12 个月租约 + 15–30 天宽限 + 到期只读降级）
- **完整气隙部署**（Enterprise）：离线安装/升级包、代理/DMZ 配置模板
- 浮动许可（License Server）、多租户隔离、集中配置分发与远程管控
- SSO（SAML / OIDC / 离线 LDAP）、SCIM 用户生命周期管理

---

## 🏗 Design Principles

LocLens 采用 **Rust 核心 + Tauri 壳层 + Vue 前端** 的三层结构：
┌──────────────────────────────────────────────┐
│ Vue 3 前端（呈现与交互） │
└───────────────────────┬──────────────────────┘
│ 安全 IPC / SSE / WebSocket
┌───────────────────────▼──────────────────────┐
│ Tauri 壳层（窗口、菜单、更新、权限） │
└───────────────────────┬──────────────────────┘
│
┌───────────────────────▼──────────────────────┐
│ Rust 核心引擎 │
│ · 数据 I/O 与格式互转 │
│ · 统计分析与可视化计算 │
│ · 实时流处理与窗口状态 │
│ · 报告导出（Excel / PDF / PPT） │
│ · 调度器、API 服务、插件运行时 │
│ · 审计、许可校验、加密 │
└───────────────────────┬──────────────────────┘
│
┌───────────────────────▼──────────────────────┐
│ 本地存储（SQLite / 嵌入式 KV / 本地文件 / 可选集中式存储）│
└──────────────────────────────────────────────┘


设计原则：

- **本地优先** — 单机即可完成完整分析闭环，无网络依赖
- **计算下沉** — 前端只负责呈现与交互，所有数据计算由 Rust 核心承担
- **能力分层通过 feature flag + license token 校验**，不使用文件大小 / 行数硬切上限
- **默认安全** — 本地加密、审计哈希链、最小权限 IPC

> 本图为高层能力示意，不代表具体实现细节。LocLens 的具体内部实现属于闭源范畴。

---

## 📥 下载与安装

所有正式版本均发布于 **[Releases](https://github.com/ceevas/loclens/releases)** 页面。

### 最新版本

前往 **[Latest Release](https://github.com/ceevas/loclens/releases/latest)** 下载对应平台的安装包：

| 平台 | 安装包 | 说明 |
|------|--------|------|
| Windows | `.msi` / `.exe` | 支持静默安装与 GPO / SCCM / Intune 推送（Team 及以上） |
| macOS | `.dmg` / `.pkg` | 支持 MDM 分发 |
| Linux | `.AppImage` / `.deb` / `.rpm` | 桌面安装 |

### 安装步骤

1. 从 Releases 下载对应平台的安装包
2. 运行安装程序，按向导完成安装
3. 首次启动后按提示**导入许可证**（Community 版无需激活）
4. 完成初始配置向导（数据目录、语言、加密选项）

### 校验安装包

每个 Release 附带 `SHA256SUMS` 与签名文件，请下载后校验：

```bash
# 校验哈希
sha256sum -c SHA256SUMS --ignore-missing

# 校验签名（以 GPG 为例）
gpg --verify LocLens-<version>-<platform>.sig LocLens-<version>-<platform>
