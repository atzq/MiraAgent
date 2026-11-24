> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 编排：让 AI 按流程跑

除了一问一答，也常需要把多个 AI 步骤按流程串联。本篇说明流程由哪三块拼成、内核提供哪些拼装件，以及哪一部分需由调用方编写。

## 场景

以每周自动生成周报为例：先从内部接口拉取本周数据，交给大模型总结，判断总结是否为空，最后输出。每个节点要做的事不同，用到的模型、工具也不同。目标是把它描述成一张流程图，再由执行器按图运行，而不是把逻辑硬编码成一长串调用。

## 三样东西的分工

一张能跑的流程，由三份东西拼成，各自回答一个问题：

| 东西       | 回答的问题                 | 内容                              |
| -------- | --------------------- | ------------------------------- |
| **配置快照** | 这个系统**有什么**           | 有哪些模型、哪些工具包、系统提示词。全局共享，一份就够     |
| **流程定义** | 节点**怎么连**、每个节点**用什么** | 节点清单、连线、每个节点的配置数据（用哪个模型、挂哪些工具包） |
| **调用参数** | **这一次**跑的是什么          | 这一次的输入是什么、挂哪个会话                 |

三者关系：流程定义按名字去配置快照里取资源，调用参数在运行时补给流程。流程里的节点写 `model_id: "deepseek-v3"`，这个别名必须能在配置快照的 `models` 里找到；节点写 `question` 为必需输入，这个值来自调用参数。对不上就跑不了——这正是下一节"先校验"要解决的问题。

## 一份最小的流程定义

流程定义不是内核规定的格式，它就是应用自身的数据模型。最小形态是"节点 + 连线 + 节点配置"：

```json
{
  "id": "weekly-report",
  "nodes": [
    {
      "id": "start",
      "type": "start-node",
      "data": {}
    },
    {
      "id": "fetch",
      "type": "data-fetch-node",
      "data": {
        "endpoint": "https://internal.example.com/weekly",
        "maxRows": 200
      }
    },
    {
      "id": "summarize",
      "type": "ai-chat-node",
      "data": {
        "model_id": "deepseek-v3",
        "toolkits": ["report-tools"],
        "prompt": "把下面的数据总结成一份周报"
      }
    },
    {
      "id": "check",
      "type": "condition-node",
      "data": {
        "expression": "output.length > 0"
      }
    },
    {
      "id": "end",
      "type": "end-node",
      "data": {}
    }
  ],
  "edges": [
    { "from": "start", "to": "fetch" },
    { "from": "fetch", "to": "summarize" },
    { "from": "summarize", "to": "check" },
    { "from": "check", "to": "end", "when": "true" }
  ]
}
```

读法：

- `nodes[].type` 是**节点类型**，决定"谁来执行这个节点"。
- `nodes[].data` 是**节点私有配置**，形状由节点类型自己定。比如 AI 节点里 `model_id` 是模型别名、`toolkits` 是工具包 id 列表。
- `edges` 是**连线**，决定执行顺序；带条件的连线（`when`）决定分支走向。

上面 `model_id`、`toolkits` 这些键名是这套流程自己的约定。内核不认识 `model_id`，它只认识"需要一个模型别名"这件事——把外部键名映射成内核能理解的引用，是下一篇文章要讲的装配动作。

## 内核提供哪几块拼装件

三块，分别对应上文的三份东西：

| 拼装件                 | 对应         | 作用                                               |
| ------------------- | ---------- | ------------------------------------------------ |
| `Invocation`        | 调用参数       | 描述"这一次怎么跑"：问题、图片、工作空间、模型覆盖、思考开关、会话 id、扩展键        |
| `NodeBlueprint`     | 流程定义里的一个节点 | 把一个节点翻译成内核能理解的"需求清单"：要哪些调用参数、按名字引用哪些资源、节点私有配置是什么 |
| `ExecutionResolver` | 校验         | 把"需求"和"供给"对上：模型别名在不在、工具包能不能解析、必需参数齐不齐，产出可执行结论    |

## 执行器主循环需由调用方编写

**内核不管调度。** 它不会代为遍历节点、不会判断分支、不会决定下一个是谁、也不会发"节点开始/结束"的事件。内核只负责"接收一个节点 + 这次的调用参数，说明这个节点能不能跑、跑起来用哪个模型和哪些工具"。

所以一个完整的编排执行器基于内核搭建，由以下部件组织，每个部件标注了其对应的内核 API：

| # | 部件        | 做什么                                   | 对应内核 API                                                                           |
| - | --------- | ------------------------------------- | ---------------------------------------------------------------------------------- |
| 1 | 流程数据模型    | 节点 + 连线 + 每个节点的配置数据                   | 自定义的 POJO / JSON，内核无对应                                                             |
| 2 | 执行器主循环    | 按连线取下一个节点执行，执行前后各发一个事件                | 自己写；事件用 `EventCenter` + `AnsResponse`                                              |
| 3 | 节点执行器注册表  | 按节点类型找到"谁来执行"                         | 自己写（一个 `Map<类型, 执行器>`）                                                             |
| 4 | 插件式节点     | 一个节点类型 = 一份清单（类型名、必填输入、引用的配置项）+ 一个执行器 | 设置值读 `GlobalConfigManager.pluginSetting(...)`；设置项定义用 `PluginSettingDef`            |
| 5 | 节点里跑 AI   | 先校验再执行，工具包 id 换成工具名集合                 | `NodeBlueprint.from(...)` + `ExecutionResolver.resolve(...)` + 通过实现 `ToolResolver` |
| 6 | 每个节点自己的记忆 | 每个节点一个会话 id，挂同一个持久化实现                 | `IdGen.uuid()` 生成会话 id；`AgentContextHolder.historyId`                              |
| 7 | 输出统一走事件   | 决定哪些事件透传、哪些自己收尾                       | `EventCenter` + `DefaultEventListener`                                             |

