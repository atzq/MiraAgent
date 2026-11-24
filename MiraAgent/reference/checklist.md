> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 上层应用接入清单

按顺序排列的待办清单，逐项完成后形成应用骨架。

## 场景

基于内核构建应用的接入步骤汇总。清单按执行顺序排列，逐项列出要写的 API、原因与对应文档。

## 清单

按顺序排列，括号内为这一步产出的内容。

### 1. 声明运行时

| 项   | 内容                                                               |
| --- | ---------------------------------------------------------------- |
| 做什么 | `KernelRuntime.builder().name(...).dataDir(...).build().start()` |
| 为什么 | 内核的一切都从这套运行时取；数据目录决定文件落哪                                         |
| API | `KernelRuntime.builder()` / `KernelRuntime.start()`              |
| 文档  | [内核运行时装配](/MiraAgent/guide/runtime/kernel-runtime.md)            |

### 2. 设置默认运行时

| 项   | 内容                                                    |
| --- | ----------------------------------------------------- |
| 做什么 | 紧接着 `KernelRuntime.installDefault(rt)`                |
| 为什么 | 不带 `runtime` 的上下文会落到默认运行时；晚装会被懒创建的空实例抢先               |
| API | `KernelRuntime.installDefault(...)`                   |
| 文档  | [内核运行时装配](/MiraAgent/guide/runtime/kernel-runtime.md) |

### 3. 写配置（模型 / 提示词 / 工具来源）

| 项   | 内容                                                                                                                                            |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| 做什么 | 往 `rt.declaration()` 里写模型、提示词、工具来源、默认模型                                                                                                       |
| 为什么 | 配置快照说明"这个世界有什么"，是后面一切查找的依据                                                                                                                    |
| API | `declaration().setModels(...)` / `setEmbeddings(...)` / `setPrompts(...)` / `setToolProviders(...)` / `getAgentConfig().setDefaultModel(...)` |
| 文档  | [配置中心](/MiraAgent/guide/config/config-center.md)、[接入一个模型](/MiraAgent/guide/config/models.md)                                                  |

### 4. 应用配置

| 项   | 内容                                               |
| --- | ------------------------------------------------ |
| 做什么 | 配置写完后调 `rt.declaration().applyAllInstance()`     |
| 为什么 | 不应用，模型/embedding 不会进注册表，模型列表就是空的                 |
| API | `declaration().applyAllInstance()`               |
| 文档  | [配置中心](/MiraAgent/guide/config/config-center.md) |

### 5. 实现持久化

| 项   | 内容                                                                           |
| --- | ---------------------------------------------------------------------------- |
| 做什么 | 实现 `PersistencePort`，挂到 `rt.setPersistencePort(...)`                         |
| 为什么 | 不挂 = 跑完即弃，没有记忆、不能回放                                                          |
| API | `PersistencePort.loadContext / persistContext / appendEvent / ensureSession` |
| 文档  | [消息持久化](/MiraAgent/guide/data/persistence.md)                                |

### 6. 实现工具包解析（要做编排才需要）

| 项   | 内容                                                                            |
| --- | ----------------------------------------------------------------------------- |
| 做什么 | 实现 `ToolResolver`：把"工具包 id"换成"工具名集合"                                          |
| 为什么 | 编排里按工具包 id 取工具，解析不出就判失败，不猜默认值                                                 |
| API | `ExecutionResolver.resolve(blueprint, invocation, toolResolver, declaration)` |
| 文档  | [执行前先校验](/MiraAgent/reference/orchestration/execution-resolver.md)            |

### 7. 组装一次对话

| 项   | 内容                                                                                                 |
| --- | -------------------------------------------------------------------------------------------------- |
| 做什么 | 建 `AgentContextHolder`（会话 id、模型别名、提示词、工作空间），再 `new AutoAgent(holder)` 挂工具                          |
| 为什么 | 调用参数说明"这一次怎么跑"，用完即弃，不进配置快照                                                                         |
| API | `AgentContextHolder.builder()`、`AutoAgent.mountToolsIfAbsent(...)`、`chat(...)` / `chatStream(...)` |
| 文档  | [基础对话](/MiraAgent/guide/agent/basic-chat.md)、[工具调用](/MiraAgent/guide/agent/tool-calling.md)        |

