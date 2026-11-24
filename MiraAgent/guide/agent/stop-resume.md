> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 停止与续跑

用户点击停止时，内核停止生成；停止后历史保留，可从断点继续。

## 场景

场景一：用户在界面上点了"停止"，希望 AI 别再往下调工具。
场景二：某个工具的结果要等人工确认。先停下来，等用户点了"确认"，再从原地继续——而不是从头发一遍。

## 最小可用代码

停止机制：向顶层 Agent 设置一个停止标记。内核在每轮模型调用前与每个工具执行前各检查一次该标记，命中则停止。

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.runtime.KernelRuntime;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;
import net.itzq.mira.modules.ai.utils.ThreadPoolUtil;

import java.util.concurrent.CountDownLatch;

public class StopAndResume {

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
                .prompt("你是一个会动手读文件的助手。")
                .workspacePath("D:/workspace/demo")
                .vfsId(historyId)
                .runtime(rt)
                .build();

        AutoAgent agent = new AutoAgent(holder);
        agent.mountReadOnlyTools();

        // ===== 起一轮对话 =====
        CountDownLatch latch = agent.chatStream("把 demo 目录下每个 Java 文件都读一遍并逐个总结");

        // 模拟用户 3 秒后点了"停止"
        // 用内核自带的线程池（被 TTL 包装过），业务里的 ThreadLocal 才能带到子线程，不要自行 new Thread
        ThreadPoolUtil.getExecutorService().submit(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException ignored) {
                }
                agent.setStopped(true);   // 打停止标记：内核在工具调用间隙会看到
            }
        });

        latch.await();
        System.out.println("已停止。history 末条角色：" +
                holder.getHistory().get(holder.getHistory().size() - 1).getRole());
        System.out.println("当前深度 currentDeep = " + holder.getCurrentDeep());

        // ===== 之后续跑：先清掉停止标记，再从断点继续 =====
        agent.setStopped(false);
        CountDownLatch resumeLatch = agent.chatStreamResume();
        resumeLatch.await();
        System.out.println("续跑完成");

        rt.close();
    }
}
```

同步路径（`chat()`）对应的续跑入口是 `chatSyncResume()`：

```java
String more = agent.chatSyncResume();   // 从断点继续，返回末轮回复文本
```

## 跑起来会看到什么

- 调用 `agent.setStopped(true)` 之后，内核不会"立刻"掐断正在进行的模型请求或正在跑的工具，而是在**下一个检查点**停止：
  - 下一轮模型调用前发现标记 → 直接结束循环；
  - 已经发出去的一批工具调用里，还没执行的会被跳过，并各自补一条"该调用已中断（用户停止了本轮对话）"的工具结果。
- 停止后 `history` 停在半路：可能是带工具调用的 assistant 消息，也可能是补过占位的结果消息。如果这一轮根本没有回复被写进去，内核会补一条 assistant 消息"（本轮已被用户停止）"，保证历史不会以 user 结尾。
- 续跑时 `chatStreamResume()` **不会重复追加用户输入**，也从当前深度接着跑，所以不会从头再来。

## 换个场景怎么改

### 停止后继续（工具结果需人工确认）

如果停下来的地方是"带工具调用、还没结果的 assistant 消息"，续跑前要先把结果补上，否则上下文不完整：

```java
import net.itzq.mira.modules.ai.client.openai.chat.entity.ChatMessage;
import net.itzq.mira.modules.ai.tool.ToolCallResult;

// toolCallId 从之前的 CallToolBegin 事件里取（info 的 toolCallId 字段）
String toolCallId = "call_abc123";
holder.addHistory(ChatMessage.withTool(
        ToolCallResult.success("用户已确认，允许删除该文件"),
        toolCallId));

// 补完结果后，清标记再续跑
agent.setStopped(false);
agent.chatStreamResume();
```

### 速查表

| 目标            | 改哪儿                                                         |
| ------------- | ----------------------------------------------------------- |
| 停止            | `agent.setStopped(true)`（对顶层那个 Agent 调；子代理会跟着停）             |
| 续跑（流式）        | 清标记后调用 `agent.chatStreamResume()`，返回倒计数锁，需 `await()` 等待本轮结束 |
| 续跑（同步）        | 清标记后 `agent.chatSyncResume()`，直接返回字符串                       |
| 查是不是被要求停止了    | `holder.isStopRequested()`                                  |
| 判断"这一轮真的结束了没" | 用 `agent.isRunning()`；它只在消息落库之后才变 false                     |

## 注意事项

- 续跑前需清除停止标记（`agent.setStopped(false)`）；否则新循环首轮会立即中断。
- 停止标记是"顶层 Agent"的：子代理通过父链读同一个标记，所以主对话一停，分身也停。
- 停止为协作式：内核在检查点收手，不应强制终止线程。
- 续跑复用同一 `holder`，`historyId`、`currentDeep` 延续原值；重建 `holder` 会丢失上下文。

> 相关：[工具执行前钩子（EventHook）](/MiraAgent/guide/agent/event-hook.md)、[流式输出](/MiraAgent/guide/agent/streaming.md)、[子代理（SubAgent）](/MiraAgent/guide/agent/sub-agent.md)
