# 第十七章：代码证据索引

[返回总目录](../README.md)

## 1. 本章导读

这一章不做展开分析，只做证据索引，便于从总目录跳到源码。

## 2. 入口与启动链路

- [`src/entrypoints/cli.tsx`](../src/entrypoints/cli.tsx)
- [`src/main.tsx`](../src/main.tsx)
- [`src/entrypoints/init.ts`](../src/entrypoints/init.ts)
- [`src/setup.ts`](../src/setup.ts)
- [`src/replLauncher.tsx`](../src/replLauncher.tsx)
- [`src/screens/REPL.tsx`](../src/screens/REPL.tsx)

## 3. Query / Agent 内核

- [`src/query.ts`](../src/query.ts)
- [`src/QueryEngine.ts`](../src/QueryEngine.ts)
- [`src/utils/queryContext.ts`](../src/utils/queryContext.ts)
- [`src/constants/prompts.ts`](../src/constants/prompts.ts)
- [`src/context.ts`](../src/context.ts)

## 4. 工具与权限

- [`src/tools.ts`](../src/tools.ts)
- [`src/Tool.ts`](../src/Tool.ts)
- [`src/services/tools/toolOrchestration.ts`](../src/services/tools/toolOrchestration.ts)
- [`src/utils/permissions/permissionSetup.ts`](../src/utils/permissions/permissionSetup.ts)

## 5. 状态与 UI

- [`src/state/AppStateStore.ts`](../src/state/AppStateStore.ts)
- [`src/state/store.ts`](../src/state/store.ts)
- [`src/interactiveHelpers.tsx`](../src/interactiveHelpers.tsx)

## 6. transcript / 持久化 / 会话恢复

- [`src/utils/sessionStorage.ts`](../src/utils/sessionStorage.ts)
- [`src/services/api/sessionIngress.ts`](../src/services/api/sessionIngress.ts)
- [`src/utils/settings/types.ts`](../src/utils/settings/types.ts)
- [`src/utils/gracefulShutdown.ts`](../src/utils/gracefulShutdown.ts)

## 7. memory 体系

- [`src/memdir/memdir.ts`](../src/memdir/memdir.ts)
- [`src/memdir/paths.ts`](../src/memdir/paths.ts)
- [`src/memdir/findRelevantMemories.ts`](../src/memdir/findRelevantMemories.ts)
- [`src/memdir/teamMemPaths.ts`](../src/memdir/teamMemPaths.ts)
- [`src/services/extractMemories/extractMemories.ts`](../src/services/extractMemories/extractMemories.ts)
- [`src/services/SessionMemory/sessionMemory.ts`](../src/services/SessionMemory/sessionMemory.ts)
- [`src/services/compact/sessionMemoryCompact.ts`](../src/services/compact/sessionMemoryCompact.ts)
- [`src/tools/AgentTool/agentMemory.ts`](../src/tools/AgentTool/agentMemory.ts)
- [`src/tools/AgentTool/agentMemorySnapshot.ts`](../src/tools/AgentTool/agentMemorySnapshot.ts)
- [`src/tools/AgentTool/loadAgentsDir.ts`](../src/tools/AgentTool/loadAgentsDir.ts)

## 8. analytics / 隐私 / 反馈

- [`src/services/analytics/index.ts`](../src/services/analytics/index.ts)
- [`src/services/analytics/config.ts`](../src/services/analytics/config.ts)
- [`src/services/analytics/metadata.ts`](../src/services/analytics/metadata.ts)
- [`src/services/analytics/sink.ts`](../src/services/analytics/sink.ts)
- [`src/services/analytics/datadog.ts`](../src/services/analytics/datadog.ts)
- [`src/utils/privacyLevel.ts`](../src/utils/privacyLevel.ts)
- [`src/utils/user.ts`](../src/utils/user.ts)
- [`src/utils/fileOperationAnalytics.ts`](../src/utils/fileOperationAnalytics.ts)
- [`src/services/api/grove.ts`](../src/services/api/grove.ts)
- [`src/components/grove/Grove.tsx`](../src/components/grove/Grove.tsx)
- [`src/components/FeedbackSurvey/submitTranscriptShare.ts`](../src/components/FeedbackSurvey/submitTranscriptShare.ts)

## 9. MCP / remote / swarm / team memory

- [`src/services/mcp/client.ts`](../src/services/mcp/client.ts)
- [`src/services/mcp/auth.ts`](../src/services/mcp/auth.ts)
- [`src/entrypoints/mcp.ts`](../src/entrypoints/mcp.ts)
- [`src/bridge/bridgeMain.ts`](../src/bridge/bridgeMain.ts)
- [`src/utils/swarm/backends/registry.ts`](../src/utils/swarm/backends/registry.ts)
- [`src/utils/swarm/spawnInProcess.ts`](../src/utils/swarm/spawnInProcess.ts)
- [`src/services/teamMemorySync/index.ts`](../src/services/teamMemorySync/index.ts)
- [`src/services/teamMemorySync/watcher.ts`](../src/services/teamMemorySync/watcher.ts)

## 10. 本章小结

本索引章节的作用是把“结论”与“证据”分离。前几章用于叙述，当前章节用于追溯和复核。
