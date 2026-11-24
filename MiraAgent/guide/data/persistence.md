> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 消息持久化

本篇说明如何将会话的历史消息与过程事件持久化，以实现会话列表、历史记录，以及刷新页面后仍能恢复答案。

## 场景

某对话应用存在如下情况：刷新页面后，之前的问答丢失；也没有可供继续的会话列表。将每轮消息落库，并把过程中产生的思考、工具调用等事件按序存储，即可支持回放。

内核将"是否存储、存储位置"交由调用方决定：只依赖一个接口 `PersistencePort`。由调用方实现，内核在相应时机回调；不实现时，会话仅存在于内存，运行结束即丢弃。

## 最小可用代码

先用内存版验证链路（尚未建库也可执行）：

```java
import net.itzq.mira.modules.ai.client.openai.chat.entity.ChatMessage;
import net.itzq.mira.modules.ai.entity.chat.AnsResponse;
import net.itzq.mira.modules.ai.persistence.PersistencePort;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

/** 用内存 Map 实现持久化：够验证链路，重启会丢。 */
public class MemoryPersistence implements PersistencePort {

    private final Map<String, List<ChatMessage>> contexts = new ConcurrentHashMap<>();
    private final Map<String, List<AnsResponse>> events = new ConcurrentHashMap<>();
    private final Set<String> sessions = ConcurrentHashMap.newKeySet();

    @Override
    public List<ChatMessage> loadContext(String historyId) {
        List<ChatMessage> saved = contexts.get(historyId);
        return saved == null ? new ArrayList<ChatMessage>() : new ArrayList<ChatMessage>(saved);
    }

    @Override
    public void persistContext(String historyId, List<ChatMessage> messages) {
        // 内核每轮只把「本轮新增」的消息交给调用方，按 historyId 追加即可
        contexts.computeIfAbsent(historyId, k -> new ArrayList<ChatMessage>()).addAll(messages);
    }

    @Override
    public void appendEvent(String historyId, AnsResponse event) {
        events.computeIfAbsent(historyId, k -> new ArrayList<AnsResponse>()).add(event);
    }

    @Override
    public void ensureSession(String historyId, String title, String mode) {
        sessions.add(historyId);
    }

    /** 回放：把一个会话的事件按顺序读出来 */
    public List<AnsResponse> replay(String historyId) {
        List<AnsResponse> list = events.get(historyId);
        return list == null ? new ArrayList<AnsResponse>() : new ArrayList<AnsResponse>(list);
    }
}
```

接上运行时和对话：

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;
import net.itzq.mira.modules.ai.persistence.PortHistoryPersist;
import net.itzq.mira.modules.runtime.KernelRuntime;