第 5 项是最核心的，也是唯一需要"内核与调用方各写一半"的地方。

## 最小可用代码

以一个 AI 节点为例：装配 → 校验 → 执行。把这套动作放进上面第 2 项的循环里，就是完整的执行器。

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.runtime.KernelRuntime;
import net.itzq.mira.modules.config.AgentConfig;
import net.itzq.mira.modules.config.GlobalConfigManager;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;
import net.itzq.mira.modules.ai.client.config.ModelApiConfig;
import net.itzq.mira.modules.ai.orchestration.ExecutionResolver;
import net.itzq.mira.modules.ai.orchestration.Invocation;
import net.itzq.mira.modules.ai.orchestration.NodeBlueprint;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

public class OneNodeDemo {

    public static void main(String[] args) {
        KernelRuntime rt = KernelRuntime.builder()
                .name("orchestration-demo")
                .dataDir("D:/my-app/data")
                .build();
        KernelRuntime.installDefault(rt);
        rt.start();

        // 配置快照：注册一个模型（流程定义要按别名引用它）
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

        // 流程定义里的一个节点（实际从数据模型读）
        Map<String, Object> nodeData = new HashMap<String, Object>();
        nodeData.put("model_id", "deepseek-v3");
        nodeData.put("prompt", "把下面的数据总结成一份周报");

        // 装配成内核能理解的"需求清单"
        NodeBlueprint bp = NodeBlueprint.from(
                "ai-chat-node",
                nodeData,
                Arrays.asList("question"),
                Collections.singletonMap("model", "model_id"));

        // 调用参数：这一次跑的是什么
        Invocation invocation = Invocation.builder()
                .question("本周进展：完成登录改造、修复 3 个缺陷")
                .historyId(IdGen.uuid())
                .build();

        // 先校验：模型在不在、必需参数齐不齐
        ExecutionResolver.Resolution resolution =
                ExecutionResolver.resolve(bp, invocation, null, decl);
        if (!resolution.canExecute()) {
            System.out.println(resolution.getReport().summary());
            rt.close();
            return;
        }

        // 校验通过，执行这个节点
        AgentContextHolder holder = AgentContextHolder.builder()
                .runtime(rt)
                .userId("u1")
                .historyId(IdGen.uuid())
                .modelAlias(resolution.getModelAlias())
                .prompt((String) nodeData.get("prompt"))
                .build();
        holder.addTools(resolution.getToolWhitelist().toArray(new String[0]));

        String output = new AutoAgent(holder).chat(invocation.getQuestion());
        System.out.println("节点输出 = " + output);

        rt.close();
    }
}
```

## 跑起来会看到什么

- 模型注册后，控制台打印 `节点输出 = 本周周报：...`。
- 如果故意把 `nodeData` 里的 `model_id` 改成 `deepseek-vX`（一个没注册的别名），校验会拦住，打印类似：

```text
[INCOMPLETE] missing=1, warnings=0
  - model: deepseek-vX (配置快照（declaration.models）中不存在该 alias)
```

- 如果故意不传 `question`，打印里会出现 `invocation: question` 的缺失项。

预校验的意义在于：配置错误在执行前就能被发现，而不是执行到中途才失败。

## 换个场景怎么改

**串行流程 vs 并行分支**

| 目标     | 主循环怎么改                                                                                        |
| ------ | --------------------------------------------------------------------------------------------- |
| 串行     | 维护一个"当前节点 id"，按 `edges` 取下一个，一个接一个                                                            |
| 并行分支   | 一个节点有多条出边时，把下游节点提交到内核自带线程池并发跑（`ThreadPoolUtil.getExecutorService()`，无需自行创建线程池），用计数器 / 屏障等它们汇合 |
| 条件分支   | 读连线上的 `when` 条件，只走命中的那条出边                                                                     |
| 人工审批节点 | 见下                                                                                            |

**加人工审批节点**

做法和[停止与续跑](/MiraAgent/guide/agent/stop-resume.md)是同一套机制：

1. 节点执行到"审批"时，向调用方发一个事件（`AnsResponse`）说明"等你决定"。
2. 不在循环里阻塞，直接结束这一轮，把流程状态（当前节点、已产出）存起来。
3. 外部做出决定后，带着决定再次进入循环，从这个节点继续往下。

审批节点本身不需要调模型，所以它不碰 `ExecutionResolver`；只有真正要跑 AI 的节点才走"装配 → 校验 → 执行"那三步。

## 约束

- **内核不提供调度**：没有现成的"运行流程图"方法，主循环、分支、并发均需自行实现。
- **流程定义里的别名需先在配置快照里存在**：先 `applyAllInstance()` 把模型 / 工具注册好，再跑节点。
- **每个节点用自己的会话 id**：想"每个节点有自己的记忆"，就给每个节点单独生成一个 `historyId`。

> 相关：[调用参数：这一次怎么跑](/MiraAgent/reference/orchestration/invocation.md)、[节点配置：这个节点需要什么](/MiraAgent/reference/orchestration/node-blueprint.md)、[执行前先校验](/MiraAgent/reference/orchestration/execution-resolver.md)
