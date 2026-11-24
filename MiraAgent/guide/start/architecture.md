> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 整体结构

组件分层、调用方向与一次对话涉及的 ID 标识

## 分层与调用方向

系统只有三层，调用方向自上而下，结果以事件形式自下而上回到应用。

```text
┌───────────────────────────────────────────────┐
│  应用（界面 / 业务 / 对外接口）              │
│  · 决定配置快照里有什么                          │
│  · 组装这一次怎么跑                              │
└───────────────┬───────────────────────────────┘
                │ 调用：装配 + 一次对话
                ▼
┌──────────────────────────────────────────────┐
│  内核运行时（KernelRuntime）                     │
│  · 配置快照 / 模型注册表 / 工具注册表            │
│  · 会话附件包（VFS）/ 技能 / 可选持久化          │
└───────────────┬───────────────────────────────┘
                │ 调用：发请求、读写文件、执行工具
                ▼
┌──────────────────────────────────────────────┐
│  大模型 API ／ 本地文件 ／ 会话附件包 ／ 工具 ／ MCP │
└───────────────┬───────────────────────────────┘
                │ 事件：思考、工具调用、正文、结束、报错
                ▲
                └──── 一路直播回应用（负责推给前端）
```

三层关系如下：

- 应用层只与内核运行时交互，不直接调用大模型或工具。
- 内核运行时只依赖自身的配置与扩展实现，不感知任何数据库或外部框架。
- 运行过程只以事件形式流出，由应用决定如何呈现。

## 三个核心资产的流转

内核在层间传递的只有三样东西：

```text
配置快照（系统里有什么）   调用参数（这次怎么跑）   事件（直播出来）
      ↓                        ↓                    ↑
  有哪些模型、            问什么、用哪个模型、     思考 / 调工具 /
  有哪些工具、           挂哪个会话、要不要附件    正文 / 结束 / 报错
  系统提示词是什么
```

流转顺序：

1. **配置快照**：应用启动时写入内核，描述"系统里有什么"。它是长期存在、可导出的。
2. **调用参数**：每次对话时临时组装，描述"这一次怎么跑"。用完即弃，不进入配置快照。
3. **执行**：内核按配置快照定位模型和工具，按调用参数执行一轮对话。
4. **事件**：运行过程中的每一个动作都以事件形式流出，由应用推送给前端。

配置快照在内核中对应的接口是 `declaration`，通过 `rt.declaration()` 获取。

## 场景

以搭建一个 AI 应用为例：前端一个对话框，后端一个接口。需要明确启动时做什么、一次对话传什么、以及过程如何呈现给用户。

## 最小可用代码

以下装配骨架展示各步骤的顺序，注释说明了顺序约束：

```java
import net.itzq.mira.modules.runtime.KernelRuntime;
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;

public class AppBootstrap {

    public static KernelRuntime bootstrap() {
        // 1. 创建运行时：先给数据目录，之后会话附件包等文件都落在它下面
        KernelRuntime rt = KernelRuntime.builder()
                .name("my-app")
                .dataDir("D:/my-app/data")
                .build()
                .start();                        // start 会扫描内置工具、应用当前配置快照

        // 2. 立刻安装为默认运行时
        //    为什么最早：静态便捷入口会懒创建一个"没有数据目录"的默认运行时，
        //    只要在它之前读到配置，那个空实例就被固定下来了。
        KernelRuntime.installDefault(rt);

        // 3. 写入配置快照（模型 / 提示词 / 工具来源）
        //    这里先用文字省略，模型篇有完整写法
        rt.declaration().getAgentConfig().setDefaultModel("deepseek-v3");

        // 4. 让配置生效：applyAllInstance 把模型、embedding 装进本运行时的注册表
        //    为什么在写完之后：先 apply 再写配置，注册表里还是空的。
        rt.declaration().applyAllInstance();

        // 5. 挂持久化（可选）：不挂 = 跑完即弃，没有记忆
        //    rt.setPersistencePort(new MyPersistencePort());

        return rt;
    }

    /** 一次对话 */
    public static void chatOnce(KernelRuntime rt, String question) {
        String historyId = IdGen.uuid();          // 会话 id：32 位无连字符

        AgentContextHolder holder = AgentContextHolder.builder()
                .userId("u1")
                .historyId(historyId)             // 这个会话是谁
                .modelAlias("deepseek-v3")        // 用哪个模型
                .prompt("你是一个严谨的助手")
                .vfsId(historyId)                 // 附件包 id：可以和会话 id 相同
                .build();                         // 不写 runtime → 用默认运行时

        AutoAgent agent = new AutoAgent(holder, "demo-agent");
        String answer = agent.chat(question);
        System.out.println(answer);
    }
}
```

