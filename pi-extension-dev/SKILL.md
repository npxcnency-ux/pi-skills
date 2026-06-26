---
name: pi-extension-dev
description: 开发 pi 扩展时避坑用的核心知识。当用户要写新 pi 扩展、注册 tool/command/event、调用 ctx.newSession、处理 subagent 协作、对接 OpenAI-compatible provider、或调试扩展加载错误时使用。
---

# Pi Extension 开发避坑指南

写 pi 扩展时容易踩的坑，按场景分类。

## 1. ExtensionContext vs ExtensionCommandContext

> ⚠️ **最常见的混淆**

| 拿到的 ctx 类型 | 来源 | 拥有的能力 |
|----------------|------|----------|
| `ExtensionCommandContext` | slash command handler | **完整**：含 `newSession` / `switchSession` / `fork` |
| `ExtensionContext` | event handler（`agent_end`、`before_agent_start`、`tool_call`、`turn_end`、`session_start` 等） | **受限**：没有 session 操作能力 |
| `ExtensionToolContext` | tool 的 `execute` 函数 | **更受限**：只有 cwd / sessionManager |

### 后果

```ts
pi.on("agent_end", async (event, ctx) => {
  await ctx.newSession({...});  // ❌ ctx.newSession is not a function
});
```

### 正确做法

事件里需要触发 session 操作时，**通过 notify 提示用户运行 slash command**：

```ts
pi.on("agent_end", async (event, ctx) => {
  ctx.ui.notify("计划完成，运行 /myext exec 启动执行", "info");
});
```

## 2. session vs mode

| 概念 | 本质 | 切换方式 | 持久化 |
|------|------|---------|-------|
| **session** | pi 的对话单元 | `ctx.newSession()` / `ctx.switchSession()` | ✅ 磁盘 jsonl |
| **mode** | 扩展自定义状态 | 改 `state.xxx` 变量 | ❌ 进程内存 |

### 关键事实

- 扩展的 `state` 闭包变量**跨 session 共享**（同一进程内）
- session 切换**不重置**扩展 state
- 进程重启 → state 全清 → 必须从磁盘恢复

```ts
export default function (pi: ExtensionAPI) {
  const state = { count: 0 };  // 跨 session 共享，跨进程不共享

  pi.registerCommand("count", {
    handler: (args, ctx) => {
      state.count++;
      // 即使切到新 session，state.count 也保留
    }
  });
}
```

## 3. subagent / worker 是独立子进程

> ⚠️ subagent 不能访问主进程的内存 state

```
主 pi 进程
  ├── 主 session: state.plan = {...}
  └── subagent worker [独立 pi 子进程]
       └── 加载扩展时 state.plan = undefined（新进程，新闭包）
```

### 后果

```ts
pi.registerTool({
  name: "update_task",
  async execute(_, params, _signal, _onUpdate, ctx) {
    if (!state.plan) return { isError: true, ... };  // ❌ subagent 调用永远报错
    state.plan.tasks[0].status = "done";  // ❌ 改的是子进程内存，主进程看不到
  }
});
```

### 正确做法

跨进程共享必须走文件系统：

```ts
async execute(_, params, _signal, _onUpdate, ctx) {
  // 直接读盘，不依赖内存
  const data = readJsonFile(path.join(ctx.cwd, ".myext/state.json"));
  data.tasks[0].status = "done";
  writeJsonFile(path.join(ctx.cwd, ".myext/state.json"), data);

  // 主进程内存有的话顺便同步（可选优化）
  if (state.plan) state.plan.tasks = data.tasks;
}
```

## 4. context filter 的索引陷阱

```ts
pi.on("context", async (event) => {
  // ❌ 危险：新 session 索引重置为 0
  return {
    messages: event.messages.filter((_, idx) => idx >= state.startIdx)
  };
});
```

如果 `state.startIdx` 是旧 session 设置的（比如 5），新 session 里所有消息（idx 0~N）都被过滤掉 → 触发 `messages: at least one message is required`。

### 正确做法

按 customType 或 role 过滤，不依赖索引：

```ts
return {
  messages: event.messages.filter((msg: any) => {
    return msg.customType !== "myext-injected-prompt";
  })
};
```

## 5. OpenAI-compatible provider 的 reasoning 坑

pi 的 openai-responses 实现：

```ts
// pi-mono/packages/ai/src/providers/openai-responses.ts
if (model.reasoning) {
  if (options?.reasoningEffort) {
    params.reasoning = { effort: ..., summary: "auto" };
  } else if (model.thinkingLevelMap?.off !== null) {
    params.reasoning = { effort: "none" };  // ⚠️ 即使 thinking=off 也加
  }
}
```

如果你对接的网关（如 kPI、第三方代理）不支持这个参数，会返回 `400 (no body)` 或 `404 page not found`。

### 正确做法

provider 注册模型时，**只对真支持 reasoning 的模型**设 `reasoning: true`：

```ts
pi.registerProvider("kpi", {
  models: [
    // anthropic 协议支持 → reasoning: true
    { id: "claude-opus-4-7", reasoning: true, ... },
    // OpenAI-responses 协议但网关不识别 → reasoning: false
    { id: "kivy-deepseek-v4-pro", reasoning: false, ... },
  ],
});
```

调用方也要按 provider 自动降级 thinking：

```ts
function thinkingFor(model: ModelDef): ThinkingLevel {
  return model.provider === "anthropic" ? "high" : "off";
}
pi.setThinkingLevel(thinkingFor(targetModel));
```

## 6. promptSnippet 让 LLM 知道扩展 tool

注册 tool 时只设 `description` 不够，LLM 不一定知道 tool 存在。用 `promptSnippet` 注入到 system prompt：

