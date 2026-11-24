> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 内核运行时装配

本文描述应用启动时装配内核的方式，以及装配后的使用与释放。

## 场景

服务常驻进程。进程启动时装配内核（配置模型、工具、挂接存储实现），之后每个请求复用同一运行时，进程退出时统一释放。

## 最小可用代码

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;
import net.itzq.mira.modules.ai.client.config.ModelApiConfig;
import net.itzq.mira.modules.runtime.KernelRuntime;

import java.util.Arrays;

public class ServerBootstrap {

    private KernelRuntime rt;

    /** 启动时装配一次 */
    public void onStartup() {
        // ① 启动：扫描内置工具 + 应用当前配置快照
        rt = KernelRuntime.builder()
                .name("my-app")
                .dataDir("D:/my-app/data")
                .build()
                .start();

        // ② 安装为默认运行时（必须在第一次读配置之前）
        KernelRuntime.installDefault(rt);

        // ③ 写配置
        rt.declaration().setModels(Arrays.asList(ModelApiConfig.builder()
                .alias("deepseek-v3")
                .providerName("deepseek")
                .apiModelName("deepseek-chat")
                .apiHost("https://api.deepseek.com")
                .apiEndpoint("/chat/completions")
                .apiKey(System.getenv("DEEPSEEK_API_KEY"))
                .build()));
        rt.declaration().getAgentConfig().setDefaultModel("deepseek-v3");
        rt.declaration().applyAllInstance();

        // ④ 挂上持久化实现（不挂 = 跑完即弃）
        rt.setPersistencePort(new MyPersistencePort());
    }

    /** 每个请求复用同一个运行时 */
    public String onRequest(String question) {
        String historyId = IdGen.uuid();
        AgentContextHolder holder = AgentContextHolder.builder()
                .userId("u1")
                .historyId(historyId)
                .modelAlias("deepseek-v3")
                .prompt("你是一个严谨的助手")
                .vfsId(historyId)
                .build();
        AutoAgent agent = new AutoAgent(holder, "demo-agent");
        return agent.chat(question);
    }

    /** 进程退出时释放 */
    public void onShutdown() {
        if (rt != null) {
            rt.close();
        }
    }

    // 占位：替换为实际实现（可参考 /guide/data/persistence）
    static class MyPersistencePort implements
            net.itzq.mira.modules.ai.persistence.PersistencePort {
        public java.util.List<net.itzq.mira.modules.ai.client.openai.chat.entity.ChatMessage>
                loadContext(String historyId) {
            return new java.util.ArrayList<>();
        }
        public void persistContext(
                String historyId,
                java.util.List<net.itzq.mira.modules.ai.client.openai.chat.entity.ChatMessage> messages) {
        }
        public void appendEvent(String historyId,
                net.itzq.mira.modules.ai.entity.chat.AnsResponse event) {
        }
        public void ensureSession(String historyId, String title, String mode) {
        }
    }
}
```

## builder 每个参数的含义

| 参数                           | 必填   | 含义                                           |
| ---------------------------- | ---- | -------------------------------------------- |
| `name(...)`                  | 否    | 运行时名字，只用于日志识别；不给默认叫 `runtime`                |
| `dataDir(...)`               | 基本必填 | 数据根目录，会话附件包等文件落在这里                           |
| `declarationJson(json)`      | 否    | 启动时直接带一份配置快照（JSON 字符串）                       |
| `persistencePort(port)`      | 否    | 挂持久化实现；不挂 = 跑完即弃                             |
| `vfsConfig(workspaceConfig)` | 否    | 显式指定文件读写的数据目录；不指定则跟随配置快照，配置快照没给就回落 `dataDir` |

`build()` 只是构造，还没启动；`build().start()` 或 `builder().start()` 会构造并启动。

## start() 做了什么

1. **扫描内置工具**：把内核自带的工具（文件读写、命令、检索、附件包、技能、MCP 等）注册进本运行时的工具注册表。
2. **应用配置快照**：把模型、embedding 装进本运行时的注册表，并设好默认模型。

`start()` 是幂等的：已启动过的实例再次调用无副作用，也**不会重新应用配置**。配置写入后通过显式调用 `applyAllInstance()` 生效。

## close() 释放了什么

`close()` 会清空工具注册表、清空模型与 embedding 注册表、关闭技能仓储（释放技能包文件句柄），并将状态标记为已关闭。适用于进程退出，或一次性临时运行时使用完毕后的释放。

## 访问器一览

| 访问器                   | 能拿到什么                               |
| --------------------- | ----------------------------------- |
| `declaration()`       | 配置快照：模型、embedding、提示词、工具来源、默认值、插件设置 |
| `modelRegistry()`     | 对话模型注册表：按别名查找模型                     |
| `embeddingRegistry()` | embedding 注册表                       |
| `toolRegistry()`      | 工具注册表：查工具、看数量、注册自定义工具               |
| `vfs()`               | 会话附件包服务：按会话 id 加载附件包                |
| `skills()`            | 技能仓储：按 slug 取技能、匹配技能                |
| `persistencePort()`   | 当前挂的持久化实现（没挂就是空实现）                  |

另有 `exportDeclaration(policy)` / `importDeclaration(json)` 用于配置快照的导出与导入，见[配置导出与导入](/MiraAgent/guide/config/export-import.md)。

## 跑起来会看到什么

启动时控制台打印：

```text
【KernelRuntime】已启动: name=my-app, tools=17, models=1
【KernelRuntime】默认运行时已安装: name=my-app, dataDir=D:/my-app/data
```

退出时打印：

```text
【KernelRuntime】已关闭: name=my-app
```

若挂接了持久化实现，一次对话结束后，历史消息和事件即写入存储。

## 换个场景怎么改

| 目标             | 改哪里                                                   |
| -------------- | ----------------------------------------------------- |
| 启动时就带一份现成配置    | 用 `builder().declarationJson(json)`                   |
| 配置由外部文件决定      | 用 `rt.importDeclaration(json)` 并检查返回的校验报告             |
| 不用默认运行时，全部显式指定 | 不调 `installDefault`；每个 context 加 `.runtime(rt)`       |
| 文件读写要落到别处      | 用 `builder().vfsConfig(workspaceConfig)` 指定 `dataDir` |
| 不要记忆           | 不挂持久化，或 `rt.setPersistencePort(null)`                 |
| 运行时用完只想清空      | `rt.close()`                                          |

## 失败模式

- **`installDefault` 须在第一次读取配置之前调用**：静态便捷入口会懒创建一套没有数据目录的默认运行时，若晚于首次读配置则被固定为无数据目录的实例。
- **写完配置须调用 `applyAllInstance()`**：仅调用 `start()` 不会重新应用配置。
- **`close()` 之后发起对话无效**：注册表已清空，`isClosed()` 为 true。
- **常驻服务只装配一次运行时**：每个请求都新建会导致文件句柄与注册表反复重建。

> 相关：[一个进程里跑多套运行时](/MiraAgent/guide/runtime/multi-instance.md)、[配置中心](/MiraAgent/guide/config/config-center.md)、[消息持久化](/MiraAgent/guide/data/persistence.md)
