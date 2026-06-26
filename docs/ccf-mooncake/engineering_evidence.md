# 工程证据

团队：999感冒灵
队长：李佳斌
队员：李佳禾、姚舒文

## 覆盖范围

验证重点是 Mooncake Store promotion-on-hit retry 修改路径：

- transient admission failure 释放资源后重试成功。
- permanent/resolved 状态不重试。
- 超过最大尝试次数后清理候选。
- metadata reset 清理候选状态。
- cleanup 竞态下 candidate counter 不下溢。
- deterministic public-API benchmark 验证后台 retry 可见。

## 突变/扰动验证

本次没有声称完成全仓库 mutation testing。针对改动路径，采用确定性故障注入和边界扰动覆盖关键分支：queue cap 拒绝、watermark 拒绝、holder queue push failure、对象删除、MEMORY 副本已出现、LOCAL_DISK source 缺失、已有 in-flight task、candidate 超时放弃和 metadata reset。每个扰动都对应可复现的单元测试或 public-API benchmark 观察点。

本材料不声称覆盖整个 Mooncake 仓库；覆盖范围与本次修改的行为边界一致。

## 故障分类

可重试：

- DRAM watermark admission rejection。
- holder queue capacity rejection。
- recoverable holder queue push failure。

不重试：

- retry 前对象已删除。
- 已存在 MEMORY 副本。
- 不存在 LOCAL_DISK source 副本。
- promotion task 已经在 flight。

## 协议边界

实现不改变外部 RPC、client heartbeat schema 或 public API。retry state 仅存在于 master 内部，最终入队仍复用原 promotion task、holder queue、refcnt pin、metrics 和 metadata lock 路径。

## 随机性

benchmark workload 为 `two_hot_local_disk_only_keys`，流程确定，不需要随机种子。

## 环境

- WSL2 Ubuntu-26.04
- Mooncake Store C++ test targets
- 上游 PR：https://github.com/kvcache-ai/Mooncake/pull/2680