public class PersistenceDemo {
    public static void main(String[] args) throws Exception {
        MemoryPersistence port = new MemoryPersistence();

        KernelRuntime rt = KernelRuntime.builder()
                .name("my-app")
                .dataDir("D:/my-app/data")
                .persistencePort(port)   // 也可以在 start 之后用 rt.setPersistencePort(port)
                .build()
                .start();

        String historyId = IdGen.uuid();
        // 会话列表要能看到这条，先登记一次
        port.ensureSession(historyId, "第一次对话", "chat");

        AgentContextHolder holder = AgentContextHolder.builder()
                .userId("u1")
                .historyId(historyId)
                .modelAlias("deepseek-v3")
                .runtime(rt)
                .build();

        // 装载历史：这一步由应用自行调用（内核不知道存储长什么样）
        holder.getHistory().addAll(port.loadContext(historyId));

        AutoAgent agent = new AutoAgent(holder);
        // 让内核在每轮把新消息、事件交回给持久化接口
        agent.setHistoryPersist(PortHistoryPersist.of(port, historyId));

        agent.chatStream("帮我写一句项目简介").await();

        // 刷新页面时：重新装载上下文 + 回放事件
        System.out.println("历史消息数: " + port.loadContext(historyId).size());
        System.out.println("事件数: " + port.replay(historyId).size());
    }
}
```

### 这五个方法分别在什么时候被调用

- `loadContext(historyId)`：**由应用调用**。开新一轮会话前，把历史消息灌进上下文（`holder.getHistory()`）。内核不代调，因为它不知道存储长什么样。
- `persistContext(historyId, messages)`：内核在每轮对话结束、消息落库回调触发时调用。参数是**本轮新增**的消息（不是整个上下文），按 historyId 追加即可。
- `appendEvent(historyId, event)`：内核把一条条 `AnsResponse` 事件交给调用方。默认经 `PortHistoryPersist` 转发；若事件已在管道中落库，构造时传 `persistEvents=false`，避免重复落库。
- `ensureSession(historyId, title, mode)`：**由应用调用**。新建会话时登记一次，历史面板才会出现这条会话；`mode` 由实现方定义展示分组语义（例如 `chat`）。
- `isEnabled()`：执行前的开关判断。默认实现返回 `true`；内置的 `PersistencePort.NOOP` 返回 `false`，表示"即用即释放"。

不实现会怎样：`AgentContextHolder` 和运行时默认都用 `PersistencePort.NOOP`——不落库、没记忆，每一轮跑完消息就散在内存里，进程重启即丢。

### `AbstractHistoryPersist` / `PortHistoryPersist` 减少了哪些工作量

- 内核 agent 循环的落库回调其实是 `AbstractHistoryPersist`（两个抽象方法：`saveChatMessages`、`saveEventMessages`）。接入持久化无需实现它——`PortHistoryPersist.of(port, historyId)` 已将实现的 `PersistencePort` 转为该回调。接入时调用 `agent.setHistoryPersist(...)` 即可。
- 也无需自行攒批：内核会按批回调，攒批、清队列这些都在内核侧完成。

### 事件回放怎么用

`appendEvent` 存储的均为 `AnsResponse`，带 `type`（`ChatContent`、`ChatReasonContent`、`CallToolBegin`、`CallToolEnd`、`ChatEnd` 等）及 `historyId`/`msgId`。按顺序读取并交给前端，即可重放整条时间线；前端按 `type` 分流渲染。

## 跑起来会看到什么

- 第一轮对话正常出结果，同时消息和事件被写进内存 Map。
- `历史消息数` 和 `事件数` 都大于 0。
- 若将这段逻辑接入 HTTP 接口：刷新页面时前端以 `historyId` 重新请求，返回 `loadContext` 的消息与 `replay` 的事件，页面即可恢复原状。

## 换个场景怎么改

| 目标存储          | 需要动的                                                                                                                                                       |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| MySQL / 任意关系库 | 只改 `MemoryPersistence` 里的方法体：`loadContext` 查表、`persistContext` 批量 insert、`appendEvent` insert 事件行、`ensureSession` insert 会话行；`ChatMessage` 序列化为 JSON 存一列即可 |
| Redis         | 同样只改方法体：把三个容器换成 Redis 的 list/hash；`appendEvent` 用 `RPUSH` 天然保序                                                                                             |
| 本地文件          | 每个 historyId 一个 JSON 文件，`persistContext` 追加、`appendEvent` 一行一行追加                                                                                           |
| 换实现不用改业务代码    | 实现 `PersistencePort` 后 `.persistencePort(port)` 或 `rt.setPersistencePort(port)`；对话侧一行不用改                                                                   |
| 事件已自行落过库      | `PortHistoryPersist.of(port, historyId, false)`，避免重复落库                                                                                                     |

## 注意事项

- **先登记会话**：不调 `ensureSession`，`loadContext`/`persistContext` 照样能工作，但会话列表里不会出现这条。
- **`persistContext` 拿到的不是全量**：是"本轮新增消息"。按 historyId 追加，不应覆盖整表。
- **`port` 为 null 会静默退化**：`PortHistoryPersist.of(port, ...)` 传 null 时退化成 NOOP，不报错，但也不存。
- **顺序**：先 `loadContext` 灌历史，再 `setHistoryPersist` 开跑，两边都用同一个 `historyId`。
- **锚点字段**：交给持久化的 user 消息可能是 `PersistedMessage`（`ChatMessage` 的子类，多一个轮次锚点 `id`）。按运行时类型 JSON 序列化时该字段自然带出，读回时未知字段被忽略，不影响其作为普通消息使用。

> 相关：[基础对话](/MiraAgent/guide/agent/basic-chat.md) · [流式输出](/MiraAgent/guide/agent/streaming.md) · [内核运行时装配](/MiraAgent/guide/runtime/kernel-runtime.md)
