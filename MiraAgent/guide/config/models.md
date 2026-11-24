> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 接入一个模型

接入 DeepSeek、OpenAI 或自行部署的本地服务（如 vLLM），本质都是往配置快照的 `models` 里增加一条记录：向内核声明该别名对应的地址、模型名与密钥。下文逐字段说明，并重点说明**哪些参数设置后才会发出、哪些设为 null 则完全不发出**。

## 场景

以接入两个模型为例：一个线上 DeepSeek 作为主力，一个本地 vLLM 用于低成本试跑。两次对话分别使用两者，并可按需开启思考模式。这些配置不写死在代码里。

## 模型记录字段

`ModelApiConfig` 的一行就是"一个可用的模型"。字段分四组：

**基本信息**

| 字段                | 含义                                 |
| ----------------- | ---------------------------------- |
| `alias`           | 别名。注册、查找、切换模型都用它，必须唯一且非空           |
| `providerName`    | 供应方名字，仅作标识                         |
| `apiModelName`    | 真正发给服务端的模型名，如 `deepseek-chat`      |
| `apiHost`         | 主机地址，如 `https://api.deepseek.com`  |
| `apiEndpoint`     | 路径，如 `/v1/chat/completions`        |
| `apiKey`          | 密钥                                 |
| `apiHeaders`      | 额外请求头，`Map<String,String>`，可空      |
| `sseEventHandler` | 流式事件处理器的全类名，可空；不写用默认的 OpenAI 兼容处理器 |
| `displayName`     | 展示名，给界面使用；为空回退 `alias`             |
| `modelType`       | 模型类型，如 `chat`，可空                   |

**能力字段**

| 字段                    | 含义                                        |
| --------------------- | ----------------------------------------- |
| `supportImage`        | 是否支持图片输入。`1` 支持，`0` 不支持；**为 null 时按支持处理** |
| `contextWindowInput`  | 输入上下文窗口（token 数），可空                       |
| `contextWindowOutput` | 输出上下文窗口（token 数），可空                       |
| `toolCallRounds`      | 工具调用最大轮数，可空                               |

**采样参数——null 语义很重要**

| 字段                    | 含义    |
| --------------------- | ----- |
| `samplingTemperature` | 温度    |
| `samplingTopP`        | top-p |
| `samplingTopK`        | top-k |

这三个字段**为 null 时，该参数不出现在请求体里**。若不需某个默认值影响模型，应留空而非填 0。内核装配请求参数时只发出非 null 的采样参数；`topK` 经透传通道发出（作为 `top_k`）。

**思考模式与自定义参数**

| 字段                     | 含义                                       |
| ---------------------- | ---------------------------------------- |
| `thinkingEnableParam`  | 开启思考时，往请求体里加什么。是一个 JSON 对象**字符串**，逐键展开   |
| `thinkingDisableParam` | 关闭思考时，往请求体里加什么。同样是 JSON 对象字符串            |
| `customParams`         | 开放对象：JSON 对象字符串，**逐键原样透传进请求体**，用来接各家私有参数 |

`customParams` 是这四组里最灵活的：任何服务端的私有参数（`repetition_penalty`、`chat_template_kwargs` 之类）都能塞进去。装配顺序上它优先级最高，会和前面的参数合并。

## 最小可用代码

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.runtime.KernelRuntime;
import net.itzq.mira.modules.config.AgentConfig;
import net.itzq.mira.modules.config.GlobalConfigManager;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;
import net.itzq.mira.modules.ai.client.config.ModelApiConfig;

import java.util.ArrayList;
import java.util.List;

public class ModelsDemo {

