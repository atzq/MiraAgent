> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 一个进程里跑多套运行时

同一个 JVM 内可运行多套运行时，各用各自的模型、工具与文件，互不干扰。

## 场景

SaaS 后台可能在同一个进程内同时服务 A 客户与 B 客户。两方模型网关不同，上传的文件也需互不可见。为每个客户各起一个 JVM 通常不可行。

## 最小可用代码

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;
import net.itzq.mira.modules.ai.client.config.ModelApiConfig;
import net.itzq.mira.modules.runtime.KernelRuntime;

import java.util.Arrays;

public class TwoTenants {

    public static void main(String[] args) {
        // —— A 客户：自己的数据目录 + 自己的模型 ——
        KernelRuntime rtA = KernelRuntime.builder()
                .name("tenant-a")
                .dataDir("D:/data/tenant-a")     // 隔离靠它：文件都落在这里
                .build()
                .start();
        rtA.declaration().setModels(Arrays.asList(ModelApiConfig.builder()
                .alias("a-model")
                .providerName("deepseek")
                .apiModelName("deepseek-chat")
                .apiHost("https://api.deepseek.com")
                .apiEndpoint("/chat/completions")
                .apiKey(System.getenv("A_API_KEY"))
                .build()));
        rtA.declaration().getAgentConfig().setDefaultModel("a-model");
        rtA.declaration().applyAllInstance();

        // —— B 客户：另一个数据目录 + 另一个模型 ——
        KernelRuntime rtB = KernelRuntime.builder()
                .name("tenant-b")
                .dataDir("D:/data/tenant-b")
                .build()
                .start();
        rtB.declaration().setModels(Arrays.asList(ModelApiConfig.builder()
                .alias("b-model")
                .providerName("qwen")
                .apiModelName("qwen-plus")
                .apiHost("https://dashscope.aliyuncs.com/compatible-mode")
                .apiEndpoint("/v1/chat/completions")
                .apiKey(System.getenv("B_API_KEY"))
                .build()));
        rtB.declaration().getAgentConfig().setDefaultModel("b-model");
        rtB.declaration().applyAllInstance();

        // —— 两条会话各跑各的 ——
        System.out.println(chat(rtA, "tenant-a", "a-model"));
        System.out.println(chat(rtB, "tenant-b", "b-model"));
    }

    private static String chat(KernelRuntime rt, String user, String modelAlias) {
        String historyId = IdGen.uuid();
        AgentContextHolder holder = AgentContextHolder.builder()
                .userId(user)
                .historyId(historyId)
                .modelAlias(modelAlias)
                .prompt("你是一个严谨的助手")
                .vfsId(historyId)
                .runtime(rt)                     // 关键：显式指定用哪套运行时
                .build();
        AutoAgent agent = new AutoAgent(holder, "worker");
        return agent.chat("你好，你用的是哪个模型？");
    }
}
```

这段代码未调用 `installDefault`，因此两套运行时互不干扰，地位对等。

## 默认运行时

内核维护一个"默认运行时"位置。当某个 context 未显式指定 `runtime` 时，会使用默认运行时。

| 静态方法                               | 作用                 |
| ---------------------------------- | ------------------ |
| `KernelRuntime.installDefault(rt)` | 把某套运行时装到默认位置上      |
| `KernelRuntime.defaultRuntime()`   | 取默认运行时；没有就懒创建一套    |
| `KernelRuntime.hasDefault()`       | 是否已经装了默认运行时（不触发创建） |

**单套应用**：将运行时设为默认后，所有 context 可不写 `runtime`，统一落到该默认运行时。启动时调用 `installDefault(rt)` 即可。

**多套并存**：可在最早期将其中一套安装为默认，作为未显式指定 `runtime` 的兜底；每套自身的会话仍需显式写 `.runtime(...)`。

## 安装时机

`defaultRuntime()` 采用懒创建：若 `installDefault` 之前已有代码读取配置，内核会自行创建一套**没有数据目录**的默认运行时并固定下来。之后即使再调用 `installDefault` 替换，在此之前已读取旧实例的代码不会自动改回。

装配顺序应为：**先建运行时 → 立刻 `installDefault` → 再读任何配置**。偏离该顺序的常见表现为"模型列表为空"与"VFS 未初始化"。

## 数据隔离

隔离由各自的 `dataDir` 保证。每套运行时持有自己的配置快照、模型注册表、工具注册表、附件包服务、技能仓储。即使名称相同，只要根目录不同，**同名的会话 id / 附件包 id 在两边互不可见**。

```text
D:/data/tenant-a/  ← rtA 的附件包、工作空间
D:/data/tenant-b/  ← rtB 的附件包、工作空间
```

`historyId`、`vfsId` 本身不需要加任何前缀，隔离由根目录保证。

## 跑起来会看到什么

启动日志会分别打出两套：

```text
【KernelRuntime】已启动: name=tenant-a, tools=17, models=1
【KernelRuntime】已启动: name=tenant-b, tools=17, models=1
```

两条会话的回答分别来自各自的模型别名。A 客户问"你用的是哪个模型"、B 客户再问一次，两边会得到不同的模型名——因为各读各的模型注册表。

`D:/data/tenant-a` 与 `D:/data/tenant-b` 下的文件确实分属两套运行时。

## 换个场景怎么改

| 目标         | 改哪里                                               |
| ---------- | ------------------------------------------------- |
| 单套应用       | 启动时 `installDefault(rt)`，context 不写 `runtime`     |
| 每套用同一个模型网关 | 各自 `setModels` 时给不同别名，`modelAlias` 相应改            |
| 某一套不带记忆    | 那一套不挂持久化（另挂各自实现也互不影响）                             |
| 动态增删租户     | 用 Map 维护 `租户 id → KernelRuntime`，新租户现建现 `start()` |
| 释放某一套      | 单独 `rt.close()`，不影响其他                             |

### 默认运行时与显式指定

| 情形               | 用默认运行时 | 显式 `.runtime(rt)` |
| ---------------- | ------ | ----------------- |
| 单套应用             | 适用     | 可省                |
| 多套并存、要严格隔离       | 仅作兜底   | 必须                |
| 临时跑一段、调用方自己管生命周期 | 不适用    | 适用                |
| 测试里复现生产装配        | 启动时装一次 | 视情况               |

## 失败模式

- **多套并存且需隔离时，每处 context 均显式写 `.runtime(rt)`**；否则该 context 会落到默认运行时。
- **`installDefault` 仅在最早期调用一次**；在每次请求里反复调用会重建默认运行时。
- **临时运行时用完调用 `rt.close()`**；仍在被其他 context 使用的默认运行时若被关闭，依赖它的 context 将失效。
- **多套配置不通过静态便捷入口读写**：静态便捷入口只作用于默认运行时，多套场景使用 `rt.declaration()`。

> 相关：[内核运行时装配](/MiraAgent/guide/runtime/kernel-runtime.md)、[配置中心](/MiraAgent/guide/config/config-center.md)、[整体结构](/MiraAgent/guide/start/architecture.md)
