# Mooncake Store Promotion Retry 方案说明

团队：999感冒灵
队长：李佳斌
队员：李佳禾、姚舒文

## 问题

Mooncake Store 的 promotion-on-hit 会在 hot key 只有 LOCAL_DISK 副本时尝试提升到 MEMORY。原链路的 admission 判断是一次性的：如果当时 DRAM watermark 较高、global in-flight cap 已满，或 holder queue push 临时失败，前台 Get 仍会成功返回，但这次 promotion 机会被丢弃。后续请求只能继续从 LOCAL_DISK fallback，直到新的前台访问再次触发 promotion。

这个行为不是数据正确性问题，但会影响 hot key 收敛到 L1/MEMORY 的可靠性，也会放大恢复期间的尾延迟和 L2 读压力。

## 目标

- 不改变外部协议和 public API。
- 不让前台 Get 等待后台 promotion。
- 只重试可恢复 admission failure。
- 给后台 retry 明确上限，避免形成新的后台放大器。
- 继续使用原有 promotion task 和 holder queue，减少集成风险。

## 实现

`TryPushPromotionQueue` 从内部 `void` 路径改为返回 `PromotionQueueResult`。前台调用仍可忽略返回值；后台 retry 根据结果分类处理。

新增 `PromotionCandidate`，记录 `first_seen`、`next_retry_time`、`retry_attempts`、`last_reason` 和 `last_error`。候选状态挂在 `TenantState` 下，受 metadata shard lock 保护。

transient 类型：

- `kWatermarkRejected`
- `kQueueCapRejected`
- `kPushFailed`

永久或已解决类型：

- `kNotFound`
- `kMemoryReplicaPresent`
- `kNoLocalDiskSource`
- `kAlreadyInFlight`

后台调度在 eviction loop 中扫描候选。扫描时先在 shard lock 下收集到期 key，然后释放 shard lock，再在 snapshot shared lock 下重新进入原 admission path。这样避免锁重入，也保证最终入队仍由原 refcnt pin、holder queue 和 `promotion_tasks` 路径完成。

## 边界

- 全局 candidate 上限：50000。
- TTL：60 秒。
- 最大尝试次数：8。
- 退避：10ms 起步，最大 1000ms。
- 每轮最多扫描 64 个 metadata shard。

清理路径覆盖对象删除、metadata reset、候选过期、超过最大尝试次数、成功入队和已解决状态。全局计数使用原子预留和饱和递减，避免 reset 与清理交错时下溢。

## 验证

本地验证集中在修改路径：

- transient failure 释放资源后可自动重试并成功入队。
- permanent/resolved 状态不重试。
- 达到最大尝试次数后清理候选。
- metadata reset 清理 transient state。
- candidate counter 在竞态清理下不下溢。
- deterministic benchmark 验证 slot 释放后不需要额外 foreground Get。

本地环境没有 RDMA/NVLink/CXL 多机集群，因此初赛阶段采用单机可复现测试和 public-API benchmark。真实集群下应继续测量 promotion success rate、fallback LOCAL_DISK read 次数和 tail latency。
