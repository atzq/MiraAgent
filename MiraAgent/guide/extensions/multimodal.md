> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 图片输入

图片作为对话消息的一部分发送给模型，模型在回答中描述图片内容。

## 场景

示例：用户贴一张报错截图并询问错误原因；图片随问题一同发送给模型，模型在回答中描述图片内容。

图片理解不是独立能力，而是对话的一部分：图片作为消息发出，其余流程与普通对话一致。

## 最小可用代码

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.runtime.KernelRuntime;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;
import net.itzq.mira.modules.ai.client.openai.chat.entity.ChatMessage;

import java.util.Arrays;
import java.util.List;

public class ImageChatDemo {

    // 看图要选支持视觉的对话模型（配置里 supportImage 为 1）
    private static final String DECLARATION_JSON =
            "{\"schemaVersion\":\"1.0.0\","
            + "\"models\":[{\"alias\":\"gpt-4o\",\"providerName\":\"openai\","
            + "\"apiModelName\":\"gpt-4o\",\"apiHost\":\"https://api.openai.com\","
            + "\"apiEndpoint\":\"/v1/chat/completions\",\"supportImage\":1,"
            + "\"apiKey\":\"sk-替换为实际密钥\"}],"
            + "\"agentConfig\":{\"defaultModel\":\"gpt-4o\"}}";

    public static void main(String[] args) throws Exception {
        KernelRuntime rt = KernelRuntime.builder()
                .name("my-app")
                .dataDir("D:/my-app/data")
                .declarationJson(DECLARATION_JSON)
                .build();
        KernelRuntime.installDefault(rt);
        rt.start();

        AgentContextHolder holder = AgentContextHolder.builder()
                .userId("u1")
                .historyId(IdGen.uuid())
                .modelAlias("gpt-4o")           // 必须是支持视觉的对话模型
                .prompt("你是一个严谨、简洁的中文助手。")
                .runtime(rt)
                .build();

        AutoAgent agent = new AutoAgent(holder);

        // 图片既可以是公网 URL，也可以是 data:image/png;base64,xxxx 这种内联数据
        List<String> images = Arrays.asList(
                "https://example.com/error.png",
                "data:image/png;base64,iVBORw0KGgo...");
        agent.chatStream("这张截图里报的什么错？", images).await();

        // 也可以把图片消息直接塞进历史
        holder.addHistory(ChatMessage.withUser("再看看这张", "https://example.com/error2.png"));
        agent.chatStream("这两张图说的是一回事吗？").await();

        rt.close();
    }
}
```

## 跑起来会看到什么

- 两轮都不调任何工具，模型直接在回答里描述图片内容。
- 第一轮回答会说清截图上的报错信息。
- 第二轮因为历史里带着第二张图，模型能同时看到两张图并做对比。
- 本页未注册监听器，控制台没有逐字输出；分段输出的处理方式见[流式输出](/MiraAgent/guide/agent/streaming.md)。

## 换个场景怎么改

| 目标                | 改哪里                                                                                                     |
| ----------------- | ------------------------------------------------------------------------------------------------------- |
| 换服务商 / 换网关        | 改配置快照里这个模型的 `apiHost` + `apiEndpoint` + `apiKey`；网关要是要求特殊请求头，用 `apiHeaders` 接管                          |
| 换成别的视觉模型          | 改 `modelAlias("...")`，同时把配置快照 `models` 里的 alias 对上                                                      |
| 用户上传的图片，而不是公网 URL | 本地文件读成 base64，拼成 `data:image/png;base64,...` 再传，和公网 URL 等价                                              |
| 一次发多张图            | `images` 里多放几个元素，顺序不影响                                                                                  |
| 只做纯文本对话           | 不传 `images`，用 `agent.chat(...)` 或 `agent.chatStream(...)`，见[基础对话](/MiraAgent/guide/agent/basic-chat.md) |
| 每个用户一条独立会话        | 各自生成 `historyId` 和 `userId`，各建一个 holder                                                                 |

## 使用要点

- **取决于模型能力**：发给不支持视觉的模型时，请求会失败或被忽略。配置快照中该模型的 `supportImage` 设为 `1`（不填默认按支持处理）。
- **流式入口需 await()**：`chatStream` 返回倒计数锁；未调用 `await()` 时 `main` 会提前退出，回答丢失。
- **密钥不写入代码**：从环境变量或配置快照读取值，不写死在代码中。

> 相关：[基础对话](/MiraAgent/guide/agent/basic-chat.md) · [流式输出](/MiraAgent/guide/agent/streaming.md) · [接入一个模型](/MiraAgent/guide/config/models.md)
