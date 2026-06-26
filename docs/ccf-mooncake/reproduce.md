# 复现说明

团队：999感冒灵
队长：李佳斌
队员：李佳禾、姚舒文

## 分支

- 比赛交付分支：`ljb/ccf-store-promotion-retry`
- 上游 PR 分支：`ljb/store-promotion-retry-upstream`
- 上游 PR：https://github.com/kvcache-ai/Mooncake/pull/2680

## 构建与测试

在 WSL2 Ubuntu-26.04 的 Mooncake 仓库根目录运行：

```bash
cmake --build build-ccf-ninja --target promotion_on_hit_test promotion_retry_benchmark_test -j2
./build-ccf-ninja/mooncake-store/tests/promotion_on_hit_test --gtest_brief=1
./build-ccf-ninja/mooncake-store/tests/promotion_retry_benchmark_test --gtest_brief=1
```

## 预期结果

- `promotion_on_hit_test`：43/43 PASS
- `promotion_retry_benchmark_test`：PASS
- benchmark 目标：`fallback_reads_needed=0`

benchmark 输出文件：`promotion_retry_benchmark_result.json`。
