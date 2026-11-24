> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 自定义工具

给普通 Java 方法加注解即可将其变为 AI 可调用工具。以下以"查订单"为例走完流程，并说明一处易混淆点：**工具注册是全局的，白名单是每次对话的**。

## 场景

若有内部业务方法（比如查订单状态），可让 AI 调用它：用户问"订单 SO20260101 到哪了"，AI 就自己去调这个方法，把结果讲给用户。

## 最小可用代码

### 方法注解

内核靠两个注解认识一个工具：方法上的 `@Tool`，参数上的 `@ToolParam`。

```java
import net.itzq.mira.modules.ai.tool.annotation.Tool;
import net.itzq.mira.modules.ai.tool.annotation.ToolParam;
import net.itzq.mira.modules.ai.tool.ToolCallResult;

public class OrderTool {

    // 替换为实际业务依赖
    private final OrderService orderService = new OrderService();

    @Tool(
        name = "queryOrder",                 // 工具唯一名字，AI 就是靠这个名字点它
        display = "查询订单",                 // 给界面看的展示名
        description = "按订单号查询订单状态。" // 给模型看的说明：它决定什么时候该用这个工具
                + "orderId 是订单号，例如 SO20260101。查不到订单时会明确返回查不到。"
    )
    public String queryOrder(
            @ToolParam(description = "订单号，例如 SO20260101") String orderId,
            @ToolParam(description = "是否同时返回物流信息，默认 false", required = false) Boolean withLogistics) {

        try {
            String status = orderService.findStatus(orderId);   // 项目内部的业务方法
            if (status == null) {
                return ToolCallResult.error("没有找到订单：" + orderId);
            }
            String text = "订单 " + orderId + " 当前状态：" + status;
            if (Boolean.TRUE.equals(withLogistics)) {
                text += "\n物流：" + orderService.findLogistics(orderId);
            }
            return ToolCallResult.success(text);
        } catch (Exception e) {
            return ToolCallResult.error("查询订单失败：" + e.getMessage());
        }
    }
}
```

注解每个属性的意思：

| 注解 / 属性                        | 作用                                           |
| ------------------------------ | -------------------------------------------- |
| `@Tool(name=...)`              | 工具唯一名。AI 调用时用它，白名单里也用它。不填则默认用方法名。            |
| `@Tool(name=...).display`      | 展示名，给前端界面显示（例如"查询订单"）。                       |
| `@Tool(name=...).description`  | 给模型看的说明，**决定模型什么时候会想到用它**，写清楚用途和参数含义。        |
| `@Tool(name=...).subAgent`     | 标成 `true` 表示这是个子代理型工具（默认 false）。             |
| `@Tool(name=...).returnDirect` | 是否让工具结果直接返回、不再回给模型（默认 false）。                |
| `@ToolParam(description=...)`  | 单个参数的说明。                                     |
| `@ToolParam(required=...)`     | 该参数是否必填，默认 `true`。可选的参数标 `required = false`。 |

工具方法还可以多接收一个 `AgentContextHolder` 参数（不用打 `@ToolParam`），内核会自动把本次会话的上下文传进来——要用工作目录、会话附件、模型别名时就用得上。

### 注册到运行时（全局）

注册是**注册进运行时的工具注册表**，一次注册，这个运行时里的所有对话都能看到它。

```java
import net.itzq.mira.modules.runtime.KernelRuntime;

rt.toolRegistry().scanTools(OrderTool.class);
```

可以一次传多个类：

```java
rt.toolRegistry().scanTools(OrderTool.class, UserTool.class);
```

不使用注解时也可手写工具定义，调用 `rt.toolRegistry().registerTool(...)`；使用注解时以 `scanTools` 最为直接。

### 挂到对话白名单（每次对话）

注册完还不等于能用。AI 能不能用它，取决于这次对话的白名单 `holder.getTools()`。

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;

import java.util.Arrays;

String historyId = IdGen.uuid();
AgentContextHolder holder = AgentContextHolder.builder()
        .userId("u1")
        .historyId(historyId)
        .modelAlias("deepseek-v3")
        .prompt("你是客服助手。用户问订单时，调用 queryOrder 工具去查，不要凭空回答。")
        .runtime(rt)
        .vfsId(historyId)
        .build();

AutoAgent agent = new AutoAgent(holder);

// 把刚注册的工具 + 内核基线工具一起挂上（第一个参数是"应用内自行注册的工具名"）
agent.mountToolsIfAbsent(
        AutoAgent.systemBaselineTools(Arrays.asList("queryOrder")).toArray(new String[0]));

String answer = agent.chat("帮我查一下订单 SO20260101 现在什么状态");
System.out.println(answer);
```

## 跑起来会看到什么

- 模型读到工具说明后，判断该调 `queryOrder`，发来一个 `{"orderId":"SO20260101"}` 的调用；
- 内核反射调用 Java 方法，把返回值作为工具结果塞回历史；
- 模型拿着结果输出最终回答："您的订单 SO20260101 当前状态：已发货"；
- 前端会先收到 `CallToolBegin`（展示"查询订单"），执行完收到 `CallToolEnd`。

工具方法的返回值就是给模型看的工具结果文本。以 `ToolCallResult.success(...)` / `ToolCallResult.error(...)` 包装时，会在结果首行写入 `[mira:ok]` / `[mira:err]`，模型与前端据此分辨调用成败。

## 换个场景怎么改

| 目标            | 改哪儿                                                      |
| ------------- | -------------------------------------------------------- |
| 换个业务方法        | 换 `@Tool(name=...)` 和 `description`，方法体里调用业务代码           |
| 参数可选          | 在 `@ToolParam` 上加 `required = false`                     |
| 参数是枚举         | 直接写枚举类型，内核会自动把枚举值作为可选值给模型                                |
| 拿到会话上下文       | 方法参数里加一个 `AgentContextHolder`                            |
| 只在某些对话里开放这个工具 | 注册保持全局不变，只在需要的对话里把它加进 `holder.getTools()`                |
| 注册多个工具类       | `rt.toolRegistry().scanTools(A.class, B.class, C.class)` |

## 注意事项

- `scanTools(...)` 与 `mountToolsIfAbsent(...)` 是两件事：`scanTools(...)` 将工具注册进运行时，全局生效，调用一次即可；`mountToolsIfAbsent("queryOrder")` 将工具加入本次对话白名单，每个新会话都要执行。已注册但未进入白名单时，模型无法调用该工具。
- 工具名不可重复：同名工具后注册的覆盖先注册的，内核记一条 warning。
- 工具方法不应向外抛异常（内核会兜住并包成 `[mira:err]`）；以 `ToolCallResult.error(...)` 返回可明确表达错误。
- 参数名要和 AI 传来的 JSON key 对得上，内核按参数名取值。

> 相关：[工具调用](/MiraAgent/guide/agent/tool-calling.md)、[内置工具一览](/MiraAgent/guide/agent/builtin-tools.md)、[HTTP 工具](/MiraAgent/guide/extensions/http-tools.md)