### 8. 把事件推给前端

| 项   | 内容                                                                                      |
| --- | --------------------------------------------------------------------------------------- |
| 做什么 | 注册 `EventCenter` + 监听器，把 `AnsResponse` 推给前端；先落库再推送                                      |
| 为什么 | 思考过程、工具调用、正文、结束都靠事件流出；重连时用事件 id 补推                                                      |
| API | `EventCenter.register(...)`、`DefaultEventListener.sseEmitterSend(...)` / `onError(...)` |
| 文档  | [流式输出](/MiraAgent/guide/agent/streaming.md)                                             |

### 9. 附件与工作空间

| 项   | 内容                                                                                              |
| --- | ----------------------------------------------------------------------------------------------- |
| 做什么 | 用 `rt.vfs()` 读写会话附件包；用 `holder.workspacePath` 指定 AI 能直接操作的真实目录                                  |
| 为什么 | 附件包按会话隔离、懒加载；工作空间给工具一个可读写的根                                                                     |
| API | `vfs().load(sessionId)` / `loadVfs(sessionId)`、`Workspace.write / readString`                   |
| 文档  | [会话附件（VFS）](/MiraAgent/guide/data/vfs.md)、[工作空间（Workspace）](/MiraAgent/guide/data/workspace.md) |

### 10. 技能与 MCP

| 项   | 内容                                                                                                       |
| --- | -------------------------------------------------------------------------------------------------------- |
| 做什么 | 解析本次激活的技能，把技能/MCP 配套工具并进工具白名单；预解析 MCP 配置                                                                 |
| 为什么 | 技能让 AI 按需加载"说明书"，MCP 接外部工具服务                                                                             |
| API | `AutoAgent.resolveActiveSkills(holder)`、`AutoAgent.mountSkillAndMcpTools(...)`、`McpManager.prepare(...)` |
| 文档  | [技能包（Skills）](/MiraAgent/guide/extensions/skills.md)、[接入 MCP 工具服务](/MiraAgent/guide/extensions/mcp.md)   |

### 11. 中断与续行

| 项   | 内容                                                                                                        |
| --- | --------------------------------------------------------------------------------------------------------- |
| 做什么 | 用 `EventHook.onBeforeToolCall` 决定放行 / 代答 / 挂起；挂起后补结果再续行                                                   |
| 为什么 | 需要人工审批、向用户追问时，不能让对话直接跑完                                                                                   |
| API | `HookResult.respond(...)` / `HookResult.suspend()`、`chatStreamResume()` / `chatSyncResume()`              |
| 文档  | [停止与续跑](/MiraAgent/guide/agent/stop-resume.md)、[工具执行前钩子（EventHook）](/MiraAgent/guide/agent/event-hook.md) |

### 12. 配置导出导入

| 项   | 内容                                                                                     |
| --- | -------------------------------------------------------------------------------------- |
| 做什么 | 导出配置快照给别人用，或导入一份现成配置并检查校验报告                                                            |
| 为什么 | 分发时用引用形态避免泄密；导入时缺什么要能看见                                                                |
| API | `exportDeclaration(CredentialPolicy)` / `importDeclaration(json)` → `ValidationReport` |
| 文档  | [配置导出与导入](/MiraAgent/guide/config/export-import.md)                                    |

### 13. 收尾释放

| 项   | 内容                                                    |
| --- | ----------------------------------------------------- |
| 做什么 | 进程退出时 `rt.close()`                                    |
| 为什么 | 释放工具/模型注册表与技能仓储持有的文件句柄                                |
| API | `KernelRuntime.close()`                               |
| 文档  | [内核运行时装配](/MiraAgent/guide/runtime/kernel-runtime.md) |

## 最小可用代码

清单前 7 步连成的骨架，后续步骤按上表文档补全：

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;
import net.itzq.mira.modules.ai.client.config.ModelApiConfig;
import net.itzq.mira.modules.runtime.KernelRuntime;

import java.util.Arrays;

public class AppSkeleton {

