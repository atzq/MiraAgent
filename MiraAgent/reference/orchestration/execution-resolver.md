> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 执行前先校验

流程配错了一个模型名，不希望跑到一半才出错。`ExecutionResolver` 就是那个"执行前的检查站"：把节点的需求（`NodeBlueprint`）和这一次的供给（`Invocation`）对一遍，说明模型在不在、工具包能不能解析、必需参数齐不齐，并给出可以直接拿去执行的模型别名和工具白名单。

## 场景

流程定义由运维在界面上拼，很容易把模型别名写错，或者引用一个已经被删掉的工具包。如果这些错误要等模型调用返回 404 才发现，用户体验非常糟。目标是在执行前先跑一次校验：错了就说清楚错在哪，对了就直接拿到"用哪个模型、带哪些工具"。

## `resolve(...)` 的执行流程

```java
ExecutionResolver.resolve(node, invocation, toolResolver, declaration)
```

四个参数：

| 参数             | 含义                                         |
| -------------- | ------------------------------------------ |
| `node`         | 节点的需求清单（`NodeBlueprint`），可传 null = 当成空节点   |
| `invocation`   | 这一次的调用参数（`Invocation`），可传 null = 当成空调用     |
| `toolResolver` | **实现的工具包解析回调**，可传 null（此时无法解析工具包，只在报告里记警告） |
| `declaration`  | 用哪份配置快照来校验模型（可传 null → 回落默认运行时）            |

它依次做三件事：

1. **`refs.model`** → 值不为空就查配置快照的 `models` 里有没有这个别名；值为空就回落配置快照的默认模型；两者都没有则记一条 `model` 缺失。
2. **`refs.toolkits`** → 值是非空的 `List` 时，交给 `ToolResolver` 解析成工具名集合；解析抛异常则记一条 `toolkit` 缺失；`toolResolver` 传了 null 则记一条警告。
3. **`requires`** → 逐个检查 `invocation` 里有没有这个键（`invocation.question` 和 `question` 两种写法都认）；缺了记一条 `invocation` 缺失。`refs` 里 `@invocation.<key>` 形态的间接引用如果取不到值，同样记 `invocation` 缺失。

## `Resolution` 的产出

| 方法                   | 作用                                               |
| -------------------- | ------------------------------------------------ |
| `canExecute()`       | 能不能执行。等价于报告 `isPass()`：无致命、无缺失                   |
| `getReport()`        | 校验报告 `ValidationReport`，含 `missing` / `warnings` |
| `getModelAlias()`    | 解析后的模型别名（可能来自节点，也可能是回落的默认模型）                     |
| `getToolWhitelist()` | 解析后的工具名集合，内容取决于传入的 `ToolResolver`                |
| `getResolvedRefs()`  | 全部引用解析后的结果，自定义引用名也在这里                            |

## 失败为什么不猜

校验报告**绝不静默补默认值**。具体来说：

- 节点写了一个不存在的模型别名 → 记**缺失**，不让执行。不会偷偷换成一个"看起来差不多"的模型。
- 工具包解析不出来 → 记**缺失**。不会给一个空工具集让节点"裸奔"。
- 必需参数没给 → 记**缺失**。

宁可挡住，也不猜。因为流程配错时，"猜一个相近的模型""给个空工具集"都会让错误更难排查。

`ToolResolver` 也要遵守同样的原则：

```java
public interface ToolResolver {
    Set<String> resolve(List<?> kitIds);
}
```

它把"工具包 id"解析成"工具名集合"。**解析不出来必须报错（抛异常），而不是返回空集合或 null。** 返回空集合会被当成"这个工具包就是空的"，于是流程带着一个错误的假设继续跑；抛异常则会被 `resolve` 记成缺失项，正大光明地挡下来。

## 最小可用代码

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.runtime.KernelRuntime;
import net.itzq.mira.modules.config.AgentConfig;
import net.itzq.mira.modules.config.GlobalConfigManager;
import net.itzq.mira.modules.config.ValidationReport;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;
import net.itzq.mira.modules.ai.client.config.ModelApiConfig;
import net.itzq.mira.modules.ai.orchestration.ExecutionResolver;
import net.itzq.mira.modules.ai.orchestration.ExecutionResolver.ToolResolver;
import net.itzq.mira.modules.ai.orchestration.Invocation;
import net.itzq.mira.modules.ai.orchestration.NodeBlueprint;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.HashMap;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;

public class ExecutionResolverDemo {

    // 工具包 id → 工具名集合；真实场景里来自数据库或配置
    private static final Map<String, Set<String>> TOOLKITS = new HashMap<String, Set<String>>();
    static {
        Set<String> tools = new LinkedHashSet<String>();
        tools.add("Read");
        tools.add("Grep");
        TOOLKITS.put("report-tools", tools);
    }

    // 实现的工具包解析回调：换不到就抛异常（fail-closed）
    private static final ToolResolver TOOL_RESOLVER = new ToolResolver() {
        @Override
        public Set<String> resolve(List<?> kitIds) {
            Set<String> names = new LinkedHashSet<String>();
            for (Object id : kitIds) {
                Set<String> one = TOOLKITS.get(String.valueOf(id));
                if (one == null) {
                    throw new IllegalStateException("工具包不存在: " + id);
                }
                names.addAll(one);
            }
            return names;
        }
    };