    public static void main(String[] args) {
        KernelRuntime rt = KernelRuntime.builder()
                .name("models-demo")
                .dataDir("D:/my-app/data")
                .build();
        KernelRuntime.installDefault(rt);
        rt.start();

        // 模型一：线上 DeepSeek
        ModelApiConfig deepseek = ModelApiConfig.builder()
                .alias("deepseek-v3")
                .providerName("deepseek")
                .apiModelName("deepseek-chat")
                .apiHost("https://api.deepseek.com")
                .apiEndpoint("/v1/chat/completions")
                .apiKey(System.getenv("DEEPSEEK_API_KEY"))
                .samplingTemperature(0.3)
                .samplingTopP(0.9)
                // 不设 samplingTopK，所以请求里不会出现 top_k
                .thinkingEnableParam("{\"enable_thinking\":true}")
                .thinkingDisableParam("{\"enable_thinking\":false}")
                .build();

        // 模型二：本地 vLLM（OpenAI 兼容）
        ModelApiConfig vllm = ModelApiConfig.builder()
                .alias("local-vllm")
                .providerName("vllm")
                .apiModelName("Qwen2.5-7B-Instruct")
                .apiHost("http://127.0.0.1:8000")
                .apiEndpoint("/v1/chat/completions")
                .apiKey("sk-local")
                .samplingTemperature(0.7)
                .customParams("{\"repetition_penalty\":1.05}")
                .build();

        List<ModelApiConfig> models = new ArrayList<>();
        models.add(deepseek);
        models.add(vllm);

        GlobalConfigManager decl = rt.declaration();
        decl.setModels(models);

        AgentConfig agentConfig = new AgentConfig();
        agentConfig.setDefaultModel("deepseek-v3");
        decl.setAgentConfig(agentConfig);

        // 应用：两个模型注册进运行时，defaultModel 生效
        decl.applyAllInstance();
        System.out.println("可用模型 = " + rt.modelRegistry().chatAliases());

        // 第一次对话：用默认模型
        AgentContextHolder h1 = AgentContextHolder.builder()
                .userId("u1")
                .historyId(IdGen.uuid())
                .runtime(rt)
                .modelAlias("deepseek-v3")
                .prompt("你是一个严谨的助手")
                .build();
        String a1 = new AutoAgent(h1).chat("用一句话解释什么是向量数据库");
        System.out.println("deepseek 回答 = " + a1);

        // 第二次对话：临时换成本地模型
        AgentContextHolder h2 = AgentContextHolder.builder()
                .userId("u1")
                .historyId(IdGen.uuid())
                .runtime(rt)
                .modelAlias("local-vllm")
                .prompt("你是一个严谨的助手")
                .build();
        String a2 = new AutoAgent(h2).chat("再换个说法解释一遍");
        System.out.println("vllm 回答 = " + a2);

        rt.close();
    }
}
```

## 跑起来会看到什么

- 应用配置时日志出现两条 `注册对话模型: deepseek-v3`、`注册对话模型: local-vllm`。
- 控制台：

```text
可用模型 = [deepseek-v3, local-vllm]
deepseek 回答 = 向量数据库是把文本、图片等转成向量...
vllm 回答 = 可将其想象为一个按"语义距离"排序的仓库...
```

两次对话走的是两个不同的模型：第一次是 `deepseek-v3`，第二次是 `local-vllm`，因为两次 `AgentContextHolder` 的 `modelAlias` 不同。

## 换个场景怎么改

**场景一：接入任何 OpenAI 兼容服务**

| 要改的点           | 怎么填                              |
| -------------- | -------------------------------- |
| `apiHost`      | 服务根地址，如 `https://api.openai.com` |
| `apiEndpoint`  | `/v1/chat/completions`           |
| `apiKey`       | 对应的 key，多数服务走 `Bearer` 头         |
| `apiModelName` | 服务方要求的模型名，如 `gpt-4o-mini`        |

填好这些字段即可使用，因为内核默认按 OpenAI 兼容格式发请求。

**场景二：格式非 OpenAI 兼容**

内核装配请求参数的逻辑集中在 `AgentRequestParamsFactory`：

- `AgentRequestParamsFactory.build(model, enableThinking)` 按一条模型配置构造出请求参数（`ApiRequestParams`）：思考模式 → 采样 → `customParams`，后写的覆盖先写的。
- 大多数"格式差异"其实只是**多几个私有字段**，这类用 `customParams` 直接透传就能解决。
- 如果差异大到要改请求体的结构，用 `IApiReqParamsCallback` 这个扩展接口——它只有一个方法 `callback(ApiRequestParams apiRequestParams)`，提供一次"拿到即将发出的参数、就地改它"的机会。它的具体挂载由调用链决定。

**场景三：控制采样与思考参数**

模型上的采样与思考字段经 `AgentRequestParamsFactory` 装配进 `ApiRequestParams`。一次对话使用 `AgentContextHolder.requestParams`，因此要使某次请求带上这些参数，需按模型装配后再置入：

```java
import net.itzq.mira.modules.ai.client.config.AgentRequestParamsFactory;
import net.itzq.mira.modules.ai.client.config.ModelApiConfig;
import net.itzq.mira.modules.ai.client.handle.ApiRequestParams;

ModelApiConfig model = rt.declaration().getModels().get(0);
ApiRequestParams params = AgentRequestParamsFactory.build(model, Boolean.TRUE); // 打开思考
holder.setRequestParams(params);
```

**场景四：换模型**

| 粒度     | 怎么改                                                                                                                       |
| ------ | ------------------------------------------------------------------------------------------------------------------------- |
| 改全局默认  | `agentConfig.setDefaultModel("local-vllm")` 再 `applyAllInstance()`；或直接 `rt.modelRegistry().setDefaultModel("local-vllm")` |
| 改这一次   | `holder.setModelAlias("local-vllm")`（构造时写 `.modelAlias(...)`）                                                             |
| 编排节点里改 | 用 `Invocation.modelAlias`，见[调用参数：这一次怎么跑](/MiraAgent/reference/orchestration/invocation.md)                                |

## 注意事项

- **`alias` 必须非空**：注册时 alias 为空的记录会被跳过并打日志。
- **null 不等于默认值**：采样参数设 null 表示不向服务端发该参数，由服务端使用其默认；不应以 0 代替。
- **`supportImage` 的 null**：JSON 导入得到的模型可能没带这个字段，此时按"支持图片"处理，不会报错。
- **默认模型没注册**：`agentConfig.defaultModel` 指向的别名如果不在 `models` 里，调用时会报"ChatService 未注册"。

> 相关：[接入向量模型](/MiraAgent/guide/config/embedding.md)、[配置导出与导入](/MiraAgent/guide/config/export-import.md)、[基础对话](/MiraAgent/guide/agent/basic-chat.md)
