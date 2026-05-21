# 首页右侧 CodeViewer 展示代码草案

这份代码用于首页右侧 `CodeViewer` 展示，目标是**简洁表达 EdgeOne 上创建 Agent 的关键流程**，不要求直接运行。重点展示：

- `context.tools`：EdgeOne 沙箱工具；
- `context.store`：会话消息持久化；
- `claude_session_store()`：注入 Claude Agent SDK 会话记忆；
- `query()`：启动 Claude Agent。

```ts
import { query, createSdkMcpServer, tool } from '@anthropic-ai/claude-agent-sdk';
import { z } from 'zod';

const SYSTEM_PROMPT = `...`;

export async function onRequest(context: any) {
  const message = context.request.body?.message ?? '';
  const conversationId = context.conversation_id;

  // 1. EdgeOne Store：保存消息 + 提供 Agent 会话记忆
  const store = context.store;
  const sessionStore = store?.claude_session_store?.();

  // 2. EdgeOne Tools：读取平台沙箱工具
  const platformTools = context.tools?.all?.() ?? [];

  const commands = tool(
    'commands',
    'Execute shell commands in EdgeOne sandbox',
    { cmd: z.string(), cwd: z.string().optional() },
    (args) => callEdgeOneTool(platformTools, 'commands', args),
  );

  const files = tool(
    'files',
    'Read/write/list files in EdgeOne sandbox',
    {
      op: z.enum(['read', 'write', 'list', 'exists', 'remove', 'makeDir']),
      path: z.string(),
      content: z.string().optional(),
    },
    (args) => callEdgeOneTool(platformTools, 'files', args),
  );

  const code = tool(
    'code_interpreter',
    'Run code in an isolated interpreter',
    { language: z.string(), code: z.string() },
    (args) => callEdgeOneTool(platformTools, 'code_interpreter', args),
  );

  const browser = tool(
    'browser',
    'Fetch or interact with web pages',
    {
      op: z.string(),
      url: z.string().optional(),
      selector: z.string().optional(),
      text: z.string().optional(),
      script: z.string().optional(),
    },
    (args) => callEdgeOneTool(platformTools, 'browser', args),
  );

  // 3. 注册 EdgeOne MCP Server
  const edgeone = createSdkMcpServer({
    name: 'edgeone',
    tools: [commands, files, code, browser],
    alwaysLoad: true,
  });

  // 4. 创建 Agent 运行参数
  const options = {
    model: process.env.AI_GATEWAY_MODEL ?? '@Pages/hy3-preview',
    systemPrompt: SYSTEM_PROMPT,

    sessionStore,
    mcpServers: { edgeone },
    allowedTools: [
      'mcp__edgeone__commands',
      'mcp__edgeone__files',
      'mcp__edgeone__code_interpreter',
      'mcp__edgeone__browser',
    ],

    env: {
      ANTHROPIC_BASE_URL: process.env.AI_GATEWAY_BASE_URL,
      ANTHROPIC_API_KEY: process.env.AI_GATEWAY_API_KEY,
    },
  };

  // 5. 启动 Claude Agent
  const result = query({ prompt: message, options });

  // 这里省略 SSE / text_delta / tool_called 的具体处理逻辑
  const assistantText = await collectAssistantText(result);

  // 6. EdgeOne Store：保存助手回复，供历史恢复
  await store?.appendMessage?.({
    conversationId,
    role: 'assistant',
    content: assistantText,
  });

  return Response.json({ answer: assistantText });
}

async function callEdgeOneTool(tools: any[], name: string, args: unknown) {
  const item = tools.find((tool) => (tool.name ?? tool.function?.name) === name);
  const execute = item?.execute ?? item?.handler ?? item?.invoke;
  return execute?.call(item, args);
}

async function collectAssistantText(result: AsyncIterable<any>) {
  // 伪代码：消费 Claude Agent SDK 输出并拼接 assistant 文本
  return '...';
}
```

## 建议在 CodeViewer 中突出展示的流程

1. `context.store`：读写用户/助手消息；
2. `store.claude_session_store()`：为 Claude Agent SDK 注入会话记忆；
3. `context.tools.all()`：获取 EdgeOne 沙箱工具；
4. `tool()`：把 EdgeOne tools 包装成 Claude Agent SDK 工具；
5. `createSdkMcpServer()`：注册 EdgeOne MCP Server；
6. `query({ prompt, options })`：启动 Claude Agent；
7. `store.appendMessage()`：保存助手回复，支持历史恢复。
