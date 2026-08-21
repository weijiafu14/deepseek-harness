# Agent Note: Agent setup contributors

Status: implemented

English | [中文](2026-08-21-agent-setup-contributors.zh.md)

## Problem

`CreateAgentOptions.setup` and `ResumeAgentOptions.setup` provide the correct unpublished transaction for scoped tools, prompt sections, listeners, and awaited child plugins. The callback belongs to the immediate Agent creator, however, so a deployment plugin that must equip every Agent has no general registration point. Requiring each surface to forward that plugin's callback couples Web, terminal, ACP, SDK, workflow, and subagent creators to every cross-cutting capability. Mounting the capability on the host instead changes its ownership: multiple live Agents share one plugin closure and one teardown path even when the capability models per-session state.

`agent/session-start` occurs after publication and is a synchronous notification. It can announce startup, but it cannot await composition or roll a failed mount back before the Agent becomes observable.

## Decision

`AgentRegistry.create()` and `AgentRegistry.resume()` compose the caller's `AgentSetup` with a serial `agent/setup` event. The registry invokes the caller first, retains its optional `AgentSetupCommit`, awaits every contributor with the unpublished Agent, its `agentCtx`, and creation or resume request facts, and returns the caller commit to the factory. The factory invokes that commit at its existing publication boundary.

A contributor registers once on the host context and mounts capabilities only through the payload's `agentCtx`. The Agent scope owns those effects, so disposal unwinds one Agent without changing another. A contributor rejection follows the existing setup failure path: the factory disposes the unpublished Agent scope and publishes neither Agent nor Session.

The event is deliberately not an Agent-subject lifecycle event. Its subject is an unpublished context passed as an explicit capability; it has no published Agent carrier and is excluded from scoped-event dispatch generation. Runtime lifecycle begins with `agent/session-start` after publication as before.

## Verification

The AgentRegistry contract tests pin caller → contributor → commit → publication order for create and resume and prove a contributor rejection aborts creation. Scope invariant coverage records that `agent/setup` is the one `agent/*` event without a published subject carrier. Product compositions exercise the same registry methods, so Web, terminal, ACP, SDK, workflow, and subagent callers receive contributors without surface-specific forwarding.

## Alternatives considered

**Require every Agent creator to merge deployment setup callbacks.** Rejected because each surface would own the same forwarding logic and could omit it on create, resume, replacement, or child creation. A cross-cutting deployment capability would become a surface compatibility matrix.

**Mount per-session plugins from `agent/session-start`.** Rejected because returned promises cannot delay startup, failures occur after publication, and the first prompt or command can race an unfinished mount.

**Mount one plugin instance on the host and route by current Agent.** Rejected because closure state, asynchronous initialization, and disposal remain shared. Routing calls by Agent cannot make package-private state or teardown ownership per-Agent.

**Add a contributor registry to one surface.** Rejected because it solves only that surface. The Agent registry is the common creation boundary already used by every surface and child-agent path.

## Consequences

Deployment plugins can install one fresh capability instance per Agent without changing Agent creators. Setup latency includes every serial contributor, and any contributor can veto publication by rejecting, so contributors must perform only required composition work and report failures precisely.

The registry event must be active before an Agent creation begins. Composition manifests remain responsible for loading contributor plugins before a surface that creates an Agent during startup; the Core contract does not retroactively modify an already published Agent.
