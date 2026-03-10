# ContextLibrary: 上游 499 误分类与熔断修复

## 基本信息

| 字段 | 内容 |
|------|------|
| 需求时间 | 2026-03-10 |
| 修改时间 | 2026-03-10 |
| 涉及文件 | src/app/v1/_lib/proxy/errors.ts, src/app/v1/_lib/proxy/forwarder.ts, src/app/v1/_lib/proxy/response-handler.ts, tests/integration/proxy-errors.test.ts, tests/unit/proxy/proxy-forwarder-endpoint-audit.test.ts, docs/plans/upstream-499-misclassification.md |
| 需求摘要 | 修复 CCH 将上游慢响应/上游取消误记为客户端 499，导致供应商失败未正确进入熔断的问题 |

## 需求详情

用户在线上观察到间歇性 `499`，典型耗时约 60 秒或 89.99 秒，且确认不是人工断开连接。结合链路现象，问题集中在以下两类情况：

- 上游真实返回 `HTTP 499`
- 首字节长期未到达时，请求侧链路先取消，CCH 只能看到 `clientAbortSignal`，最终被内部记成 `499`

这会让供应商失败漏记，导致熔断与故障切换失真。

## 解决方案

采用最小根治方案，核心思路如下：

- 为代理内部生成的客户端 `499` 增加显式分类提示 `classificationHint = "client_abort"`
- 收紧 `isClientAbortError()` 判定，不再把任意 `ProxyError(499)` 直接认定为客户端断开
- 增加“疑似上游首字节前取消”判定：
  - 客户端 abort 已发生
  - 尚未收到首字节
  - forward 已开始
  - 本地 response timeout 未触发
  - 等待时间达到 45 秒阈值
- 命中上述条件时，将原本的客户端 `499` 重归因为供应商 `502`，进入既有熔断和供应商切换逻辑
- 在流式延迟结算中同步应用同一归因规则，避免内部统计继续落成 `499`

## 关键决策

| 决策点 | 选择 | 原因 |
|--------|------|------|
| 客户端中断识别 | 仅信任显式 hint 或标准 AbortError 名称 | 避免把上游真实 HTTP 499 误判为客户端断开 |
| 首字节前取消阈值 | 45 秒 | 保守识别长等待场景，尽量减少误伤真实主动取消 |
| 重归因状态码 | 502 | 保持“供应商失败”语义，复用现有熔断/切换链路 |
| 流式场景处理 | 同步修复延迟结算 | 避免请求前段修复后，后台结算仍落成 499 |

## 问题与解决

| 问题 | 解决方法 |
|------|----------|
| 上游真实 HTTP 499 被当成客户端取消 | 取消 `statusCode === 499` 的宽泛客户端判定，改为显式 hint |
| 首字节前长等待后断链未计入熔断 | 在 forwarder 中重归因为 `502` 并调用既有 provider failure 路径 |
| 流式统计仍可能记为 499 | 在 response-handler 延迟结算中改为 `502` |

## 成果

- [x] `ProxyError` 支持显式客户端取消分类提示
- [x] `isClientAbortError()` 收紧，真实上游 `499` 改走 provider error
- [x] `ProxyForwarder.send()` 增加首字节前取消的供应商失败重归因
- [x] `ResponseHandler` 延迟结算同步修复
- [x] 补充上游 `499` 与首字节前取消场景的回归测试

## 验证

- 已执行：`git diff --check`，无空白错误
- 未执行：测试命令。原因是用户明确要求“不需要测试”

## 备注

- 本次提交未触碰用户已有的 `docker-compose.yaml` 与 `cch-log.txt`
- 当前本地 `main` 分支相对 `origin/main` 落后 1 个提交，若要继续推送远端，需要先安全处理远端同步
