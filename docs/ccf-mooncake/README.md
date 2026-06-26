# CCF Mooncake Store 初赛材料

团队：999感冒灵
队长：李佳斌
队员：李佳禾、姚舒文

## 参赛方向

赛题 2：优化 Mooncake Store 吞吐性能、高可用功能和可扩展性。

本提交解决 Mooncake Store promotion-on-hit 链路中的 transient admission failure 问题。当前台读命中只有 LOCAL_DISK 副本的 hot key 时，如果 MEMORY admission 被水位、队列上限或临时 push 失败挡住，原链路会丢失这次 promotion 机会。本实现将该对象记录为有边界的 retry candidate，资源恢复后重新进入原 admission path。

对应社区 issue：https://github.com/kvcache-ai/Mooncake/issues/2233

## 核心代码

- `mooncake-store/include/master_service.h`
- `mooncake-store/src/master_service.cpp`
- `mooncake-store/tests/promotion_on_hit_test.cpp`
- `mooncake-store/tests/promotion_retry_benchmark_test.cpp`
- `mooncake-store/tests/CMakeLists.txt`

## 设计摘要

- 不修改客户端 RPC、heartbeat 协议和外部 API。
- 只对 transient admission failure 重试，包括 watermark、global in-flight cap 和 recoverable push failure。
- 对对象删除、已有 MEMORY 副本、无 LOCAL_DISK source、已有 in-flight task 等状态直接清理。
- 通过 candidate 上限、TTL、最大尝试次数、指数退避和 bounded shard scan 限制后台开销。
- 复用原 promotion task、holder queue、refcnt pin、metrics 和 metadata lock 边界。

## 材料清单

- [方案说明](solution.md)
- [实验结果](experiment_results.md)
- [复现说明](reproduce.md)
- [工程证据](engineering_evidence.md)
- [benchmark JSON](promotion_retry_benchmark_result.json)
- [初赛报告 PDF](preliminary_report.pdf)
- [答辩 PPT](mooncake-promotion-retry.pptx)
- [演示视频](mooncake-promotion-retry-demo-ace55009f928.mp4)

上游 PR：https://github.com/kvcache-ai/Mooncake/pull/2680
