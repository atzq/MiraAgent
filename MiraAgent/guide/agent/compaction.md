> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 长对话压缩（ContextCompactor）

对话轮数增多后，历史变长、token 开销上升。内核将较早的对话折叠为一段摘要，仅保留最近内容。

## 场景

用户跟 AI 已经来回聊了 40 轮，还带了一堆工具调用记录。再问一句，光历史就快把模型的上下文占满了。目标是将"很久以前"的对话压成一段摘要，最近的对话保留原文，然后接着聊。

## 最小可用代码

内核提供了 `ContextCompactor`：输入一份消息列表，输出压缩后的列表。它用**双预算**决定"保护到哪里"：

- `protectedTurns`：至少保护最近多少轮（按 user 提问数轮）；
- `protectedTokens`：至少保护最近多少 token；
- **最后一轮永远保护**，不管预算多小。

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.runtime.KernelRuntime;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;
import net.itzq.mira.modules.ai.compact.ContextCompactor;
import net.itzq.mira.modules.ai.client.openai.chat.entity.ChatMessage;

import java.util.List;

public class CompactChat {

    public static void main(String[] args) throws Exception {
        KernelRuntime rt = KernelRuntime.builder()
                .name("my-app")
                .dataDir("D:/my-app/data")
                .declarationJson("{\"schemaVersion\":\"1.0.0\",\"models\":[{\"alias\":\"deepseek-v3\","
                        + "\"providerName\":\"deepseek\",\"apiModelName\":\"deepseek-chat\","
                        + "\"apiHost\":\"https://api.deepseek.com\",\"apiEndpoint\":\"/chat/completions\","
                        + "\"apiKey\":\"sk-替换为实际密钥\"}],"
                        + "\"agentConfig\":{\"defaultModel\":\"deepseek-v3\"}}")
                .build();
        KernelRuntime.installDefault(rt);
        rt.start();

        String historyId = IdGen.uuid();
        AgentContextHolder holder = AgentContextHolder.builder()
                .userId("u1")
                .historyId(historyId)
                .modelAlias("deepseek-v3")
                .prompt("你是一个耐心的中文助手。")
                .runtime(rt)
                .build();

        AutoAgent agent = new AutoAgent(holder);

        // ... 这里先聊很多轮（agent.chat(...) 若干次）...

        // ===== 压缩：保护最近 10 轮 / 8192 token，其余折叠成摘要 =====
        List<ChatMessage> messages = holder.getHistory();
        ContextCompactor.Outcome outcome = ContextCompactor.compact(
                messages,
                10,      // protectedTurns：保护最近 10 轮
                8192,    // protectedTokens：保护最近 8192 token
                null,    // 摘要器：null = 用内建机械摘要
                null);   // token 估算器：null = 用内建估算

        if (outcome.compacted()) {
            // 1) 用压缩后的上下文替换内存里的历史
            holder.setHistory(outcome.messages());
            // 2) 落库：覆盖写回同一个 historyId（先压缩，再覆盖持久化）
            rt.persistencePort().persistContext(historyId, outcome.messages());

            System.out.println("已折叠 " + outcome.archivedCount() + " 条旧消息；"
                    + "保护最近 " + outcome.protectedTurns() + " 轮；"
                    + "摘要方式 = " + outcome.summaryType());
        }

        // 压缩后接着问，历史已经变短了
        String answer = agent.chat("接着刚才的话题，继续讲");
        System.out.println(answer);

        rt.close();
    }
}
```

使用大模型生成摘要（而非内建机械拼接）时，向 `compact(...)` 传入一个 `ContextCompactor.Summarizer`：

```java
ContextCompactor.Summarizer summarizer = new ContextCompactor.Summarizer() {
    @Override
    public String summarize(List<ChatMessage> archived) {
        // 把 archived 这批旧消息交给模型总结，返回一段摘要正文
        // 返回 null / 空字符串 / 抛异常，都会自动降级为内建机械摘要
        return mySummarize(archived);
    }
};

ContextCompactor.Outcome outcome = ContextCompactor.compact(
        holder.getHistory(), 10, 8192, summarizer, null);
