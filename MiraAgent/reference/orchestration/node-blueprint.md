> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 节点配置：这个节点需要什么

流程定义里的一个节点，写的是"该节点有哪些数据"：模型 id 叫 `model_id`、工具包叫 `toolkits`。可内核不认识这些长得各不相同的键名，它只认"需要一个模型别名""需要几个工具包"。把节点数据翻译成内核能理解的**需求清单**，用的就是 `NodeBlueprint`（节点配置声明）。

## 场景

流程里每个节点的数据形状都不一样：取数节点有 `endpoint`、`maxRows`；AI 节点有 `model_id`、`toolkits`、`prompt`。不希望由内核去猜每种节点的字段名，只在装配的那一步明确告知："`model_id` 这个字段，就是所需的模型引用。"这样节点类型爱怎么定键名就怎么定，内核只管照映射取。

## NodeBlueprint 的结构

它描述**一个节点在执行前需要什么**，由三个部分组成：

| 部分         | 类型                    | 含义                                          |
| ---------- | --------------------- | ------------------------------------------- |
| `requires` | `List<String>`        | 执行**必须**的调用参数键。比如 `["question"]`，缺了就不该跑     |
| `refs`     | `Map<String, Object>` | 按名字引用配置快照里的资源。内核约定两个名字：`model` 和 `toolkits` |
| `config`   | `Map<String, Object>` | 节点私有配置，形状由节点类型自己定                           |

外加一个 `type`：节点类型名，比如 `ai-chat-node`。

关于 `refs` 还有两点：

- 键是**内核认识的名字**（`model` / `toolkits`，也可放入别的自定义名）；值通常是一个别名或 id 列表。
- 值可以是 `@invocation.<key>` 形态，表示"这个引用不在节点里写死，运行时从调用参数里取"。比如 `refs.model = "@invocation.modelAlias"`，那么这次用哪个模型由调用参数决定。

## `NodeBlueprint.from(...)` 的四个参数

```java
NodeBlueprint.from(String nodeType, Map<String, Object> nodeData,
                   List<String> requires, Map<String, String> refKeyMap)
```

| 参数          | 含义                                                |
| ----------- | ------------------------------------------------- |
| `nodeType`  | 节点类型名，写进 `type`                                   |
| `nodeData`  | 节点私有配置原文，直接成为 `config`                            |
| `requires`  | 协议里的必需调用参数键清单，直接成为 `requires`                     |
| `refKeyMap` | **引用键名 → 节点数据里的字段名**的映射，用来从 `nodeData` 里抓出 `refs` |

`refKeyMap` 的映射方向是：键为内核认识的名字，值为 `nodeData` 里实际存的字段名。

比如 `refKeyMap = {"model": "model_id", "toolkits": "toolkits"}`，含义是：

- 内核要 `model` 这个引用 → 去 `nodeData` 里读 `model_id` 字段；
- 内核要 `toolkits` 这个引用 → 去 `nodeData` 里读 `toolkits` 字段。

`from` 的行为细节：

- `config` 就是 `nodeData` 本身（不是拷贝）。
- `requires` 会被复制一份，之后改原列表不会影响已装配的结果。
- 只有当 `nodeData` 里对应字段**不为 null** 时，才会写进 `refs`；字段缺失就不产生这个引用。

装好之后还能手工补引用：`bp.getRefs().put("model", "local-vllm")`。

## 最小可用代码