## 跑起来会看到什么

启动时控制台会打出内核的日志，形如：

```text
【KernelRuntime】已启动: name=my-app, tools=17, models=1
【KernelRuntime】默认运行时已安装: name=my-app, dataDir=D:/my-app/data
```

`tools` 是扫描到的内置工具数量，`models` 为注册进去的模型别名数量。如果 `models=0`，说明第 3、4 步的顺序或写法有问题。

一次对话结束后，`chat` 返回模型最终的回答文本；若使用流式，则会在前端看到一串事件（见下文）。

## 需要实现的两个扩展接口

内核不感知数据库，也不感知工具包存储，因此有两件事只能由调用方提供。

| 扩展接口                   | 实现它可获得              | 不实现会怎样               |
| ---------------------- | ------------------- | -------------------- |
| 持久化（`PersistencePort`） | 会话有记忆、可回放、可列历史      | 内核用内置的空实现：跑完即弃，刷新就没了 |
| 工具包解析（`ToolResolver`）  | 编排里按"工具包 id"取到一组工具名 | 解析不出工具名，校验直接判失败（不猜）  |

- 持久化：`loadContext` 读历史、`persistContext` 存消息、`appendEvent` 逐条存事件、`ensureSession` 保证会话行存在。
- 工具包解析：给一个"工具包 id"集合，返回"工具名"集合；解析不到就让它失败，不要静默补默认值。

仅做普通对话、不做编排时，第二个接口可暂不实现。

## 一个会话里有哪几个 id

一次对话里会出现好几个 id，各自含义不同：

| id          | 含义                                              | 怎么来                |
| ----------- | ----------------------------------------------- | ------------------ |
| `historyId` | 会话 id：这轮对话属于哪个会话，历史按它存                          | `IdGen.uuid()`     |
| `vfsId`     | 附件包 id：读取"上传给 AI 的文件"用哪个包                       | 建议和 `historyId` 相同 |
| `agentId`   | 谁在回答：主代理还是某个子代理                                 | 内核自动生成             |
| `roundId`   | 模型循环里的问答轮次标识（提问>工具循环>回答算作一轮问答，从问到答期间roundId均一致） | 内核自动生成             |
| `step`      | 每轮问答期间工具循环时循环次数标识（Int数值递增）                      | 内核自动生成             |
| `msgId`     | 单条消息 id                                         | 内核自动生成             |

前两个在组装时提供，后三个由内核在运行中生成，随事件一起发给调用方。

## 换个场景怎么改

| 目标               | 改哪里                                                                                                     |
| ---------------- | ------------------------------------------------------------------------------------------------------- |
| 只跑一次脚本，不做常驻服务    | 不调 `installDefault`；用完直接 `rt.close()`                                                                   |
| 一个进程里跑多套互不干扰的运行时 | 不装默认（或最早期装其中一套），每个 context 显式 `.runtime(rt)`，见[一个进程里跑多套运行时](/MiraAgent/guide/runtime/multi-instance.md) |
| 每次对话换模型          | 改 `AgentContextHolder` 里的 `modelAlias`                                                                  |
| 让 AI 会读文件        | 给 agent 挂工具白名单，见[工具调用](/MiraAgent/guide/agent/tool-calling.md)                                          |
| 会话要有记忆           | `rt.setPersistencePort(...)`，见[消息持久化](/MiraAgent/guide/data/persistence.md)                             |
| 做流程编排            | 实现工具包解析，见[编排：让 AI 按流程跑](/MiraAgent/reference/orchestration/overview.md)                                 |

## 约束与顺序

- `installDefault` 需在首次读取配置前完成，否则默认运行时是被懒创建的空壳。
- `applyAllInstance` 需在写完配置后调用；`start()` 已启动过则不会再应用一次配置。
- 未写 `runtime` 的 context 会落到默认运行时；单套应用下可省略此步，多套隔离时必须显式指定运行时。

> 相关：[内核适用场景](/MiraAgent/guide/start/introduction.md)、[跑通第一次对话](/MiraAgent/guide/start/getting-started.md)、[内核运行时装配](/MiraAgent/guide/runtime/kernel-runtime.md)
