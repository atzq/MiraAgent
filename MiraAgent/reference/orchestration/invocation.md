> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 调用参数：这一次怎么跑

配置快照回答"系统有什么"，调用参数回答"**这一次**怎么跑"：问什么、带不带图片、用哪个模型、挂哪个会话。它是一次性的，用完即弃。下文说明其各字段含义、不能写入配置文件的原因，以及它与执行现场（`AgentContextHolder`）的关系。

## 场景

同一个模型、同一套提示词，用户这次问的是"审阅合同"，下次问的是"撰写周报"。这些"这一次"的信息，不该混进系统配置里。需要一个干净的容器，把"这一次的输入和覆盖项"装起来，交给执行环节。

## `Invocation` 的每个字段

| 字段               | 含义        | 说明                              |
| ---------------- | --------- | ------------------------------- |
| `question`       | 用户问题      | **通常必填**；节点的 `requires` 里一般就写着它 |
| `images`         | 图片附件      | base64 data URI 列表，可空           |
| `workspacePath`  | 会话工作空间路径  | 真实磁盘路径，可空                       |
| `modelAlias`     | 调用级模型覆盖   | 可空；为空用配置快照里的默认模型                |
| `enableThinking` | 调用级思考开关覆盖 | 可空；为空用配置快照里的默认                  |
| `historyId`      | 会话 id     | 可空；**为空 = 不挂持久化 = 即用即释放**       |
| `extra`          | 自定义的扩展键   | `Map<String, Object>`，不参与校验     |

几个配套方法：

- `inv.get("key")`：按键取值，`question` / `images` / `modelAlias` 等与 `extra` 里的键都能取。
- `inv.has("key")`：这个键有没有可用值。null、空串、空集合都算"没有"——校验必需参数靠的就是它。
- `inv.toMap()` / `Invocation.fromMap(map)` / `Invocation.fromJson(json)`：在"协议形态"和对象之间互转，方便落盘或跨进程传递。

## 为什么它必须"用完即弃"

`Invocation` 描述的是**一次调用**，所以它绝对不能写进配置快照或流程定义文件，原因有三：

1. **语义不符**：配置快照是"系统有什么"，流程定义是"节点怎么连"，都不该出现"某个用户某个时刻问的这句话"。写进去等于把一次性的输入冻结成了全局配置。
2. **会串**：多个用户同时用，各自的 `question`、`workspacePath` 不可能共享一份。
3. **可能带敏感数据**：用户输入、本地路径、图片，都不适合长期躺在配置文件里。

调用方在"发送这一次请求"时现场构造，用完丢弃；需留档的内容另存到会话记录，不回流到配置。

## 它和执行现场是什么关系

两句话区分：

- **`Invocation` 是"这次要什么"**：一份输入清单，只读、一次性。
- **`AgentContextHolder` 是"执行时的现场"**：模型、工具、历史、运行时、事件通道都挂在这上面，是真正被 agent 使用的运行态。

关系为**单向填充**：`Invocation` 的值取出填入 `AgentContextHolder` 后交由 agent 执行；反向不成立，现场状态不回写调用参数。

| 关心的事   | 看 `Invocation` | 看 `AgentContextHolder`                        |
| ------ | -------------- | --------------------------------------------- |
| 这一次问什么 | `question`     | `getHistory()` 里已有的对话                         |
| 用哪个模型  | `modelAlias`   | `getModelAlias()`                             |
| 挂哪个会话  | `historyId`    | `getHistoryId()`                              |
| 业务扩展键  | `extra`        | `getTempVariables()` / `getGlobalVariables()` |

## 最小可用代码

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.runtime.KernelRuntime;
import net.itzq.mira.modules.config.AgentConfig;
import net.itzq.mira.modules.config.GlobalConfigManager;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;
import net.itzq.mira.modules.ai.client.config.ModelApiConfig;
import net.itzq.mira.modules.ai.orchestration.Invocation;

import java.util.ArrayList;
import java.util.Arrays;

public class InvocationDemo {

