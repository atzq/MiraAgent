> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 接入 MCP 工具服务

已有的 MCP 服务接入内核后，AI 可调用其提供的工具，无需自行编写工具代码。

## 场景

示例：代码助手需要读写本地文件。这类能力已有现成的 MCP 服务实现，通过一段配置即可接入，无需重写。

内核通过 **MCP 支持** 实现：提供一段描述服务的 JSON，内核连接一次并列出服务提供的工具；之后 AI 需要调用时，通过统一的 `mcp_call` 工具发起。

## 最小可用代码

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.core.utils.PropsMap;
import net.itzq.mira.core.utils.PromptLoader;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;
import net.itzq.mira.modules.ai.mcp.McpManager;
import net.itzq.mira.modules.ai.mcp.McpPrepared;
import net.itzq.mira.modules.ai.mcp.McpToolInfo;
import net.itzq.mira.modules.runtime.KernelRuntime;

import java.util.LinkedHashSet;
import java.util.Set;

public class McpDemo {
    public static void main(String[] args) throws Exception {
        KernelRuntime rt = KernelRuntime.builder()
                .name("my-app")
                .dataDir("D:/my-app/data")
                .build()
                .start();

        // 1. 一段 MCP 服务配置（JSON 字符串）。这里用命令行方式（stdio）拉起一个本地服务
        String mcpJson = "{\"mcpServers\":{"
                + "\"filesystem\":{"
                + "\"command\":\"npx\","
                + "\"args\":[\"-y\",\"@modelcontextprotocol/server-filesystem\",\"D:/projects/demo\"]"
                + "}}}";

        // 2. 探活：连上去 -> 列出它提供的工具 -> 断开。得到"这次能用哪些工具"
        McpPrepared prepared = McpManager.prepare(mcpJson);
        for (McpToolInfo t : prepared.getTools()) {
            System.out.println("可用: " + t.getFullName() + " -> " + t.getDescription());
        }

        // 3. 会话上下文里带上这份配置
        AgentContextHolder holder = AgentContextHolder.builder()
                .userId("u1")
                .historyId(IdGen.uuid())
                .modelAlias("deepseek-v3")
                .runtime(rt)
                .build();
        holder.setMcpConfig(prepared);

        // 4. 白名单 = 系统基线工具 + mcp_call（有 MCP 工具时才会加进去）
        Set<String> whitelist = new LinkedHashSet<>(AutoAgent.systemBaselineTools(null));
        AutoAgent.mountSkillAndMcpTools(whitelist, null, prepared);

        AutoAgent agent = new AutoAgent(holder);
        agent.mountToolsIfAbsent(whitelist.toArray(new String[0]));

        // 5. 把 MCP 工具清单写进系统提示词，AI 才知道有哪些工具可以调
        PropsMap params = AutoAgent.buildSkillMcpPromptParams(null, prepared);
        holder.setPrompt(PromptLoader.processTemplate(
                PromptLoader.createTemplate("你是一个助手。\n\n可用 MCP 工具:\n${mcpTools}"),
                params));

        // 6. 开聊：AI 会调 mcp_call，内核按需现连服务再执行
        agent.chatStream("看看 D:/projects/demo 下都有什么文件").await();
    }
}
```

### 补充说明

- 每个 MCP 工具都有一个全名，形如 `mcp__服务名__工具名`，例如 `mcp__filesystem__read_file`。这个名字在探活时就能拿到。

- 换个连法只有一段 JSON 的差别。HTTP 方式（SSE）长这样：

  `{"mcpServers":{"weather":{"url":"http://localhost:3001/sse"}}}`

- 一次挂多个服务，就是在 `mcpServers` 里写多个键，各自一套 `command`/`args` 或 `url`。命令行方式还可以加 `"env":{"KEY":"值"}` 给子进程传环境变量。

## 跑起来会看到什么

- `McpManager.prepare(...)` 会实际启动或连接服务，并在控制台打印其提供的工具全名与描述。
- 对话时，AI 先调 `mcp_call`，参数是 `tool_name`（工具全名）和 `arguments`（一个 JSON 字符串形式的参数对象）；内核按放入会话上下文的那份配置现连服务、把参数递过去执行、拿到结果再断开，最终把结果以 JSON 文本返回给模型。
- 服务无法启动时，探活这一步会列不出工具（列表为空），后续 AI 也不会去调。

## 换个场景怎么改

| 目标               | 改哪里                                                                  |
| ---------------- | -------------------------------------------------------------------- |
| 从命令行方式换成 HTTP 方式 | 把服务配置里的 `command`/`args` 换成 `url`                                    |
| 一次挂多个服务          | 在 `mcpServers` 里加多个服务名，每个写自己的 `command`/`args` 或 `url`               |
| 给服务传密钥或环境变量      | 命令行方式在服务配置里加 `"env":{"KEY":"值"}`；密钥不写死在代码里                           |
| 不同用户/会话挂不同的服务    | 每次会话各自 `McpManager.prepare(...)` 一份，写进自己的 `holder.setMcpConfig(...)` |
| 只让 AI 用其中几个工具    | 内核层做不到：白名单里只有统一的 `mcp_call`，要收窄只能在服务端裁剪，或拆成不同服务挂                     |

## 使用要点

- **配置需落到会话上下文**：`mcp_call` 从 `holder.getMcpConfig()` 读取配置；未调用 `setMcpConfig` 时，AI 调工具会收到"本次对话未配置MCP"。
- **一次调用一次连接**：内核不常驻服务连接，每次 `mcp_call` 都按配置现连。本地命令行服务存在进程启动开销，网络服务需关注超时，避免在高峰期频繁调用。
- **探活即实际连接**：`prepare` 非纯本地解析，会实际连接服务并列出工具，因此可能失败或较慢，宜放在会话准备阶段，而非对话主流程。
- **JSON 为字符串**：配置需注意转义；密钥从环境变量或配置快照读取，不写入代码。
- **工具名分隔符**：全名以双下划线分段，拼接时需保持一致。

> 相关：[技能包（Skills）](/MiraAgent/guide/extensions/skills.md) · [工具调用](/MiraAgent/guide/agent/tool-calling.md) · [配置中心](/MiraAgent/guide/config/config-center.md)
