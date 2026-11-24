> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 基础对话

本篇介绍如何用几十行代码把内核跑起来，实现一问一答，并且追问时能延续上一轮的内容。

## 场景

以一个可聊天的小工具为例：接收用户输入的一句话，将回答打印出来。需要支持多轮——第二轮要能基于第一轮的内容继续回应。

## 最小可用代码

最小可用的结构包含三个部分：内核运行时（持有模型、工具、会话文件等资产）、一次会话所需的上下文，以及承载对话的 Agent。

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.runtime.KernelRuntime;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;

import java.util.concurrent.CountDownLatch;

public class FirstChat {
    
    // 配置快照：这次运行里 "有什么"。至少要有 1 个对话模型，并用 agentConfig.defaultModel 指定默认用哪个
    // 这里的json不需要自己写，由程序自动导出，通常用于跨进程执行。
    // 从代码组装参数参考“快速开始”
    private static final String DECLARATION_JSON =
            "{\n"
            + "  \"schemaVersion\": \"1.0.0\",\n"
            + "  \"models\": [{\n"
            + "    \"alias\": \"deepseek-v3\",\n"
            + "    \"providerName\": \"deepseek\",\n"
            + "    \"apiModelName\": \"deepseek-chat\",\n"
            + "    \"apiHost\": \"https://api.deepseek.com\",\n"
            + "    \"apiEndpoint\": \"/chat/completions\",\n"
            + "    \"apiKey\": \"sk-替换为实际密钥\"\n"
            + "  }],\n"
            + "  \"agentConfig\": { \"defaultModel\": \"deepseek-v3\" }\n"
            + "}";

    public static void main(String[] args) throws Exception {
        // 1) 建运行时：起名字 + 指定数据目录
        KernelRuntime rt = KernelRuntime.builder()
                .name("my-app")
                .dataDir("D:/my-app/data")
                .declarationJson(DECLARATION_JSON)
                .build();
        // 必须在第一次读配置之前装好，否则静态便捷入口会自己懒建一个没有数据目录的运行时
        KernelRuntime.installDefault(rt);
        rt.start();

        // 2) 建"这一次对话"的上下文
        String historyId = IdGen.uuid();          // 32 位无连字符的会话 id
        AgentContextHolder holder = AgentContextHolder.builder()
                .userId("u1")
                .historyId(historyId)
                .modelAlias("deepseek-v3")        // 对应配置快照里模型的 alias
                .prompt("你是一个严谨、简洁的中文助手。")
                .workspacePath("D:/workspace/demo")
                .vfsId(historyId)                 // 会话附件空间，可直接复用 historyId
                .runtime(rt)
                .build();

        // 3) 建 Agent
        AutoAgent agent = new AutoAgent(holder);

        // 4) 同步问一句：直接返回字符串，阻塞到回答完
        String answer = agent.chat("你好，用一句话介绍你自己");
        System.out.println("同步回答：" + answer);

        // 5) 流式问一句：返回一个倒计数锁，必须 await 等它跑完
        CountDownLatch latch = agent.chatStream("再讲讲你能帮我做什么");
        latch.await();

        // 6) 多轮追问：同一个 holder，历史已经接上了，直接问就行
        String followUp = agent.chat("那我刚才一共问了你几个问题？分别是什么？");
        System.out.println("追问回答：" + followUp);

        rt.close();
    }
}
```

若需延续之前保存过的会话，可在新建 Agent 前把历史加载回上下文：

```java
import java.util.List;
import net.itzq.mira.modules.ai.client.openai.chat.entity.ChatMessage;

// 从持久化实现里读回这个 historyId 的历史
List<ChatMessage> saved = rt.persistencePort().loadContext(historyId);
holder.addHistory(saved.toArray(new ChatMessage[0]));
// 之后再 new AutoAgent(holder).chat(...) 就会带着旧历史继续说
```

## 跑起来会看到什么

- 第 4 步：控制台在回答完整生成后才打印 `同步回答：...`，内容是一整段字符串。
- 第 5 步：`chatStream` 立即返回（不阻塞），`latch.await()` 之后这一轮才算真正结束；正文在这期间分段生成，但本页未注册监听器，因此看不到逐字输出，只能等待结束。关于逐段输出，见[流式输出](/MiraAgent/guide/agent/streaming.md)。
- 第 6 步：追问能准确说出前面两个问题，因为这两条 user 消息与对应的 assistant 回复都保留在 `holder` 的历史里。

## 换个场景怎么改

| 目标                | 改哪儿                                                           |
| ----------------- | ------------------------------------------------------------- |
| 换个模型              | 改 `modelAlias("deepseek-v3")`，同时确保配置快照的 `models` 里有这个 alias   |
| 换数据目录             | 改 `dataDir("...")`；目录要可写，会话附件、技能展开都落在这里                       |
| 只做同步问答            | 删掉第 5 步，只用 `agent.chat(...)`                                  |
| 只做流式              | 删掉第 4、6 步，用 `agent.chatStream(...)` + `latch.await()`         |
| 每个用户一条独立会话        | 每个用户生成各自的 `historyId`（`IdGen.uuid()`）和 `userId`，各建一个 `holder` |
| 消息持久化             | 给运行时挂持久化实现，见[消息持久化](/MiraAgent/guide/data/persistence.md)     |
| 调系统人设             | 改 `prompt("...")`                                             |
| 换工作目录（AI 读写文件的地方） | 改 `workspacePath("...")`                                      |

## 使用要点

- `KernelRuntime.installDefault(rt)` 需要在首次读取配置前完成；顺序相反时会得到一个空的模型列表。
- `historyId`、`vfsId` 需为 32 位无连字符的十六进制串，由 `IdGen.uuid()` 生成；带连字符的 UUID 不被接受。
- 流式入口返回的是 `CountDownLatch`，需要调用 `await()` 等待本轮结束；若未 `await`，`main` 会提前返回并退出进程，导致回答丢失。
- `holder` 非线程安全，同一会话的多次调用需串行；关于用户点击"停止"后继续对话的用法，见[停止与续跑](/MiraAgent/guide/agent/stop-resume.md)。

> 相关：[流式输出](/MiraAgent/guide/agent/streaming.md)、[工具调用](/MiraAgent/guide/agent/tool-calling.md)、[消息持久化](/MiraAgent/guide/data/persistence.md)
