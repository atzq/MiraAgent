> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# HTTP 工具

已有的 REST 接口通过一份"接口说明"即可变为 AI 可调用工具，无需为每个接口编写工具类。

## 场景

已有运行稳定的接口（查订单、查库存、改地址）。希望由 AI 查询订单状态时，为每个接口编写 Java 工具类较为繁琐，且接口增多时需同步修改代码。

所需仅为填写若干字段——工具名、请求地址、参数——AI 即可调用。内核中的 **HTTP 工具** 即用于此。

## 最小可用代码

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;
import net.itzq.mira.modules.ai.http.HttpParam;
import net.itzq.mira.modules.ai.http.HttpToolDefine;
import net.itzq.mira.modules.ai.http.HttpToolMeta;
import net.itzq.mira.modules.ai.http.HttpToolRegistry;
import net.itzq.mira.modules.runtime.KernelRuntime;

import java.util.Arrays;
import java.util.LinkedHashSet;
import java.util.Set;

public class HttpToolDemo {
    public static void main(String[] args) throws Exception {
        KernelRuntime rt = KernelRuntime.builder()
                .name("my-app")
                .dataDir("D:/my-app/data")
                .build()
                .start();

        // 1. 描述一个接口：路径里用 {id} 当占位符
        HttpParam id = new HttpParam();
        id.setName("id");
        id.setIn(HttpParam.In.PATH);      // 这个参数要填到路径里
        id.setRequired(true);
        id.setType("string");             // 给模型看的类型
        id.setDescription("订单号");

        HttpToolMeta meta = new HttpToolMeta();
        meta.setName("order_get_by_id");  // 工具函数名，唯一
        meta.setSummary("按订单号查订单");
        meta.setDescription("调用订单服务，返回订单详情 JSON");
        meta.setHttpMethod("GET");
        meta.setServerUrl("https://api.mycompany.com");
        meta.setPathTemplate("/orders/{id}");
        meta.setParams(Arrays.asList(id));
        meta.setTimeoutMs(5000);          // 不写就用全局默认 10 秒
        meta.setMaxResponseBytes(65536);  // 响应太长就截断，不写就用全局默认 64KB

        // 2. 注册两处：一处让执行时能按名字回查这份说明，一处让模型"看得见"这个工具
        HttpToolRegistry.register(meta);
        rt.toolRegistry().registerTool(new HttpToolDefine(meta));

        // 3. 会话白名单里加上这个工具名
        AgentContextHolder holder = AgentContextHolder.builder()
                .userId("u1")
                .historyId(IdGen.uuid())
                .modelAlias("deepseek-v3")
                .runtime(rt)
                .build();

        Set<String> whitelist = new LinkedHashSet<>(AutoAgent.systemBaselineTools(null));
        whitelist.add(meta.getName());

        AutoAgent agent = new AutoAgent(holder);
        agent.mountToolsIfAbsent(whitelist.toArray(new String[0]));

        // 4. 开聊
        agent.chatStream("帮我查一下订单 A10086 现在什么状态").await();
    }
}
```

### 参数位置

`HttpParam.setIn(...)` 决定参数在 HTTP 请求中的位置：

| 位置                    | 含义                                         |
| --------------------- | ------------------------------------------ |
| `HttpParam.In.PATH`   | 填进 `pathTemplate` 里的 `{名字}` 占位符，值会做 URL 编码 |
| `HttpParam.In.QUERY`  | 拼成 URL 查询串；值是数组时会展开成多个同名参数                 |
| `HttpParam.In.HEADER` | 放进请求头                                      |
| `HttpParam.In.BODY`   | 放进 JSON 请求体                                |
| `HttpParam.In.FORM`   | 放进表单（`application/x-www-form-urlencoded`）  |
| `HttpParam.In.COOKIE` | 放进 Cookie                                  |

## 跑起来会看到什么

- 对话里 AI 会调用 `order_get_by_id`，参数是 `{"id":"A10086"}`；前端工具时间线上能看到这次调用。
- 工具返回一段 JSON 文本，形如 `{"status":200,"body":{...订单详情...}}`：`status` 为 HTTP 状态码，`body` 为自动解析后的响应体（解析失败时保留原文）。
- 响应超过 `maxResponseBytes` 时，末尾追加 `...(truncated)`，表示内容被截断。
- 连接失败、超时或抛出异常时，返回 `{"error":"..."}`，不会中断整个对话。

## 换个场景怎么改

| 目标              | 改哪里                                                                                                                                       |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| 改成 POST 提交 JSON | `meta.setHttpMethod("POST")`；参数用 `In.BODY`。**只有一个** BODY 参数时，模型给的实参直接当请求体；**多个** BODY 参数时自动拼成 JSON 对象，并加 `Content-Type: application/json` |
| 提交表单            | 参数用 `In.FORM`，请求头会自动设成 `application/x-www-form-urlencoded`                                                                                |
| 加查询条件           | 参数用 `In.QUERY`                                                                                                                            |
| 固定鉴权头           | `meta.setHeaders(map)` 写死；或让模型传的话用 `In.HEADER`；或全局统一加用 `HttpToolInvoker.setGlobalHeaderProvider(() -> headers)`（每次请求现取）                   |
| 统一调超时和截断上限      | `HttpToolInvoker.setDefaultTimeoutMs(毫秒)`、`HttpToolInvoker.setDefaultMaxBytes(字节)`；单个接口想单独设，就在 `meta` 上写                                  |
| 下线一个接口          | `HttpToolRegistry.unregister("order_get_by_id")` **加上** `rt.toolRegistry().unregisterTool("order_get_by_id")`                             |
| 一批同前缀接口一起下线     | `HttpToolRegistry.unregisterByPrefix("order_")` **加上** `rt.toolRegistry().unregisterByPrefix("order_")`                                   |
| 不同会话给不同接口       | 白名单按会话拼，只把该会话允许的 `meta.getName()` 加进去                                                                                                     |

## 使用要点

- **需写入两个注册表**，这是最易遗漏的一步：

  - `HttpToolRegistry.register(meta)` 负责执行时按工具名回查这份接口说明；
  - `rt.toolRegistry().registerTool(new HttpToolDefine(meta))` 负责让工具出现在模型的工具列表里。

  只注册第一个，模型根本不知道有这个工具；只注册第二个，模型能调但执行时查不到说明，会返回 `{"error":"工具元数据缺失"}`。

- **为内存表**：接口说明仅存于进程内存，重启后需重新注册。常驻应用宜在启动装配阶段统一注册。

- **type 仅作模型可见的类型说明**：填写 `integer` 只影响生成的参数描述，不会自动将模型传入的字符串转为数字；类型对齐需在接口侧保证。

- **必填仅通过说明，不通过拦截**：`setRequired(true)` 写入给模型的参数说明，AI 通常据此传参，但内核不代为拦截；缺少参数时路径占位符留空。

- **返回值始终为字符串**：`status`/`body` 或 `error`；如需结构化数据，需自行解析这段 JSON。

- **接口名需唯一**：函数名全局唯一，且不得与内置工具名重名，宜统一加业务前缀（如 `order_`）。

> 相关：[自定义工具](/MiraAgent/guide/agent/custom-tools.md) · [内置工具一览](/MiraAgent/guide/agent/builtin-tools.md) · [配置中心](/MiraAgent/guide/config/config-center.md)