    public static void main(String[] args) {
        KernelRuntime rt = KernelRuntime.builder()
                .name("invocation-demo")
                .dataDir("D:/my-app/data")
                .build();
        KernelRuntime.installDefault(rt);
        rt.start();

        // 注册一个模型（流程/调用参数都按别名引用它）
        GlobalConfigManager decl = rt.declaration();
        ModelApiConfig model = ModelApiConfig.builder()
                .alias("deepseek-v3")
                .providerName("deepseek")
                .apiModelName("deepseek-chat")
                .apiHost("https://api.deepseek.com")
                .apiEndpoint("/v1/chat/completions")
                .apiKey(System.getenv("DEEPSEEK_API_KEY"))
                .build();
        decl.setModels(new ArrayList<ModelApiConfig>(Arrays.asList(model)));
        AgentConfig agentConfig = new AgentConfig();
        agentConfig.setDefaultModel("deepseek-v3");
        decl.setAgentConfig(agentConfig);
        decl.applyAllInstance();

        // 1) 构造调用参数：这一次要什么
        Invocation invocation = Invocation.builder()
                .question("帮我看看这份合同有没有风险点")
                .workspacePath("D:/workspace/contract-review")
                .modelAlias("deepseek-v3")
                .enableThinking(Boolean.FALSE)
                .historyId(IdGen.uuid())
                .extra("chatMode", "review")
                .extra("tenantId", "t-001")
                .build();

        // 2) 把参数填进执行现场
        AgentContextHolder holder = AgentContextHolder.builder()
                .runtime(rt)
                .userId("u1")
                .historyId(invocation.getHistoryId())
                .modelAlias(invocation.getModelAlias())
                .workspacePath(invocation.getWorkspacePath())
                .prompt("你是一个严谨的合同审阅助手")
                .build();

        // 业务扩展键顺手转成现场变量，供工具或后续节点使用
        Object tenantId = invocation.get("tenantId");
        if (tenantId != null) {
            holder.getGlobalVariables().put("tenantId", String.valueOf(tenantId));
        }

        // 3) 开跑
        if (!invocation.has("question")) {
            System.out.println("缺少 question，无法执行");
            rt.close();
            return;
        }
        String answer = new AutoAgent(holder).chat(invocation.getQuestion());
        System.out.println("回答 = " + answer);

        rt.close();
    }
}
```

## 跑起来会看到什么

- 控制台打印 `回答 = 这份合同有 3 个风险点：...`。
- 如果 `question` 传空串，`invocation.has("question")` 为 false，会打印 `缺少 question，无法执行`，不会去调模型。
- `holder.getGlobalVariables()` 里能看到 `tenantId=t-001`，供后面的工具/节点读取。

## 换个场景怎么改

**加自定义业务参数（`extra`）**

`extra` 是给业务用的自由袋，内核不校验它的内容，加多少键都行：

```java
Invocation invocation = Invocation.builder()
        .question("生成周报")
        .extra("tenantId", "t-001")
        .extra("locale", "zh-CN")
        .extra("maxTokens", 2000)
        .build();

// 读出来
String tenant = String.valueOf(invocation.get("tenantId"));
```

| 想加的东西       | 放哪                                   | 读法                            |
| ----------- | ------------------------------------ | ----------------------------- |
| 一次性的业务参数    | `extra`                              | `invocation.get("键")`         |
| 执行时要给工具看的变量 | `AgentContextHolder.tempVariables`   | `holder.getTempVariables()`   |
| 整个会话共享的变量   | `AgentContextHolder.globalVariables` | `holder.getGlobalVariables()` |

**其他常见改法**

| 目标       | 改哪儿                                                                                               |
| -------- | ------------------------------------------------------------------------------------------------- |
| 这次换成别的模型 | `.modelAlias("local-vllm")`                                                                       |
| 这次打开思考   | `.enableThinking(Boolean.TRUE)`                                                                   |
| 这次不带记忆   | 不写 `historyId`（即用即释放）                                                                             |
| 这次要带图片   | `.image("data:image/png;base64,...")`，并且用 `agent.chatStream(question, invocation.getImages())` 发送 |

## 使用要点

- **`historyId` 为空**：表示不挂持久化，跑完即弃；需要连续对话时须为其指定一个 id。
- **`question` 通常必填**：编排节点的 `requires` 经常包含它，缺了会被校验拦下。
- **`Invocation` 不序列化进配置**：其 `toMap()` 用于落盘会话数据，但那是会话数据的形态，不是配置。
- **`has` 的判空规则**：空串和空集合都算"没有"，所以传 `""` 和没传效果一样。

> 相关：[编排：让 AI 按流程跑](/MiraAgent/reference/orchestration/overview.md)、[执行前先校验](/MiraAgent/reference/orchestration/execution-resolver.md)、[停止与续跑](/MiraAgent/guide/agent/stop-resume.md)