```

## 跑起来会看到什么

- `outcome.compacted()` 为 `true` 表示真的折了一部分；为 `false` 表示预算已经覆盖全部内容，原样返回；
- `outcome.summaryType()` 是 `llm`（模型摘要）/ `mechanical`（机械摘要）/ `none`（没压）；
- 折叠进去的旧对话会变成一段"之前的对话历史已压缩为以下摘要……"的提示，并**并进保护区第一轮 user 消息里**，不会额外多出一条孤立的摘要行；
- 之后模型看到的历史明显变短，但最近 10 轮原文还在，对话连贯性不受影响。

## 触发方式

内核不会代为定时压缩，什么时候调由调用方决定：

| 方式 | 怎么做                                                                  |
| -- | -------------------------------------------------------------------- |
| 手动 | 在认为需要压缩时调一次 `ContextCompactor.compact(...)`（比如用户点了"清理上下文"，或计数到轮数超阈值） |
| 自动 | 挂一个事件监听器，在每轮 `ChatEnd` 之后检查当前历史长度/轮数，超阈值就压一次                         |

压缩策略可在配置快照中配置，对应键 `defaults.session.compact`，其下三个子键：

| 键                 | 含义                      |
| ----------------- | ----------------------- |
| `autoEnabled`     | 是否自动压缩（默认 `false`）      |
| `protectedTurns`  | 保护最近多少轮（默认 `10`）        |
| `protectedTokens` | 保护最近多少 token（默认 `8192`） |

读出来用：

```java
import java.util.Map;

@SuppressWarnings("unchecked")
private static int[] readCompactBudget(Map<String, Object> defaults) {
    int turns = 10;
    int tokens = 8192;
    Object session = defaults.get("session");
    if (session instanceof Map) {
        Object compact = ((Map<String, Object>) session).get("compact");
        if (compact instanceof Map) {
            Map<String, Object> c = (Map<String, Object>) compact;
            if (c.get("protectedTurns") instanceof Number) {
                turns = ((Number) c.get("protectedTurns")).intValue();
            }
            if (c.get("protectedTokens") instanceof Number) {
                tokens = ((Number) c.get("protectedTokens")).intValue();
            }
        }
    }
    return new int[] { turns, tokens };
}

// 用法
int[] budget = readCompactBudget(rt.declaration().getDefaults());
ContextCompactor.Outcome outcome = ContextCompactor.compact(
        holder.getHistory(), budget[0], budget[1], null, null);
```

## 换个场景怎么改

| 目标        | 改哪儿                                                                                      |
| --------- | ---------------------------------------------------------------------------------------- |
| 保护更多/更少   | 调 `compact(...)` 的两个预算参数（或改 `defaults.session.compact` 后读出来用）                            |
| 用模型写摘要    | 传一个 `ContextCompactor.Summarizer` 进去                                                     |
| 自己估 token | 传一个 `ContextCompactor.TokenEstimator` 进去（不传就用内建：中文按 1 字 ≈ 1 token，其他按 4 字符 ≈ 1 token 估算） |
| 压缩后不落库    | 跳过 `persistContext(...)`，只 `holder.setHistory(...)`（下次装载又会变回没压的状态）                       |
| 只折叠工具记录   | 目前是"按轮折叠"，工具记录跟着它所在的轮一起被折叠                                                               |

## 注意事项

- `compact(...)` 要求上下文里**至少有一条 user 消息**，否则会抛 `IllegalStateException`；空列表也一样。
- 压缩后要**同时**更新内存和持久化：先 `holder.setHistory(outcome.messages())`，再 `persistContext(historyId, ...)`。只压内存不落库，重启后又变长。
- 摘要器失败不会阻断压缩，自动降级为机械摘要；模型那一步失败不影响整体压缩。
- 无特殊需求时向摘要器与估算器传 `null`，使用内建实现。

> 相关：[子代理（SubAgent）](/MiraAgent/guide/agent/sub-agent.md)、[消息持久化](/MiraAgent/guide/data/persistence.md)、[基础对话](/MiraAgent/guide/agent/basic-chat.md)
