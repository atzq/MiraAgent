> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 流式输出

流式输出让用户看到逐字生成的正文，并呈现"正在思考""正在调用工具"等状态。

## 场景

目标是做一个聊天界面：模型正文逐字出现；模型在思考时显示"思考中"；模型要调工具时显示"正在读取文件"，调完显示一条结果。这些时刻内核都会通过事件说明。

## 最小可用代码

内核把运行过程拆成事件往外发。注册一个监听器（内核已经将原始数据翻译成统一的 `AnsResponse`），即可拿到每一个时刻。

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.runtime.KernelRuntime;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;
import net.itzq.mira.modules.ai.agent.event.DefaultEventListener;
import net.itzq.mira.modules.ai.agent.event.EventCenter;
import net.itzq.mira.modules.ai.agent.event.type.ErrorEvent;
import net.itzq.mira.modules.ai.entity.chat.AnsResponse;

import java.util.concurrent.CountDownLatch;

public class StreamChat {

    /** 监听器：内核把内部事件翻译成 AnsResponse 后交给调用方 */
    public static class ConsoleListener extends DefaultEventListener {
        // 必须实现 1：每收到一个 AnsResponse 就调一次（正文增量、思考增量、步骤、工具、结束都走这里）
        @Override
        public void sseEmitterSend(Object response) throws Exception {
            AnsResponse ans = (AnsResponse) response;
            String type = ans.getType();
            if ("ChatContent".equals(type)) {
                System.out.print(ans.getAnswer());                 // 正文增量，直接往屏幕上贴
            } else if ("ChatReasonContent".equals(type)) {
                System.out.print("[思考] " + ans.getAnswer());
            } else if ("CallToolBegin".equals(type)) {
                System.out.println("\n[开始调用工具] " + ans.getInfo().get("toolName"));
            } else if ("CallToolEnd".equals(type)) {
                System.out.println("[工具调用结束] " + ans.getInfo().get("toolName"));
            } else if ("ChatEnd".equals(type)) {
                System.out.println("\n[这一轮结束]");
            }
        }

        // 必须实现 2：出错时调用
        @Override
        public void onError(ErrorEvent event) {
            System.out.println("\n[出错] " + event.getThrowable());
        }
    }

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
                .prompt("你是一个简洁的中文助手。")
                .workspacePath("D:/workspace/demo")
                .vfsId(historyId)
                .runtime(rt)
                .build();

        // 注册监听器：一个会话一个事件中心
        EventCenter eventCenter = new EventCenter();
        eventCenter.register(new ConsoleListener());
        holder.setEventCenter(eventCenter);

        AutoAgent agent = new AutoAgent(holder);

        CountDownLatch latch = agent.chatStream("帮我看一下 D:/workspace/demo 下有哪些 Java 文件");
        latch.await();   // 等这一轮真正跑完
        rt.close();
    }
}
```

## 跑起来会看到什么

控制台上会依次出现：

1. 正文（或多个）增量片段，直接拼接成一句话；
2. 如果模型先"想"了一下，会有 `[思考] ...` 的片段；
3. `[开始调用工具] 文件列表` → `[工具调用结束] 文件列表`（中间可能有好几对，取决于模型调了几次工具）；
4. 模型根据工具结果继续输出正文；
5. 最后一行 `[这一轮结束]`。

## 换个场景怎么改

### 推送给浏览器（SSE）

要点只有三条：

1. **在哪一层做**：在对外暴露的 HTTP 接口层做 SSE，不改动内核。监听器收到 `AnsResponse` 后，将其转为一条 SSE 消息写入当前连接。`ConsoleListener` 替换为 `SseListener`，把 `System.out` 改为写入 `SseEmitter`/响应流。
2. **每个事件带上 id**：给每条 SSE 消息一个自增序号（或直接用 `ans.getRoundId() + ":" + ans.getMsgId()`），前端断线重连时会带上 `Last-Event-ID`，据此从持久化里把之后的事件补推一遍，避免丢消息。
3. **先落库再推**：收到事件后先把 `AnsResponse` 存下来，再推给当前客户端。这样刷新页面、换设备都能从历史里重建整个时间线。

前端按 `type` 分流渲染：

| type                            | 前端怎么用                                            |
| ------------------------------- | ------------------------------------------------ |
| `UserInput`                     | 把用户这条消息固定到列表顶部                                   |
| `ChatReasonContent`             | 折叠的"思考过程"区域，边收边追加                                |
| `ChatContent`                   | 正文气泡，边收边追加                                       |
| `StepBegin` / `StepEnd`         | 可选："第 N 轮"分隔条（`info.deep` 是轮次）                   |
| `CallToolBegin` / `CallToolEnd` | 工具调用时间线（`info.toolName` 展示名、`info.toolId` 定位同一条） |
| `ChatEnd`                       | 收尾：把输入框解锁、光标停掉                                   |
| `Error`                         | 红色错误提示                                           |

### 只看原始数据

演示案例较为简单，它把细节压成了字符串。若要自行掌控，直接实现 `EventListener` 的 `onEvent`，拿到的是模型原始的流式数据：

```java
import net.itzq.mira.modules.ai.agent.event.EventListener;
import net.itzq.mira.modules.ai.agent.event.type.GeneralEvent;
import net.itzq.mira.modules.ai.client.handle.StreamEventHandler;

eventCenter.register(new EventListener() {
    @Override
    public void onEvent(GeneralEvent event) {
        StreamEventHandler handler = event.getHandler();
        if (handler == null || !handler.isChatAnswer()) {
            return;   // 不是正文/思考的流，跳过
        }
        String delta = handler.getCurrStr();          // 当前这段文本
        System.out.print((handler.isReasoning() ? "[思考]" : "[正文]") + delta);
    }
});
```

`EventListener` 除了 `onEvent`，还有 `onChatInput`、`onStepBegin`、`onStepEnd`、`onCallToolBegin`、`onCallToolEnd`、`onChatEnd`、`onError` 这些方法，都是 `default` 空实现，按需覆盖即可。

## 注意事项

- 两个监听器二选一即可：`DefaultEventListener` 是默认示例封装（已翻译好），`EventListener` 是"原始版"（提供原始数据）。前者本身也实现了后者，所以注册哪个都行。
- `DefaultEventListener` 需实现两个方法 `sseEmitterSend(Object)` 与 `onError(ErrorEvent)`，缺一则编译失败。
- 要在一次对话中推送事件，需先将 `EventCenter` 注册进 `holder`（`holder.setEventCenter(...)`）；未注册时事件被丢弃，`chatStream` 仍跑完但无过程可见。
- 同步入口 `chat()` 不发流式增量事件，只有步骤/工具/结束这些主体事件。

> 相关：[基础对话](/MiraAgent/guide/agent/basic-chat.md)、[工具调用](/MiraAgent/guide/agent/tool-calling.md)、[消息持久化](/MiraAgent/guide/data/persistence.md)