```ts
pi.registerTool({
  name: "subagent",
  description: "...",
  promptSnippet: `## Available subagents\n- planner: ...\n- worker: ...`,
  parameters: ...,
});
```

`promptSnippet` 在扩展加载时收集，整体注入 system prompt。这是让 LLM 在第一轮就能正确调用 tool 的关键。

## 7. Template literal 反引号陷阱

```ts
// ❌ ParseError: Missing semicolon
return `请调用 \`update_task("1", "done")\` 标记进度`;

// ❌ 同样错（外层 template literal 被截断）
return `规则：调用 `update_task` 函数`;

// ✅ 正确
return `请调用 update_task("1", "done") 标记进度`;
return `规则：调用 \`update_task\` 函数`;  // 内层反引号必须转义
```

## 8. ctx.newSession 的 withSession 时机

```ts
await ctx.newSession({
  parentSession,
  withSession: async (newCtx) => {
    // 此时新 session 已初始化完毕，messages[] 已重建
    // 在这里发消息是安全的
    await newCtx.sendUserMessage(kickoff);
  },
});
```

### 不要这样做

```ts
// ❌ session_start 里 setTimeout 推迟发送
pi.on("session_start", async (_, ctx) => {
  setTimeout(() => pi.sendUserMessage(kickoff), 0);  // 时序不可靠
});
```

## 9. 跨 session 数据交换：握手文件

主 session 要给新 exec session 传数据时，靠**文件**而非内存：

```ts
// 主 session
async function launch() {
  fs.writeFileSync(".pending.json", JSON.stringify({ kickoff, config }));
  await ctx.newSession({ withSession: async (newCtx) => {
    await newCtx.sendUserMessage(pendingData.kickoff);
  }});
}

// 新 session 的 session_start
pi.on("session_start", async (_, ctx) => {
  if (fs.existsSync(".pending.json")) {
    const pending = JSON.parse(fs.readFileSync(".pending.json", "utf-8"));
    fs.unlinkSync(".pending.json");  // 用完即删
    // 进入扩展模式...
  }
});
```

## 10. 状态持久化：appendEntry

让扩展状态在同 session 内重启 pi 后能恢复：

```ts
function persistState() {
  pi.appendEntry("myext-state", { ...state });  // 写入 session jsonl
}

pi.on("session_start", async (_, ctx) => {
  const entries = ctx.sessionManager.getEntries();
  const saved = entries
    .filter((e) => e.type === "custom" && e.customType === "myext-state")
    .pop();
  if (saved?.data) Object.assign(state, saved.data);
});
```

## 11. extension 加载错误调试

### 报错位置
- 启动 pi 时终端会输出：`Failed to load extension "/path/to/index.ts": ...`
- 不会自动重试，加载失败的扩展整个不可用

### 常见原因
| 错误信息 | 原因 |
|---------|------|
| `ParseError: Missing semicolon` | TypeScript 语法问题，常见于 template literal 内层反引号 |
| `Cannot find module 'xxx'` | 扩展在 `~/.pi/agent/extensions/` 内，不能 require 项目本地包 |
| `Extension does not export a valid factory function` | 没有 `export default function (pi) {}` |
| `ctx.newSession is not a function` | 在事件 handler 调了，应在 command handler 调 |

### 调试技巧
- 改完扩展必须**重启 pi**（不是 `--session` 续接，要全新启动）
- 用 `console.warn` 在扩展加载时输出诊断信息（启动时可见）
- 用 `ctx.ui.notify(..., "error")` 在运行时输出错误（更可见）

## 12. agent / skill / extension / prompt 区别

| 概念 | 路径 | 角色 |
|------|------|------|
| **agent** | `~/.pi/agent/agents/*.md` | subagent 定义（planner/worker/scout 等）|
| **skill** | `~/.pi/agent/skills/*/SKILL.md` | LLM 按需读取的指令文档 |
| **extension** | `~/.pi/agent/extensions/*/index.ts` | 注册 tool/command/event 的代码 |
| **prompt** | `~/.pi/agent/prompts/*.md` | slash command 形式的 prompt 模板 |

写新功能时先想清楚是哪类——**能用 skill 就别写 extension**，extension 复杂度高很多。

## 关键文档位置

读源码比读文档快：

```
/Users/niupian/.nvm/versions/node/v22.22.3/lib/node_modules/@earendil-works/pi-coding-agent/
├── README.md              # 主文档
├── docs/                  # 各主题深入说明
└── examples/extensions/   # 官方示例（plan-mode、handoff、tps、trace 等）

# 类型定义（最权威）
src/core/extensions/types.ts  # ExtensionAPI / ExtensionContext 完整定义

# 核心实现
src/core/extensions/loader.ts  # 扩展如何加载
src/core/agent-session.ts      # session 管理
src/core/agent-session-runtime.ts  # newSession / switchSession 实现
src/core/system-prompt.ts      # system prompt 拼接逻辑
src/core/compaction/           # 上下文压缩
```

## 验证清单

新写一个扩展时按这个走一遍：

- [ ] `package.json` 的 `pi.extensions` 字段指向 `index.ts`
- [ ] `export default function (pi: ExtensionAPI)` 是同步函数返回 void
- [ ] 注册的 tool 都有 `parameters: Type.Object({...})` typebox schema
- [ ] 跨 session 数据走文件，不依赖内存 state
- [ ] event handler 不调 newSession/switchSession
- [ ] 模板字符串里没有未转义的反引号
- [ ] context filter 不依赖索引（用 customType 或 role）
- [ ] 新增模型时确认 provider 是否支持 reasoning 参数
- [ ] subagent 调用的 tool 不依赖主进程内存
- [ ] 扩展状态用 `pi.appendEntry` + `session_start` 持久化
