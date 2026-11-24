> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 配置中心

配置快照（内核中称为 declaration）描述一个系统包含的内容：有哪些模型、向量模型、可用工具、系统提示词以及默认值。下文说明其包含的若干块、修改方式，以及修改后需要"应用"一次才生效的原因。

## 场景

以搭建一个 AI 应用为例：接入两个大模型、一个向量模型，指定一套系统提示词，并为工具预留若干开关。这些配置不写死在代码里，而是导出为一份文件，运维在界面修改后重启即生效。配置由一个统一的位置管理。

## 配置快照包含的块

`rt.declaration()` 返回的就是这份快照。它按块组织，每块管一类东西：

| 配置块                                 | 管什么                            |
| ----------------------------------- | ------------------------------ |
| `schemaVersion`                     | 快照结构版本，当前是 `1.0.0`             |
| `agentConfig`                       | 默认对话模型、默认向量模型、技能目录、工作空间目录、沙箱用户 |
| `workspaceConfig`                   | 数据根目录 `dataDir`、映射根目录 `mapDir` |
| `sseClientSimpleConfig`             | 模型接口的连接超时、读取超时、连接数等            |
| `models`                            | 对话模型清单（每个模型的地址、密钥、参数）          |
| `embeddings`                        | 向量模型清单                         |
| `prompts`                           | 基线系统提示词（可只给来源，让内核去装载）          |
| `toolProviders`                     | 工具来源清单：哪些工具由内核提供、哪些由应用提供       |
| `defaults`                          | 自定义的默认值键值袋（比如会话压缩策略参数）         |
| `plugins`                           | 每个插件一组设置值                      |
| `appSettingsSchema` / `appSettings` | 应用级设置项的结构与取值                   |

以下块作补充说明：

- **`agentConfig`** 里的 `defaultModel` 决定"不指定模型时用哪个"；`skillsDir` 默认是 `data-skills/skills`，`bashWorkspceDir` 默认是当前工作目录。默认模型、技能目录、工作空间目录均在此块中，**不在 `defaults` 里**。
- **`prompts`** 有两个字段：`baseSystem`（提示词全文）和 `source`（来源标识）。`baseSystem` 为空时，内核按 `source` 去装载，默认来源是 `core:assets/prompt/agent_system.md`。读当前生效的提示词用 `declaration().baseSystemPrompt()`。
- **`defaults`** 没有固定结构，是一个开放的键值袋。写入什么键，读取处就取什么键；内核约定的策略默认值（比如会话压缩的预算）也放这里。
- **`toolProviders`** 的每一项有 `type`（`builtin` / `http` / `mcp` / `skill`）和 `source`（`core` / `app:<应用 id>` / `plugin:<插件 id>`）。导入时内核会按它校验"这些 builtin 工具在不在"。

## 改了配置，要"应用"一次才生效

**`setXxx` 仅将值写入快照，不会自动注册到运行时。**

- `rt.declaration()`：拿到这份运行时的配置快照，改它。
- `rt.declaration().applyAllInstance()`：把快照里的**模型与向量模型**注册进这个运行时，并写入默认模型别名、默认向量模型别名。

所以顺序是：先 `rt.start()`，再逐块写配置，最后 `applyAllInstance()`。`rt.start()` 内部也会应用一次配置，所以若在 `start()` 之前就把配置写入快照（比如用 `declarationJson(json)` 启动），启动后就已生效。

## 配置快照完整结构

`exportDeclaration` 导出的就是这个形状，字段名与内核里的配置类一一对应：

```json
{
  "schemaVersion": "1.0.0",
  "agentConfig": {
    "defaultModel": "deepseek-v3",
    "defaultEmbedding": "bge-m3",
    "skillsDir": "data-skills/skills",
    "bashWorkspceDir": "D:/workspace/demo",
    "bashSandBoxUser": "sandbox"
  },
  "workspaceConfig": {
    "dataDir": "D:/my-app/data",
    "mapDir": ""
  },
  "sseClientSimpleConfig": {
    "connectTimeoutMs": 15000,
    "readTimeoutMs": 900000,
    "maxConnections": 100,
    "pooledConnectionIdleTimeoutMs": 60000,
    "keepAlive": false
  },
  "models": [
    {
      "alias": "deepseek-v3",
      "providerName": "deepseek",
      "apiModelName": "deepseek-chat",
      "apiHost": "https://api.deepseek.com",
      "apiEndpoint": "/v1/chat/completions",
      "apiKey": "sk-xxxx",
      "apiHeaders": {},
      "supportImage": 0,
      "contextWindowInput": 64000,
      "contextWindowOutput": 8000,
      "samplingTemperature": 0.3,
      "thinkingEnableParam": "{\"enable_thinking\":true}",
      "thinkingDisableParam": "{\"enable_thinking\":false}",
      "customParams": ""
    }
  ],
  "embeddings": [
    {
      "alias": "bge-m3",
      "apiUrl": "https://api.example.com/v1/embeddings",
      "apiKey": "sk-yyyy",
      "model": "bge-m3",
      "dimension": 1024,
      "connectTimeout": 30000,
      "readTimeout": 60000
    }
  ],
  "prompts": {
    "baseSystem": "",
    "source": "core:assets/prompt/agent_system.md"
  },
  "toolProviders": [
    {
      "type": "builtin",
      "source": "core",
      "names": ["Read", "Write", "Glob", "Grep"],
      "toolKeys": [],
      "servers": []
    }
  ],
  "defaults": {
    "session.compact.protectedTurns": 3,
    "session.compact.protectedTokens": 8000
  },
  "plugins": {
    "ai-chat-node": {
      "version": "1.0.0",
      "settings": { "llmTimeoutSeconds": 60, "enableThinking": true }
    }
  },
  "appSettingsSchema": [
    {
      "key": "webfetch.blockPrivateNetwork",
      "type": "boolean",
      "label": "禁止访问内网地址",
      "defaultValue": "true",
      "required": false
    }
  ],
  "appSettings": {
    "webfetch.blockPrivateNetwork": true
  }
}
```

