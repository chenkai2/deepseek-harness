# Agent Note: tool-bash 前台命令中断后通过正常管道返回结果

Status: implemented

[English](2026-08-18-tool-bash-foreground-abort-natural-pipeline.md) | 中文

## 问题

当前台 bash 命令被中断（调用者信号中止，而非执行器超时）时，`tool-bash` 的 execute 函数抛出
`AbortError` 而非返回 `aborted: true` 的 `ShellRunResult`。工具运行时的 `dispatchToolBody`
已正确处理取消：在 body 返回后，`isAborted(signal)` 路由到 `toolAbortedResult()`，它会将会话
日志中提交一个正确的错误结果。但抛出的异常绕过了该路径，进入了 `catch` 块。

`catch` 块调用 `toolErrorResult()`，也会产生一个 `isError` 结果。然而，抛出异常改变了调度器
中的控制流：dispatch promise 的解析方式不同，在某些边缘情况下（例如信号和工具输出收集之间的
竞态条件），`tool/result` 事件可能会从会话日志中缺失。下一次请求将发送包含 `tool_calls` 的
`assistant` 消息，但没有对应的 `tool` 消息，DeepSeek API 以 `400001: insufficient tool
messages following tool_calls message` 拒绝该请求。会话因此卡死——后续每一轮对话都以同样的
方式失败。

## 决策

从前台执行路径中移除 `if (result.aborted) { throw ... }` 代码块。当调用者信号中止命令时，
`LocalBashExecutor.run()` 方法返回 `aborted: true`（且 `timedOut: false`——两者由 deadline
库的首次原因规则互斥）的 `ShellRunResult`。execute 函数正常返回该结果，`dispatchToolBody`
在 `await` 后的 `isAborted(signal)` 检查路由到正确的 `toolAbortedResult()` 路径。结果是
`isError: true` 且 `content: "Error: tool call aborted"` 的 `ToolExecutionResult`，
`appendToolResult` 将其作为 `tool/result` 事件记录到会话日志中。会话表面完整，下一次请求
不再出现 API 错误。

后台路径（`run_in_background`）不受影响：它已有自己的前置中止检查，在任何进程启动前抛出异常，
调度器通过准备阶段的 `toolAbortedBeforeDispatchResult` 路径处理该异常，该路径正确追加了
合成的 `tool/result` 事件。

## 验证

所有 108 个现有单元测试和 4 个集成测试通过。此变更是直接删除四行代码（`if` 代码块）；未引入
新逻辑，因此现有的超时测试（`reports timeout kills with both markers`）和前置中止后台测试
（`a pre-aborted call is skipped before the process starts`）继续覆盖中止契约。

## 备选方案

**保留抛出并修复调度器。** 调度器的 `commitReady` 路径已对所有其他工具正确工作；该 bug 是
`tool-bash` 在工具运行时期望正常返回的路径中注入了一个抛出。修复调用者更简单，并从错误分类
中移除了噪声。

**在 `dispatchToolBody` 中捕获并强制使用 `toolAbortedResult`。** `catch` 块已路由到
`toolErrorResult`，产生相同的 `isError` 结果。差异很细微（`toolAbortedResult` 路径保留了
先前结果的 `additionalContexts`，而 bash 工具从未设置过），因此不会改变模型可见的输出。但
保留抛出意味着 `dispatchToolBody` 中的 `isAborted(signal)` 分支对 bash 工具来说是死代码，
调度器的 `toolAbortedResult` 逻辑从未被最常用的中止场景执行。

## 影响

被调用者信号中断的前台 bash 命令总是在会话日志中产生 `tool/result` 事件。`tool-call aborted`
错误对模型可见，会话在后续轮次中仍可使用。`dispatchToolBody` 的 `isAborted(signal)` 分支
现在被前台 bash 执行，符合调度器取消结果选择文档中的契约。
