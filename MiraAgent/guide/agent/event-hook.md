> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 工具执行前钩子（EventHook）

`EventHook` 在工具真正执行前提供一次决策机会：放行、由调用方代答、或挂起等待外部结果。

## 场景

让 AI 能干活，需满足几个条件：

1. 危险操作（比如删文件、跑 `rm`）要弹出确认框，用户点了"确认"才执行；
2. 有些工具需自行实现（走后端接口，而不是让内核去跑）；
3. 有的工具结果要等外部系统回调，先暂停，等到了再继续。

这三件事都落在同一个点上：`EventHook.onBeforeToolCall`。

## 最小可用代码

`onBeforeToolCall` 是内核**唯一的工具行为决策点**。它在"工具开始事件已经发出、工具本身还没执行"时被调用，返回三种结果之一：

| 返回                          | 含义                                   |
| --------------------------- | ------------------------------------ |
| `null`                      | 放行，工具照常执行                            |
| `HookResult.respond("...")` | 拦截，**由调用方代答**：工具本身不执行，所给的这句话直接作为工具结果 |
| `HookResult.suspend()`      | 拦截并**挂起**：结果还没定，对话立刻中断，保留断点等待补结果     |

以下是一个"人工确认删除文件"的完整例子。

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.runtime.KernelRuntime;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;
import net.itzq.mira.modules.ai.agent.event.EventHook;
import net.itzq.mira.modules.ai.agent.event.HookResult;
import net.itzq.mira.modules.ai.agent.event.type.CallToolBeginEvent;
import net.itzq.mira.modules.ai.client.openai.chat.entity.ChatMessage;
import net.itzq.mira.modules.ai.tool.ToolCallResult;

import java.util.concurrent.CountDownLatch;

public class ConfirmDelete {

    // 本次待确认的工具调用（真实系统里应放进会话状态，别用 static）
    static volatile String pendingToolCallId = null;
    static volatile String pendingArgs = null;

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
                .prompt("你是一个能操作本地文件的助手。")
                .workspacePath("D:/workspace/demo")
                .vfsId(historyId)
                .runtime(rt)
                .build();

        AutoAgent agent = new AutoAgent(holder);
        agent.mountFullCodeTools();

        // ===== 安装钩子：危险操作挂起，等人工确认 =====
        holder.setEventHook(new EventHook() {
            @Override
            public HookResult onBeforeToolCall(CallToolBeginEvent event) {
                String funName = event.getToolCall().getFunction().getName();
                String arguments = event.getToolCall().getFunction().getArguments();

                // 这里只是示例判定；真实判定标准完全由实现方自行决定
                boolean dangerous = "Bash".equals(funName)
                        && arguments != null && arguments.contains("rm ");

                if (dangerous) {
                    pendingToolCallId = event.getToolCall().getId();   // 记下来，恢复时要用
                    pendingArgs = arguments;
                    return HookResult.suspend();                       // 挂起
                }
                return null;                                           // 其他工具放行
            }
        });

        CountDownLatch latch = agent.chatStream("删掉 demo 下的 old.txt");
        latch.await();

        // ===== 检查是否挂起 =====
        if (holder.isSuspended()) {
            System.out.println("需要人工确认，待执行的命令：" + pendingArgs);

            // 真实系统里这里换成"等用户点确认"（前端回传 / 阻塞队列 / 回调）
            boolean confirmed = askUser();

            String result = confirmed ? "用户已确认，允许执行该操作" : "用户已拒绝，取消这次操作";
            holder.addHistory(ChatMessage.withTool(
                    ToolCallResult.success(result),
                    pendingToolCallId));                 // 把结果补进上下文

            holder.setSuspended(false);                  // 复位挂起标志
            agent.setStopped(false);                     // 顺手清停止标记（若有）
            CountDownLatch resumeLatch = agent.chatStreamResume();
            resumeLatch.await();                         // 从断点继续
        }

        rt.close();
    }

    static boolean askUser() {
        return true;   // 换成真实的用户交互
    }
}
```

由调用方代答（不执行内核工具本身）时返回 `respond`：

```java
holder.setEventHook(new EventHook() {
    @Override
    public HookResult onBeforeToolCall(CallToolBeginEvent event) {
        String funName = event.getToolCall().getFunction().getName();
        if ("queryOrder".equals(funName)) {
            // 工具本身不会执行，这句话直接作为工具结果回灌给模型
            return HookResult.respond("订单 SO20260101 当前状态：已发货");
        }
        return null;
    }
});
```

## 跑起来会看到什么

- 命中危险判定的工具：前端会先收到 `CallToolBegin`（所以可以渲染一张"确认卡片"），但**工具不执行**，`chatStream` 提前回来；
- 此时 `holder.isSuspended()` 为 `true`，历史停在"带工具调用的 assistant 消息"（这是一个合法断点）；
- 补上工具结果、复位标志、调 `chatStreamResume()` 之后，模型拿着"用户已确认/已拒绝"这句话继续往下回答。

## 换个场景怎么改

| 目标        | 改哪儿                                                                              |
| --------- | -------------------------------------------------------------------------------- |
| 危险操作需用户确认 | `onBeforeToolCall` 里判定命中 → `HookResult.suspend()`，等确认后补结果 + `chatStreamResume()` |
| 某些工具走后端实现 | 委托条件 → `HookResult.respond(后端返回的结果)`，工具本身不执行                                     |
| 暂停等外部回调   | 同"人工确认"：`suspend()` → 外部结果到了补进历史 → `chatStreamResume()`                          |
| 顺手记日志     | 覆盖 `EventHook` 的 `onCallToolBegin` / `onCallToolEnd`（这些是默认空实现的观察点）               |
| 挂起后走同步续跑  | 用 `agent.chatSyncResume()` 代替 `chatStreamResume()`                               |

## 注意事项

- `onBeforeToolCall` 每个工具调用各调一次，不知道具体工具是谁——判定标准完全由实现方编写。
- 钩子抛异常时，内核按"放行"处理（只记一条 warning），不会阻断整轮对话。
- 挂起后需复位 `holder.setSuspended(false)` 再续跑，否则新一轮跑完一个来回会立即中断。
- 代答文本被当作工具结果原样回灌给模型；以 `ToolCallResult.success(...)` / `error(...)` 包装可明确表达成败状态。

> 相关：[停止与续跑](/MiraAgent/guide/agent/stop-resume.md)、[工具调用](/MiraAgent/guide/agent/tool-calling.md)、[内置工具一览](/MiraAgent/guide/agent/builtin-tools.md)
