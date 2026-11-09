# Pro Scheduler Specification

## Version | v1.0 | Status | Draft → Pending review (P-Phase) | Owner | Scheduler Team

---

### 4项默认值（P-2.1）

| # | 维度 | 默认值 | 可配置范围 | 依据 |
|---|------|--------|-----------|------|
| 1 | **崩溃恢复策略** | B — 标记 `failed`，等待管理员手动重试 | 仅策略B（非幂等安全优先） | 避免非幂等操作重复执行 |
| 2 | **Misfire 策略** | catch-up + **24h 过期窗口** | `[1h, 7d]` 可调 | 错过 < 24h → 立即补跑；> 24h → 跳过至下一周期 |
| 3 | **幂等性保证** | `(job_id, started_at)` 复合唯一索引 | — | `run_history` 表物理约束，不可关闭 |
| 4 | **超时保护** | **30分钟**（1800秒） | `[60s, 240min]` = `[1, 14400s]` | 超时至 kill 子进程树 + `timed_out` 状态 |

### 状态机（P-2.2）

```
                    ┌──────────┐
         start →   │  pending  │
                   └────┬─────┘
                        │ schedule due or manual trigger
                        ▼
                   ┌──────────┐     timeout →     ┌────────────┐
                  │ running  │ ──────────────────→│ timed_out  │
                   └────┬─────┘                    └─────┬──────┘
                        │                                │
              success   │    failure                      │ retry eligible
                        ▼                                ▼
                   ┌──────────┐                     ┌──────────┐
                  │succeeded  │◄──────── retry →   │  failed  │
                   └──────────┘                     └────┬─────┘
                                                        │ max retries reached (≥3)
                                                        ▼
                                                  ┌────────────┐
                                                 │dead_letter  │
                                                  └────┬───────┘
                                                       │ 7 days prune → DROP
```

### 重试策略（P-2.3）

- **算法**: 指数退避 + jitter：`delay = min(2 × 2^retry, 600s) ± 15% jitter`
- **最大重试次数**: 3 (可配置 `max_retries: 0..=10`)
- **死信队列表**: `dead_letter_queue(job_id, task_type, schedule, error_message, retry_count, created_at, expires_at)`
- **TTL**: 7 天自动清理，`prune_dead_letter_queue()` 每次 tick 调用

### SQL Schema

```sql
-- cron_jobs extensions:
ALTER TABLE cron_jobs ADD COLUMN job_state TEXT DEFAULT 'pending';
ALTER TABLE cron_jobs ADD COLUMN timeout_seconds INTEGER DEFAULT 1800;

-- run_history idempotency index (P-2.1#3):
CREATE UNIQUE INDEX IF NOT EXISTS idx_run_history_idempotent
    ON run_history(job_id, started_at);

-- dead_letter_queue (P-2.3):
CREATE TABLE IF NOT EXISTS dead_letter_queue (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    job_id TEXT NOT NULL,
    task_type TEXT NOT NULL,
    schedule TEXT NOT NULL,
    error_message TEXT NOT NULL,
    retry_count INTEGER NOT NULL DEFAULT 0,
    created_at TEXT NOT NULL,
    expires_at TEXT NOT NULL
);
```

### Tick Loop (后台定时执行器)

| 参数 | 值 |
|------|-----|
| 轮询间隔 | 60s |
| 超时守护 (P-2.1#4) | `tokio::time::timeout` wrap in `execute_job` |
| 崩溃恢复 (P-2.1#1) | 启动时调用 `recover_crashed_jobs()` |
| 死信清理 (P-2.3) | 每 N=60 tick (≈1h) 调用 `prune_dead_letter_queue()` |

### 接口契约

| Tauri Command | 描述 | 返回值 |
|--------------|------|--------|
| `scheduler_list_jobs()` | 列出所有定时任务 | `Vec<CronJob>` |
| `scheduler_create_job(...)` | 创建新任务 | `CronJob` |
| `scheduler_delete_job(id)` | 删除任务 | `bool` |
| `scheduler_toggle_job(id, enabled)` | 启用/禁用 | `bool` |
| `scheduler_retry_job(id)` **(NEW)** | 重试失败任务 | `bool` |

### 审计日志事件

| Event Key | 触发条件 | Payload Fields |
|-----------|---------|----------------|
| `misfire.catchup` | 错过时间 < 24h 补跑 | `job_id`, `missed_seconds` |
| `misfire.skip` | 错过 > 24h 跳过 | `job_id`, `exceeded_by` |
| `crash_recovery` | 进程恢复 + running → failed | `recovered_count` |
| `dead_letter.enqueued` | 重试耗尽入死信队列 | `job_id`, `retries_exhausted`, `error_message` |
| `dead_letter.pruned` | TTL过期清理 | `pruned_count` |
