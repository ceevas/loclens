# Loclens Ingest API 文档

数据摄入 HTTP 端点规范 — 供外部系统推送事件到流处理引擎。

## 认证机制

| 模式 | 描述 | 配额限制 |
|------|------|---------|
| API Key (Header) | `Authorization: Bearer sk-loclens-<key>` | 按版本配额 |
| HMAC Token | `X-LocLens-Signature: <sig>` | 无限 |

密钥存储采用 SHA-256 + salt，永远不会以明文形式暴露。创建时仅显示一次完整密钥。

### 密钥配额 (P14-33)

| 版本 | 最大密钥数 | RPS 限制 |
|------|-----------|----------|
| Beta | 0 | 0 |
| Community | 5 | 1,000 |
| Pro | 20 | 10,000 |
| Team | 50 | 无限 |
| Enterprise | 无限 | 无限 |

## 端点列表

### POST /api/v1/ingest

向指定 Stream Source 推送事件。

**请求头:**
```
Authorization: Bearer sk-loclens-<key>
Content-Type: application/json
X-LocLens-Timestamp: <unix_epoch_ms>
```

**RequestBody:**
```json
{
    "stream_source": "<source_id>",
    "events": [
        {
            "event_key": "unique_id_string",
            "seq": 1,
            "data": { "field_a": 123, "field_b": "value" }
        }
    ]
}
```

**响应码:**
| 状态码 | 含义 |
|--------|------|
| 200 | 事件已接收 |
| 401 | 无效/过期密钥 |
| 403 | 撤销的密钥 |
| 413 | 单请求过大 (>8MB / >10k events) |
| 429 | 配额超限 |

### GET /api/v1/ingest/status/:source_id

查询 Stream Source 实时状态（水位线、延迟等）。

**响应:**
```json
{
    "source_id": "alpha",
    "watermark_ms": 1705324800000,
    "events_received_1h": 50000,
    "error_rate_pct": 0.02,
    "state_store_size_mb": 12.5,
    "partition_id": 0
}
```

### POST /api/v1/ingest/retry/:source_id

手动触发改源重新投递未确认的事件。仅限 Pro 及以上版本，冷却期 30 秒内不重复发送。

**Response on success:** `{"retries_submitted": N, "status":"accepted"}`

### DELETE /api/v1/ingest/clean/:source_id

清除指定源的全部状态和缓存（高危操作）。需二级确认：请求头需包含 `X-Confirm: <source_id>`。

---

## 限流与幂等

### 幂等保证

通过 `event_key + seq` 去重。相同 key+seq 到达时返回 `200 AlreadyProcessed`。

| 级别 | 保序窗口 | LRU 缓存容量 |
|------|---------|------------|
| Pro / Team | ~1h | ~50M keys |
| Enterprise | 不限 | 不限 |

### RPS 限流

基于滑动窗口算法，超限后返回 `429 Too Many Requests`：

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
Retry-After: 30

{ "error": "rps limit exceeded (1000)", "retry_after_s": 30 }
```

## 重试机制

客户端应在收到以下响应时自动重试：

| 状态码 | 建议最大重试次数 | 初始退避(s) | 最大退避(s) |
|--------|---------------|-----------|-----------|
| 429 | 5 | 30 | 300 |
| 5xx | 3 | 1 | 60 |

事件被客户端缓存至确认到达，超时后写入本地 dead-letter-queue。

## 分布式场景

在多节点部署下：
- **Partition**: Source → Worker by hash(source_id)
- **Failover**: Worker down → Coordinator reassigns partition + rebalances
- **Replica Factor**: Configurable per source (default: 1, max: 3 on Enterprise)

## 示例

### cURL - Push Events
```bash
curl -X POST http://localhost:8080/api/v1/ingest \
    -H "Authorization: Bearer sk-loclens-xxxxxxxxx" \
    -H "Content-Type: application/json" \
    -d '{
        "stream_source": "alpha",
        "events": [
            {"event_key":"evt001","seq":1,"data":{"value":42}}
        ]
    }'
```

### JavaScript SDK (伪代码)
```javascript
const loclens = new LoclensIngest({ apiKey: 'sk-loclens-xxx' });
await loclens.push('alpha', [
    { eventKey: 'evt001', seq: 1, data: { value: 42 } },
]);
```
