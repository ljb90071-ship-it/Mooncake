# 实验结果

团队：999感冒灵
队长：李佳斌
队员：李佳禾、姚舒文

## 环境

- OS：Ubuntu 26.04 LTS on WSL2
- Kernel：`Linux MYLOAD 6.6.114.1-microsoft-standard-WSL2`
- Build：CMake + Ninja，启用 Mooncake Store 单元测试目标
- Branch：`ljb/ccf-store-promotion-retry`

## 命令

```bash
cmake --build build-ccf-ninja --target promotion_on_hit_test promotion_retry_benchmark_test -j2
./build-ccf-ninja/mooncake-store/tests/promotion_on_hit_test --gtest_brief=1
./build-ccf-ninja/mooncake-store/tests/promotion_retry_benchmark_test --gtest_brief=1
```

## 结果

| 项目 | 结果 |
|---|---|
| `promotion_on_hit_test` | 43/43 PASS |
| `promotion_retry_benchmark_test` | PASS |
| benchmark workload | `two_hot_local_disk_only_keys` |
| `fallback_reads_needed` | 0 |

## Benchmark 语义

benchmark 构造两个 hot、LOCAL_DISK-only 的 key，并将 `promotion_queue_limit` 设置为 1：

1. `bench_first` 先进入 promotion queue，占满 in-flight slot。
2. `bench_retry` 触发 cap gate，被记录为 retry candidate。
3. `bench_first` 通过 `NotifyPromotionFailure` 释放 slot。
4. 测试只通过 public heartbeat 观察后台 retry 是否让 `bench_retry` 入队。

核心观察值是 `fallback_reads_needed=0`：slot 释放后，后台 retry 完成入队，不再需要额外 foreground Get 才能重新触发 promotion。

## 结果文件

原始 benchmark JSON 见 `promotion_retry_benchmark_result.json`。