```java
import net.itzq.mira.modules.ai.orchestration.NodeBlueprint;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class NodeBlueprintDemo {

    public static void main(String[] args) {
        // 流程定义里一个 AI 节点的原始数据
        Map<String, Object> nodeData = new HashMap<String, Object>();
        nodeData.put("model_id", "deepseek-v3");
        nodeData.put("toolkits", new ArrayList<String>(Arrays.asList("report-tools")));
        nodeData.put("prompt", "把下面的数据总结成一份周报");
        nodeData.put("maxRows", 200);

        // 协议元数据：这个节点必须要有 question
        List<String> requires = Arrays.asList("question");

        // 引用映射：内核的 model ← 节点里的 model_id；内核的 toolkits ← 节点里的 toolkits
        Map<String, String> refKeyMap = new HashMap<String, String>();
        refKeyMap.put("model", "model_id");
        refKeyMap.put("toolkits", "toolkits");

        NodeBlueprint bp = NodeBlueprint.from("ai-chat-node", nodeData, requires, refKeyMap);

        System.out.println("节点类型 = " + bp.getType());
        System.out.println("必需参数 = " + bp.getRequires());
        System.out.println("引用     = " + bp.getRefs());
        System.out.println("私有配置 = " + bp.getConfig());
    }
}
```

## 跑起来会看到什么

```text
节点类型 = ai-chat-node
必需参数 = [question]
引用     = {model=deepseek-v3, toolkits=[report-tools]}
私有配置 = {prompt=把下面的数据总结成一份周报, model_id=deepseek-v3, toolkits=[report-tools], maxRows=200}
```

`refs` 里出现内核认识的 `model` 和 `toolkits`，值是节点里 `model_id`、`toolkits` 的真实内容；`config` 保留节点的全部原始字段（包括内核不关心的 `maxRows`）。

## 换个场景怎么改

**一个节点要用两个模型**

内核的 `ExecutionResolver` 只对名字恰好叫 `model` 的引用做模型存在性校验。使用两个模型时，为第二个模型指定一个自定义引用名并放入 `refs`：

```java
NodeBlueprint bp = NodeBlueprint.from("ai-chat-node", nodeData, requires, refKeyMap);

// 第二个模型：换个自定义引用名
bp.getRefs().put("refinerModel", "gpt-4o-mini");
```

这个 `refinerModel` 不在内核约定名单里，它会**原样出现在解析结果的 `resolvedRefs` 里**，不会被自动校验。要校验它就自己来：

```java
boolean exists = net.itzq.mira.modules.ai.orchestration.ExecutionResolver
        .hasModel("gpt-4o-mini");
```

**用工具包 id 列表怎么写**

工具包引用的名字必须是 `toolkits`，值是一个 `List`（id 列表），内核才会去走 `ToolResolver` 解析：

```java
// refKeyMap 里把内核的 toolkits 映射到节点字段
refKeyMap.put("toolkits", "toolkit_ids");   // 节点里字段叫 toolkit_ids

// 于是 nodeData 里这样写：
// nodeData.put("toolkit_ids", Arrays.asList("report-tools", "db-tools"));
```

解析出来的是一组**工具名**（不是 id），见[执行前先校验](/MiraAgent/reference/orchestration/execution-resolver.md)。

**其他常见改法**

| 目标           | 改哪儿                                       |
| ------------ | ----------------------------------------- |
| 模型运行时才定      | `refs.model` 放 `"@invocation.modelAlias"` |
| 节点必须收到某个业务参数 | 往 `requires` 里加键名                         |
| 节点数据字段改叫别的名字 | 只改 `refKeyMap` 的值，别动键                     |

## 使用要点

- **`refKeyMap` 的映射方向**：键为内核认识的名字（`model` / `toolkits`），值为节点里的字段名，二者不可颠倒。
- **字段缺失的行为**：字段缺失不会报错，也不会产生对应引用；`nodeData` 里没有 `model_id` 时，`refs` 里就没有 `model`，后续校验按"未指定模型"处理（可能回落默认模型，也可能记缺失）。
- **`config` 与 `nodeData` 同源**：二者是同一个对象，装配后修改 `nodeData` 会同步反映到 `config`。
- **自定义引用名不自动校验**：内核只处理 `model` 和 `toolkits` 两个约定名。

> 相关：[调用参数：这一次怎么跑](/MiraAgent/reference/orchestration/invocation.md)、[执行前先校验](/MiraAgent/reference/orchestration/execution-resolver.md)、[插件配置项](/MiraAgent/guide/config/plugin-settings.md)