    public static void main(String[] args) {
        // 1 + 2：装运行时，立刻设置默认
        KernelRuntime rt = KernelRuntime.builder()
                .name("my-app")
                .dataDir("D:/my-app/data")
                .build()
                .start();
        KernelRuntime.installDefault(rt);

        // 3：写配置
        rt.declaration().setModels(Arrays.asList(ModelApiConfig.builder()
                .alias("deepseek-v3")
                .providerName("deepseek")
                .apiModelName("deepseek-chat")
                .apiHost("https://api.deepseek.com")
                .apiEndpoint("/chat/completions")
                .apiKey(System.getenv("DEEPSEEK_API_KEY"))
                .build()));
        rt.declaration().getAgentConfig().setDefaultModel("deepseek-v3");

        // 4：应用配置
        rt.declaration().applyAllInstance();

        // 5：挂持久化（示例省略实现）
        // rt.setPersistencePort(new MyPersistencePort());

        // 7：组装一次对话
        String historyId = IdGen.uuid();
        AgentContextHolder holder = AgentContextHolder.builder()
                .userId("u1")
                .historyId(historyId)
                .modelAlias("deepseek-v3")
                .prompt("你是一个严谨的助手")
                .vfsId(historyId)
                .build();
        AutoAgent agent = new AutoAgent(holder, "worker")
                .mountToolsIfAbsent(AutoAgent.readOnlyTools().toArray(new String[0]));

        System.out.println(agent.chat("你好，介绍一下你自己"));

        // 13：收尾
        rt.close();
    }
}
```

## 运行输出

控制台依次出现：

```text
【KernelRuntime】已启动: name=my-app, tools=17, models=1
【KernelRuntime】默认运行时已安装: name=my-app, dataDir=D:/my-app/data
（模型回答的文本）
【KernelRuntime】已关闭: name=my-app
```

做到了第 8 步以后，前端会在一次对话中收到一串事件；做到了第 5 步以后，存储里会多出消息和事件记录。

## 换个场景怎么改

| 应用形态        | 清单怎么裁剪                                                                                          |
| ----------- | ----------------------------------------------------------------------------------------------- |
| 只是一个脚本，跑完就走 | 只做 1、3、4、7、13；不挂持久化                                                                             |
| 常驻服务、单租户    | 全做；8 改成推给长连接                                                                                    |
| 多租户 SaaS    | 2 改成"最早期装一套兜底"，其余每处显式 `.runtime(rt)`，见[一个进程里跑多套运行时](/MiraAgent/guide/runtime/multi-instance.md) |
| 纯对话，不做编排    | 跳过第 6 步                                                                                         |
| 只做流程编排执行器   | 重点做 3、4、6、8、12；对话组装收敛到节点执行器里                                                                    |

## 顺序错了会怎样

| 现象                    | 原因                          | 修法                             |
| --------------------- | --------------------------- | ------------------------------ |
| 模型列表为空                | 写完配置没 `applyAllInstance`    | 在写配置之后调一次                      |
| 模型列表为空（换写法仍为空）        | `installDefault` 晚于第一次读配置   | 把 `installDefault` 提到最前面       |
| `VFS 未初始化：dataDir 为空` | 没设置默认运行时，或 `dataDir` 没给     | 启动时设置默认运行时并给数据目录               |
| 会话没有记忆                | 没挂持久化                       | `rt.setPersistencePort(...)`   |
| 前端工具块一直转圈             | 工具事件没配齐成对                   | 每条不进正常流程的分支也补发一对开始/结束          |
| 前端刷新后消息丢了             | 事件没先落库，或没有补推                | 监听器里先落库再推，重连带事件 id 补推          |
| 编排里工具包解析不出来           | 没实现 `ToolResolver`，或解析失败被吞掉 | 实现它，解析不到就让它失败                  |
| 全量工具挂上去后 AI 乱改文件      | 白名单里带了写类工具                  | 只用只读集合，或去掉写类工具                 |
| 多租户互相看到对方文件           | 数据目录相同，或 context 没显式指定运行时   | 各自 `dataDir`，每处 `.runtime(rt)` |

> 相关：[内核运行时装配](/MiraAgent/guide/runtime/kernel-runtime.md)、[跑通第一次对话](/MiraAgent/guide/start/getting-started.md)、[内核工具类](/MiraAgent/reference/utilities.md)
