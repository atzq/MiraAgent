> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 跑通第一次对话

本篇介绍如何从零跑通第一次对话，只要一个 `main` 方法。

## 场景

先不考虑界面，就在本地写一个 Java 程序，发一句话给大模型，把回答打印出来。跑通了，再往上加工具和流式输出。

## 最小可用代码

### 第一步：引入依赖

```xml

<!-- 添加仓库 -->
<repositories>
    <repository>
        <id>itzq-repo</id>
        <url>https://maven.cnb.cool/itzq/repo/-/packages/</url>
    </repository>
</repositories>

<dependency>
    <groupId>net.itzq.mira</groupId>
    <artifactId>agent-core</artifactId>
    <version>0.2.1-SNAPSHOT</version>
</dependency>
```

> 已于多个场景验证，整体运行基本稳定，但目前仍处于持续迭代中，暂未正式发布。

### 第二步：一个 `main` 方法

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;
import net.itzq.mira.modules.ai.client.config.ModelApiConfig;
import net.itzq.mira.modules.runtime.KernelRuntime;

import java.util.Arrays;

public class FirstChat {

    public static void main(String[] args) {
        // 1. 创建运行时
        KernelRuntime rt = KernelRuntime.builder()
                .name("my-first-app")
                .dataDir("D:/my-app/data")       // 会话文件都落在这里
                .build()
                .start();                        // 扫描内置工具

        // 2. 安装为默认运行时（必须在第一次读配置之前）
        KernelRuntime.installDefault(rt);

        // 3. 声明一个模型
        rt.declaration().setModels(Arrays.asList(ModelApiConfig.builder()
                .alias("deepseek-v3")            // 别名：对话时用它
                .providerName("deepseek")
                .apiModelName("deepseek-chat")   // 厂商那边的真实模型名
                .apiHost("https://api.deepseek.com")
                .apiEndpoint("/chat/completions")
                .apiKey(System.getenv("DEEPSEEK_API_KEY"))
                .build()));

        // 4. 设为默认模型，并让配置生效
        rt.declaration().getAgentConfig().setDefaultModel("deepseek-v3");
        rt.declaration().applyAllInstance();

        // 5. 组装一次对话
        String historyId = IdGen.uuid();         // 32 位无连字符
        AgentContextHolder holder = AgentContextHolder.builder()
                .userId("u1")
                .historyId(historyId)
                .modelAlias("deepseek-v3")
                .prompt("你是一个严谨的助手")
                .workspacePath("D:/workspace/demo")
                .vfsId(historyId)
                .build();

        // 6. 开始对话
        AutoAgent agent = new AutoAgent(holder, "demo-agent");
        String answer = agent.chat("你好，用一句话介绍你自己");
        System.out.println(answer);
    }
}
```

### 加一点：让 AI 会读文件

`AutoAgent` 默认不挂任何工具。挂上只读工具集，它就能看文件了：

```java
AutoAgent agent = new AutoAgent(holder, "demo-agent")
        .mountToolsIfAbsent(AutoAgent.readOnlyTools().toArray(new String[0]));
```

`mountToolsIfAbsent` 是幂等的：已经挂过的工具不会重复挂。

### 再加一点：流式输出

`chatStream` 是异步的，返回 `CountDownLatch`，要等它。事件通过 `EventCenter` 出来：

```java
import net.itzq.mira.modules.ai.agent.event.DefaultEventListener;
import net.itzq.mira.modules.ai.agent.event.EventCenter;
import net.itzq.mira.modules.ai.agent.event.type.ErrorEvent;
import net.itzq.mira.modules.ai.entity.chat.AnsResponse;

// 一条会话一条事件通道
EventCenter eventCenter = new EventCenter();
eventCenter.register(new DefaultEventListener() {
    @Override
    public void sseEmitterSend(Object response) {
        if (response instanceof AnsResponse) {
            AnsResponse res = (AnsResponse) response;
            // 按 type 分流渲染：ChatReasonContent 思考、ChatContent 正文……
            System.out.println(res.getType() + " -> " + res.getAnswer());
        }
    }

    @Override
    public void onError(ErrorEvent event) {
        event.getThrowable().printStackTrace();
    }
});
holder.setEventCenter(eventCenter);

agent.chatStream("你好，用一句话介绍你自己").await();   // 等到本轮结束
```

## 跑起来会看到什么

启动时控制台会打印：

```text
【KernelRuntime】已启动: name=my-first-app, tools=17, models=1
【KernelRuntime】默认运行时已安装: name=my-first-app, dataDir=D:/my-app/data
```

- 同步 `chat`：最后打印一行模型回答的文本。
- 流式 `chatStream`：会连续打印很多行，`ChatReasonContent` 是思考过程，`ChatContent` 是正文，最后一条是 `ChatEnd`。
- 如果挂了工具，中间还会穿插 `CallToolBegin` / `CallToolEnd`。

## 换个场景怎么改

| 目标              | 改哪里                                                                        |
| --------------- | -------------------------------------------------------------------------- |
| 换成别的厂商模型        | 改 `ModelApiConfig` 的 `apiHost` / `apiEndpoint` / `apiModelName` / `apiKey` |
| 换模型不解绑代码        | 新加一个 `ModelApiConfig`，对话时改 `holder` 的 `modelAlias`                         |
| 让 AI 改文件、执行命令   | 换成挂全量工具：`agent.mountFullCodeTools()`                                       |
| 让 AI 在指定项目目录里干活 | 改 `holder` 的 `workspacePath`                                               |
| 给 AI 上传文件       | 用 `vfsId` 对应的附件包写文件，见[会话附件（VFS）](/MiraAgent/guide/data/vfs.md)             |
| 要永久记住历史         | 挂持久化，见[消息持久化](/MiraAgent/guide/data/persistence.md)                        |
| 换系统提示词          | 改 `holder` 的 `prompt`，或改配置快照里的提示词                                          |

## 注意事项（常见报错排查）

| 现象                           | 原因                                                      | 修法                                                                   |
| ---------------------------- | ------------------------------------------------------- | -------------------------------------------------------------------- |
| `tools=..., models=0`，模型列表为空 | 写完配置没调 `applyAllInstance()`；或 `installDefault` 晚于第一次读配置 | 写配置后调 `rt.declaration().applyAllInstance()`；把 `installDefault` 放到最前面 |
| 报错 `VFS 未初始化：dataDir 为空`     | 没装默认运行时，或运行时的 `dataDir` 为空                              | 启动时 `installDefault(rt)` 且给 `dataDir`                                |
| 报错说 sessionId 必须是 32 位       | 用了带连字符的 `UUID.randomUUID()`                             | 用 `IdGen.uuid()`                                                     |
| 模型网关报 `duplicate names`      | 工具白名单里有重复                                               | 用 `mountToolsIfAbsent`，自己拼白名单时先去重                                    |
| 工具调用参数全是 null                | 编译没保留参数名                                                | 编译插件保留 `<parameters>true</parameters>`                               |
| 流式调用像没反应                     | 忘了等 `CountDownLatch`                                    | `agent.chatStream(...).await()`                                      |
| `apiKey` 为 null 请求被拒         | 环境变量没读到                                                 | 确认 `System.getenv("DEEPSEEK_API_KEY")` 有值                            |

> 相关：[整体结构](/MiraAgent/guide/start/architecture.md)、[内核运行时装配](/MiraAgent/guide/runtime/kernel-runtime.md)、[基础对话](/MiraAgent/guide/agent/basic-chat.md)
