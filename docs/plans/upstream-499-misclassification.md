# 上游 499 误分类修复计划

> 时间：2026-03-10 11:17 CST

## 背景

当前 `/v1` 代理链路会把两类场景误记为纯客户端 `499`：
- 上游真实返回 `HTTP 499`
- 首字节迟迟未到时，请求侧链路先取消，导致 CCH 只能看到 `clientAbortSignal`

这会让供应商失败漏记，影响熔断与故障切换。

## 选定方案

在 `forwarder/errors` 内增加“疑似上游首字节前取消”归因逻辑：
- 仅在首字节尚未到达、请求侧已断开、且无本地超时命中的窗口触发
- 统一转成供应商错误而非纯客户端 `499`
- 保留真正用户主动取消的 `499` 语义

同时收紧 `isClientAbortError`，避免上游真实 `HTTP 499` 被直接判成客户端断开。

## 执行步骤

1. 先补回归测试，覆盖上游真实 `HTTP 499` 与首字节前请求侧断开场景。
2. 最小修改 `errors.ts` 的客户端中断判定与分类逻辑。
3. 最小修改 `forwarder.ts` 的首字节前取消归因、日志和 provider chain 记录。
4. 运行定向单测验证红绿回归。

## 涉及文件

- `src/app/v1/_lib/proxy/errors.ts`
- `src/app/v1/_lib/proxy/forwarder.ts`
- `tests/unit/proxy/*.test.ts`
