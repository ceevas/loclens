# Pro Performance Baseline (P-Phase)

## 测试条件

| 参数 | 值 |
|------|-----|
| CPU | Intel Xeon E5-2630 v4 @ 2.20GHz, 40 cores |
| RAM | 63 GB DDR4 ECC |
| OS | Ubuntu 22.04.5 LTS / Kernel 6.8.0-138-generic |
| Polars | 0.44 |
| Rust | rustc 1.97.1 |
| App Version | v0.2.0 |

## 测试数据集

| 文件 | 大小 | SHA256 |
|------|------|--------|
| `bench/fixtures/bench_1gb.csv` | ~1.31 GB | `0f89...a44719d3a883` |
| `bench/fixtures/bench_1gb.parquet` | ~0.45 GB | `5826...6c602a9673` |

数据生成: 确定性随机种子 (seed=42), 10M rows × 11 columns, ISO-8601 timestamps, 8 regions, 20 categories。
SHA256 校验文件: `bench/fixtures/SHA256SUMS`

## 基准指标 (p50 / best-of-3)

| # | 指标 | 目标 | 实测 | 状态 |
|---|------|------|------|------|
|| 1 | **冷启动** (CSV 50K行扫描+聚合) | ≤ 3000ms | **222ms** (best of 3) | ✅ PASS |
|| 2 | **全量 CSV 读取 + describe()** | ≤ 60s | **46.75s** (p50) / 49.0s (p95) | ✅ PASS |
|| 3 | **全量 Parquet 读取 + describe()** | ≤ 20s | **2.5s** (p50, adjusted) / 2.61s (p95) | ✅ PASS |
|| 4 | **峰值 RSS / 输入大小比** | ≤ 4.0x | **0.6x** (0.91 GB RSS vs 1.40 GB input) | ✅ PASS |
|| 5 | **图表交互代理** (100K行 filter + groupby) | ≤ 500ms | **378ms** | ✅ PASS |

> 实测时间: 2026-11-08T01:15 UTC。完整数据见 `bench-report.json`。
> CI 采用相对变化阈值保护 (默认 ±20%)。

## 复现命令

```bash
# 一键基准测试（含环境记录 → bench-report.json）
cd /home/dd/dataana/data-analyzer
bash scripts/bench-perf.sh

# CI 回归比对 (±20% 容忍)
bash scripts/bench-perf.sh prev-bench-report.json

# 自定义阈值 (±10%)
REGRESSION_THRESHOLD_PCT=10 bash scripts/bench-perf.sh prev.json
```

## 基准报告格式 (bench-report.json)

```json
{
  "version": "0.2.0",
  "timestamp": "2026-...Z",
  "environment": { "cpu_model", "cpu_cores", "total_ram_gb", "os", ... },
  "fixtures": { "csv_sha256", "parquet_sha256", "csv_size_bytes", ... },
  "benchmarks": {
    "cold_start": { "runs_ms": [...], "best_ms": number },
    "csv_read":   { "runs_s": [...], "p50_s": number, "p95_s": number },
    "pq_read":    { "runs_s": [...], "p50_s": number, "p95_s": number },
    "memory_peak":{ "rss_kb", "rss_gb", "ratio_to_input" },
    "chart_filter": { "elapsed_ms", "threshold_ms": 500, "pass": true }
  }
}
```

## CI 回归保护 (P-1.4)

在 GitHub Actions / GitLab CI 中：

```yaml
steps:
  - run: |
      # Save baseline on main branch
      if [ ! -f .bench-baseline.json ]; then
        bash scripts/bench-perf.sh
        cp bench-report.json .bench-baseline.json
      fi
      # Compare against saved baseline
      REGRESSION_THRESHOLD_PCT=20 bash scripts/bench-perf.sh .bench-baseline.json
```

阈值超过 +20% → CI 标记为 FAILED，阻止 PR 合并。

## 复现包 (P-1.7)

对外提供一键复现：

```bash
# 客户只需拉取项目并运行一条命令即可复现基准结果
python3 scripts/generate-bench-fixtures.py   # 生成固定数据 + SHA256
bash scripts/bench-perf.sh                   # 跑基准，输出 bench-report.json
```

详见 `_repro/run_benchmark.sh` 一键脚本。
