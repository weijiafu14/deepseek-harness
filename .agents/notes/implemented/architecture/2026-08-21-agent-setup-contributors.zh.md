# Agent Note: Agent setup 贡献者

Status: implemented

[English](2026-08-21-agent-setup-contributors.md) | 中文

## 问题

`CreateAgentOptions.setup` 与 `ResumeAgentOptions.setup` 为作用域工具、提示词 section、监听器及需要等待的子插件提供了正确的未发布事务。然而，该回调属于直接创建 Agent 的一方，因此必须为每个 Agent 配备能力的部署插件没有通用注册点。若要求每个 surface 转发该插件的回调，Web、终端、ACP、SDK、workflow 与 subagent 创建方都会耦合到每一项横切能力。若改为把能力挂在宿主上，其所有权又会改变：多个存活 Agent 会共享同一份插件闭包与 teardown 路径，即使该能力表达的是逐会话状态。

`agent/session-start` 发生在发布之后，并且是同步通知。它可以宣布启动，但不能等待组合完成，也不能在 Agent 可见之前回滚失败的挂载。

## 决策

`AgentRegistry.create()` 与 `AgentRegistry.resume()` 会把调用方的 `AgentSetup` 和串行 `agent/setup` 事件组合起来。注册表先调用调用方，保留其可选的 `AgentSetupCommit`，随后把尚未发布的 Agent、它的 `agentCtx` 以及创建或恢复请求事实交给每个贡献者并等待它们完成，最后把调用方的 commit 返回给工厂。工厂仍在既有发布边界调用该 commit。

贡献者在宿主上下文注册一次，并且只通过 payload 中的 `agentCtx` 挂载能力。Agent 作用域拥有这些 effect，因此销毁一个 Agent 不会改变另一个。贡献者拒绝沿用既有 setup 失败路径：工厂销毁尚未发布的 Agent 作用域，并且不发布 Agent 或 Session。

该事件刻意不是一个以 Agent 为主体的生命周期事件。它的主体是作为显式 capability 传入、尚未发布的上下文；此时没有已发布的 Agent carrier，因此该事件不参与作用域事件分发生成。发布后的运行期生命周期仍和此前一样从 `agent/session-start` 开始。

## 验证

AgentRegistry 契约测试固定了 create 与 resume 的“调用方 → 贡献者 → commit → 发布”顺序，并证明贡献者拒绝会中止创建。作用域 invariant 覆盖记录 `agent/setup` 是唯一没有已发布主体 carrier 的 `agent/*` 事件。产品组合均调用同一组注册表方法，因此 Web、终端、ACP、SDK、workflow 与 subagent 调用方无需 surface 专属转发即可获得贡献者。

## 备选方案

**要求每个 Agent 创建方合并部署 setup 回调。** 否决，因为每个 surface 都会拥有同一份转发逻辑，并且可能在创建、恢复、替换或 child 创建路径遗漏。横切部署能力将变成一张 surface 兼容性矩阵。

**从 `agent/session-start` 挂载逐会话插件。** 否决，因为返回的 Promise 无法延迟启动，失败发生在发布之后，而且第一次提示或命令可能与尚未完成的挂载发生竞态。

**在宿主挂一份插件实例，再按当前 Agent 路由。** 否决，因为闭包状态、异步初始化与销毁所有权仍然共享。按 Agent 路由调用无法把包私有状态或 teardown 所有权变成逐 Agent。

**只给一个 surface 增加贡献者注册表。** 否决，因为它只解决该 surface。Agent 注册表是所有 surface 与 child-agent 路径已经共同使用的创建边界。

## 后果

部署插件无需修改 Agent 创建方，即可为每个 Agent 安装一份全新的能力实例。Setup 延迟包含所有串行贡献者，而且任一贡献者都可以通过拒绝来否决发布，因此贡献者只能执行必要的组合工作，并应准确报告失败。

贡献者事件必须在 Agent 创建开始前已激活。组合清单仍负责让贡献者插件先于启动期间会创建 Agent 的 surface 加载；Core 契约不会追溯修改已经发布的 Agent。
