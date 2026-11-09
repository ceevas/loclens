# API Surface — Error Code Registry (Frozen / Snapshot-tested)

## P-3.2: Error code stability guarantee

This table is the **single source of truth** for all LocLens backend error codes. Every Tauri command and REST endpoint MUST return one of these codes. Adding new codes requires:
1. Updating this table with a new HTTP-equivalent status + code name
2. Running `cargo test --test api_error_codes` to verify no existing mappings changed
3. Getting approval from the maintainer

---

### Error Code Table (Frozen v0.2.0)

| HTTP等价码 | 错误码标识符 | 类别 | 描述 | 示例场景 |
|-----------|-------------|------|------|---------|
| 400 | `INVALID_REQUEST` | Client | 请求参数非法或缺失 | 空 dataframe_id、无效格式字符串 |
| 400 | `INVALID_SCHEDULE` | Client | 调度表达式不合法 | `"every_42h"` 不支持 |
| 401 | `UNAUTHORIZED` | Auth | 未认证或 token 过期 | REST API 无 Bearer token |
| 403 | `LICENSE_REQUIRED` | Auth | License 版本不足 | Community 版调用 Pro 命令 |
| 403 | `RBAC_DENIED` | Auth | 角色权限不足 | 非 Admin 调用户管理接口 |
| 404 | `DATAFRAME_NOT_FOUND` | Client | DataFrame ID 不存在 | 加载已清除的数据集 |
| 404 | `DATASOURCE_NOT_FOUND` | Client | 数据源未找到 | 引用不存在的 SQL 源 |
| 409 | `JOB_ALREADY_EXISTS` | Client | Job ID 重复 | create_job 传入已有 ID |
| 409 | `IDEMPOTENCY_CONFLICT` | Client | (job_id, run_at) 已存在 | 幂等重复提交同一执行窗口 |
| 413 | `PAYLOAD_TOO_LARGE` | Client | 文件超过大小限制 | CSV > 10GB |
| 422 | `POLARS_ERROR` | Client | Polars 计算失败 | 类型不匹配、空除数 |
| 429 | `RATE_LIMITED` | RateLimit | API 限速触发 | REST endpoint 超 QPS |
| 500 | `INTERNAL_ERROR` | Server | 未预期的内部错误 | 内存不足、磁盘满 |
| 500 | `DB_ERROR` | Server | SQLite 操作失败 | WAL 文件损坏、连接池耗尽 |
| 502 | `EXPORT_FAILED` | Server | 导出失败 | xlsx/pptx 引擎异常 |
| 503 | `SERVICE_UNAVAILABLE` | Server | 后端不可用 | 数据库锁定、集群节点宕机 |
| 504 | `TASK_TIMEOUT` | Server | 任务超时终止 | 调度任务 > timeout_seconds |

### Error Response Format (Frozen)

所有错误返回统一 JSON 结构：

```json
{
  "error": {
    "code": "DB_ERROR",
    "http_status": 500,
    "message": "SQLite connection pool exhausted",
    "trace_id": "abc123...",           // Optional: present in Pro+ editions
    "details": null                    // Optional debug info (debug builds only)
  }
}
```

### Breaking Rules

- **Existing codes MUST NOT be removed** across patch and minor versions.
- **Meaning/category of a code must not change** — `POLARS_ERROR` means "client computation error" in v0.x forever.
- **New codes may be added** at any time (server-side returns new codes, clients that don't recognize the code should treat it as `INTERNAL_ERROR`).
- **HTTP mapping is stable**: `DB_ERROR` will always map to 5xx.

### Snapshot Test

```bash
# Regenerate and diff API error code snapshot
cargo test --test api_error_codes
# On failure: update expected output with:
cp .api-error-snapshot.json.new .api-error-snapshot.json
```