    public static void main(String[] args) {
        KernelRuntime rt = KernelRuntime.builder()
                .name("resolve-demo")
                .dataDir("D:/my-app/data")
                .build();
        KernelRuntime.installDefault(rt);
        rt.start();

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

        Map<String, String> refKeyMap = new HashMap<String, String>();
        refKeyMap.put("model", "model_id");
        refKeyMap.put("toolkits", "toolkits");

        Invocation invocation = Invocation.builder()
                .question("把本周数据总结成一份周报")
                .historyId(IdGen.uuid())
                .build();

        // ============ 情况一：模型别名写错了 ============
        Map<String, Object> badNode = new HashMap<String, Object>();
        badNode.put("model_id", "deepseek-vX");                 // 不存在
        badNode.put("toolkits", Arrays.asList("report-tools"));
        badNode.put("prompt", "把下面的数据总结成一份周报");

        NodeBlueprint badBp = NodeBlueprint.from(
                "ai-chat-node", badNode, Arrays.asList("question"), refKeyMap);

        ExecutionResolver.Resolution bad =
                ExecutionResolver.resolve(badBp, invocation, TOOL_RESOLVER, decl);
        if (!bad.canExecute()) {
            System.out.println(bad.getReport().summary());
            for (ValidationReport.Missing m : bad.getReport().getMissing()) {
                System.out.println("缺少: kind=" + m.getKind()
                        + ", ref=" + m.getRef() + ", detail=" + m.getDetail());
            }
            System.out.println("--- 修正模型别名后重试 ---");
        }

        // ============ 情况二：改对了，取出模型与工具白名单执行 ============
        Map<String, Object> node = new HashMap<String, Object>();
        node.put("model_id", "deepseek-v3");
        node.put("toolkits", Arrays.asList("report-tools"));
        node.put("prompt", "把下面的数据总结成一份周报");

        NodeBlueprint bp = NodeBlueprint.from(
                "ai-chat-node", node, Arrays.asList("question"), refKeyMap);

        ExecutionResolver.Resolution resolution =
                ExecutionResolver.resolve(bp, invocation, TOOL_RESOLVER, decl);
        if (!resolution.canExecute()) {
            System.out.println(resolution.getReport().summary());
            rt.close();
            return;
        }

        String modelAlias = resolution.getModelAlias();
        Set<String> toolWhitelist = resolution.getToolWhitelist();
        System.out.println("校验通过：模型 = " + modelAlias + ", 工具 = " + toolWhitelist);

        AgentContextHolder holder = AgentContextHolder.builder()
                .runtime(rt)
                .userId("u1")
                .historyId(IdGen.uuid())
                .modelAlias(modelAlias)
                .prompt((String) node.get("prompt"))
                .build();
        holder.addTools(toolWhitelist.toArray(new String[0]));

        String output = new AutoAgent(holder).chat(invocation.getQuestion());
        System.out.println("节点输出 = " + output);

        rt.close();
    }
}
```

## 跑起来会看到什么

先看到失败的报告：

```text
[INCOMPLETE] missing=1, warnings=0
  - model: deepseek-vX (该 alias 不在配置快照的 models 里)
缺少: kind=model, ref=deepseek-vX, detail=该 alias 不在配置快照的 models 里
--- 修正模型别名后重试 ---
```

修正后：

```text
校验通过：模型 = deepseek-v3, 工具 = [Read, Grep]
节点输出 = 本周周报：...
```

如果工具包 id 写成不存在的，报告里会出现 `kind=toolkit` 的缺失项；如果 `requires` 里的 `question` 没给，会出现 `kind=invocation`。

## 换个场景怎么改

**工具包来自数据库**

把 `ToolResolver` 的实现换成查库即可，接口不变：

```java
ToolResolver dbResolver = new ToolResolver() {
    @Override
    public Set<String> resolve(List<?> kitIds) {
        Set<String> names = new LinkedHashSet<String>();
        for (Object id : kitIds) {
            List<String> toolNames = toolkitDao.findToolNames(String.valueOf(id));
            if (toolNames == null || toolNames.isEmpty()) {
                throw new IllegalStateException("工具包不存在或为空: " + id);  // fail-closed
            }
            names.addAll(toolNames);
        }
        return names;
    }
};
```

**工具包来自用户勾选**

用户在前端勾了一组工具包 id，直接填进节点数据的 `toolkits` 字段就行；`ToolResolver` 负责把这些 id 展开成工具名。勾选变化时不需要改流程定义，只改调用前的数据。

**其他常见改法**

| 目标          | 改哪儿                                                                                                                               |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------- |
| 校验时用别的配置快照  | `resolve` 第四个参数传那份 `declaration`                                                                                                  |
| 节点不指定模型、用默认 | 节点数据里不写 `model_id`，`refs` 里就没有 `model`，会回落默认模型（附一条警告）                                                                             |
| 一个节点多个模型    | 自定义引用名放进 `refs`，自己用 `ExecutionResolver.hasModel(alias)` 校验，见[节点配置：这个节点需要什么](/MiraAgent/reference/orchestration/node-blueprint.md) |

## 使用要点

- **`canExecute()` 是唯一准入判据**：以 `canExecute()` 的返回值判断准入，而非比对 `missing.size()==0`。
- **报告输出**：`summary()` 适合日志，`getMissing()` 适合回给前端渲染成"补全清单"。
- **`toolResolver` 传 null 仅产生警告而非缺失**：节点确实引用工具包时，须传入解析回调，否则流程会带着未解析的工具继续走。
- **`getToolWhitelist()` 的内容由实现方决定**：要不要在里面带上系统基线工具，取决于 `ToolResolver` 返回什么。
- **别名判定是精确匹配**：别名前后有空格也算不存在。

> 相关：[节点配置：这个节点需要什么](/MiraAgent/reference/orchestration/node-blueprint.md)、[调用参数：这一次怎么跑](/MiraAgent/reference/orchestration/invocation.md)、[接入一个模型](/MiraAgent/guide/config/models.md)
