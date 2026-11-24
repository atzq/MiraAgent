> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 工具调用

内核将模型发出的工具调用请求转换为实际执行（读文件、改文件、运行命令），并通过白名单控制可用工具。

## 场景

AI 自行读文件、改文件、运行命令，而非仅作对话。例如用户说"看看 demo 目录下那个 Java 文件写了什么"，模型实际打开该文件读取内容并返回，而非编造。

## 最小可用代码

工具调用的完整闭环是这样的：

> **AI 说要调工具** → **内核执行工具** → **把结果回灌进历史** → **AI 拿着结果继续回答**

`holder.getTools()` 这个清单就是**当前会话的工具白名单**：不在清单里的工具，即使模型点名要调，内核也会回一条"工具不存在"。

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.runtime.KernelRuntime;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;

public class ToolChat {

    public static void main(String[] args) throws Exception {
        KernelRuntime rt = KernelRuntime.builder()
                .name("my-app")
                .dataDir("D:/my-app/data")
                .declarationJson("{\"schemaVersion\":\"1.0.0\",\"models\":[{\"alias\":\"deepseek-v3\","
                        + "\"providerName\":\"deepseek\",\"apiModelName\":\"deepseek-chat\","
                        + "\"apiHost\":\"https://api.deepseek.com\",\"apiEndpoint\":\"/chat/completions\","
                        + "\"apiKey\":\"sk-替换为实际密钥\"}],"
                        + "\"agentConfig\":{\"defaultModel\":\"deepseek-v3\"}}")
                .build();
        KernelRuntime.installDefault(rt);
        rt.start();

        String historyId = IdGen.uuid();
        AgentContextHolder holder = AgentContextHolder.builder()
                .userId("u1")
                .historyId(historyId)
                .modelAlias("deepseek-v3")
                .prompt("你是一个能读写本地文件的助手。要查文件内容，先调工具去看，不要凭猜测回答。")
                .workspacePath("D:/workspace/demo")
                .vfsId(historyId)
                .runtime(rt)
                .build();

        AutoAgent agent = new AutoAgent(holder);

        // 挂上"全量代码工具"：读取 + 搜索 + 写入 + 执行命令（白名单一次性写进 holder）
        agent.mountFullCodeTools();

        // 同步跑一轮：内核会自动来回多轮，直到模型给出最终回答
        String answer = agent.chat("看看 demo 目录下有哪些 Java 文件，挑一个读一下开头 20 行，告诉我它是干什么的");
        System.out.println(answer);

        rt.close();
    }
}
```

跑完可以打印一下白名单看看内核挂了哪些工具：

```java
for (String t : holder.getTools()) {
    System.out.println("白名单工具：" + t);
}
```

## 跑起来会看到什么

- 模型第一轮先要求调用 `Glob` 或 `list_files` 找文件；
- 内核执行工具（真的去读磁盘），把结果作为一条 `tool` 消息塞回历史；
- 因为历史末尾是工具结果，内核自动进入下一轮，把结果发给模型；
- 模型可能再调一次 `Read` 读文件内容；
- 拿到内容后，模型输出最终回答，循环结束。

整个过程是**内核自动循环**的，只调了一次 `chat()`。循环次数上限由 `agent.getMaxDepth()`（默认 999）控制。

## 换个场景怎么改

### 只读的代码问答机器人

把挂载那一行换成只读工具集即可，AI 就只能看、不能改：

```java
agent.mountReadOnlyTools();   // 只挂 Read / Glob / Grep / list_files / code_explorer + VFS 只读工具
```

### 能改代码的机器人

```java
agent.mountFullCodeTools();   // 只读 ∪ 写类（Bash / Edit / Write / vfs_file_write / vfs_file_edit）
```

判断当前这轮是不是只读，用静态方法查白名单：

```java
boolean readOnly = AutoAgent.isReadOnly(holder.getTools());
```

### 从基线工具集起步

若需要的是"内核自带工具 + 应用内自行注册的工具"这个合集，用 `systemBaselineTools`：

```java
import java.util.Arrays;
import java.util.List;

// 第一个参数是"应用内自行注册的工具名"，没有就传 null
List<String> whitelist = AutoAgent.systemBaselineTools(
        Arrays.asList("queryOrder", "refundOrder"));

// 一次性、幂等地挂进白名单
agent.mountToolsIfAbsent(whitelist.toArray(new String[0]));
```

`systemBaselineTools` 返回的是"内核默认代码工具（含写类）+ VFS 只读工具 + 传入的附加工具"，已经去重。只读基线改用手写的白名单过滤（例如 `AutoAgent.readOnlyTools()`）。

### 只放开几个工具

不挂整套工具，直接点名所需工具：

```java
agent.mountToolsIfAbsent("Read", "Glob", "Grep");
```

`mountToolsIfAbsent(...)` 是幂等的，重复调用不会把同一个工具挂两遍。

## 注意事项

- `currentDeep` 是"当前跑到第几轮循环"，从 1 开始，每执行完一轮工具调用后加 1。它主要给事件用的（`info.deep`），无需手动修改。
- 每个 `tool_call` 对应一条工具结果。用户中途停止或 hook 挂起时内核自动补占位，正常路径无需处理。
- 工具的执行结果会被统一包装：工具自报成败靠返回值第一行的 `[mira:ok]` / `[mira:err]` 标记，内核据此给前端一个可靠的红绿状态。
- 工具名大小写不敏感地参与白名单比对；按配置中的字符串书写可避免误配。

> 相关：[内置工具一览](/MiraAgent/guide/agent/builtin-tools.md)、[自定义工具](/MiraAgent/guide/agent/custom-tools.md)、[工具执行前钩子（EventHook）](/MiraAgent/guide/agent/event-hook.md)