## 最小可用代码

示例用代码拼出这份快照的核心部分，随后应用并查看结果：

```java
import net.itzq.mira.modules.runtime.KernelRuntime;
import net.itzq.mira.modules.config.AgentConfig;
import net.itzq.mira.modules.config.GlobalConfigManager;
import net.itzq.mira.modules.ai.client.config.ModelApiConfig;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

public class ConfigCenterDemo {

    public static void main(String[] args) {
        // 1) 建运行时，先装成默认运行时，再启动
        KernelRuntime rt = KernelRuntime.builder()
                .name("my-app")
                .dataDir("D:/my-app/data")
                .build();
        KernelRuntime.installDefault(rt);   // 必须在第一次读配置之前
        rt.start();

        // 2) 拿到这份运行时的配置快照，逐块写
        GlobalConfigManager decl = rt.declaration();

        ModelApiConfig deepseek = ModelApiConfig.builder()
                .alias("deepseek-v3")
                .providerName("deepseek")
                .apiModelName("deepseek-chat")
                .apiHost("https://api.deepseek.com")
                .apiEndpoint("/v1/chat/completions")
                .apiKey(System.getenv("DEEPSEEK_API_KEY"))
                .build();
        decl.setModels(new ArrayList<>(Arrays.asList(deepseek)));

        AgentConfig agentConfig = new AgentConfig();
        agentConfig.setDefaultModel("deepseek-v3");
        agentConfig.setSkillsDir("data-skills/skills");
        agentConfig.setBashWorkspceDir("D:/workspace/demo");
        decl.setAgentConfig(agentConfig);

        // 3) 关键一步：应用进这个运行时，模型才真正注册
        decl.applyAllInstance();

        System.out.println("已注册模型 = " + rt.modelRegistry().chatAliases());
        System.out.println("默认模型   = " + rt.modelRegistry().getDefaultModel());
        System.out.println("系统提示词长度 = " + decl.baseSystemPrompt().length());

        rt.close();
    }
}
```

## 跑起来会看到什么

- 启动时日志里能看到运行时名字和工具数量，比如 `已启动: name=my-app, tools=..., models=0`。
- 应用配置后，日志会出现 `注册对话模型: deepseek-v3`。
- 控制台打印：

```text
已注册模型 = [deepseek-v3]
默认模型   = deepseek-v3
系统提示词长度 = 1234
```

若将 `applyAllInstance()` 这一行去掉，`已注册模型` 会变成 `[]`，默认模型为空——这就是"改了配置没应用"的典型现象。

## 换个场景怎么改

**只用代码配 vs 从 JSON 文件读**

| 目标          | 怎么改                                             |
| ----------- | ----------------------------------------------- |
| 全部在代码里拼     | 像上面一样逐个 `setXxx` + `applyAllInstance()`         |
| 启动时带一份 JSON | 建运行时时加 `.declarationJson(json)`，启动即生效           |
| 运行中换一份 JSON | `rt.importDeclaration(json)`，它会自动应用进本运行时并返回校验报告 |

**把配置持久化到自己的数据库**

配置快照本身就是一个 JSON 字符串，所以持久化很简单：

1. 保存：`String json = rt.exportDeclaration(CredentialPolicy.INLINE);` 把 `json` 存进数据表。
2. 读取：从表里取回字符串，调 `rt.importDeclaration(json)`。
3. 启动装配：拿到运行时后先 `installDefault(rt)` 再 `rt.start()`，然后导入已保存的 JSON。

`CredentialPolicy` 决定密钥怎么落盘：`INLINE` 是原文（本机自用），`REFERENCE` 输出 `${ENV:...}` 占位（分发用），细节见[配置导出与导入](/MiraAgent/guide/config/export-import.md)。

## 注意事项

- **顺序**：`KernelRuntime.installDefault(rt)` 必须在第一次读配置之前调用，否则静态便捷入口会自己懒创建一个没数据目录的运行时。
- **改了要应用**：`setModels` / `setEmbeddings` / 改 `agentConfig.defaultModel` 之后，不调 `applyAllInstance()` 不会进注册表。
- **`setEmbeddings(null)` 会抛异常**（源码显式校验该参数不能为 null），清空需传入一个空列表。

> 相关：[接入一个模型](/MiraAgent/guide/config/models.md)、[接入向量模型](/MiraAgent/guide/config/embedding.md)、[内核运行时装配](/MiraAgent/guide/runtime/kernel-runtime.md)
