> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 子代理（SubAgent）

子代理用于把复杂子任务外包给独立上下文执行，仅将结论带回主对话，中间的搜索过程不进入主对话历史。

## 场景

一项任务涉及大量文档或代码：将会话附件里的若干文档读一遍并总结要点。主对话自行读取会把数万字的原文全部写入主历史，使 token 占用急剧上升。将此类任务交给子代理，仅由子代理带回一句结论，主历史不会增长。

## 最小可用代码

子代理只继承主对话的**环境设置**，不继承它的**聊天记录**：

| 继承过来                  | 各自独立                   |
| --------------------- | ---------------------- |
| 模型（`modelAlias`）      | 系统提示词（构造时传自己的）         |
| 工作目录（`workspacePath`） | 聊天历史（全新的空历史）           |
| 会话附件（`vfsId`）         | 工具清单（自己 `addSubTools`） |
| 事件中心、事件钩子、全局变量        |                        |

```java
import net.itzq.mira.modules.ai.agent.SubAgent;

// parentHolder 是主 Agent 的上下文（主 Agent 已创建、holder.topAgent 已就位）
public static void main(String[] args) throws Exception {
    // ... 建 rt / holder / 主 AutoAgent，同前面几篇 ...

    // 派一个分身：给它一个独立的人设（系统提示词）
    SubAgent summarizer = new SubAgent(
            holder,                                 // 父上下文：供它继承环境设置
            "doc-summarizer",                       // 分身名字
            "你是一个资料摘要助手。读完指定文档后，只输出不超过 5 条要点，不要复述原文。");

    // 给分身配工具（只给只读的，够它读文档就行）
    summarizer.addSubTools("vfs_list_files", "vfs_grep", "vfs_file_read");

    // 跑子任务：内部是流式执行并等它跑完，返回分身的最终回答
    String result = summarizer.execute("把会话附件里的 readme.md 总结成 5 条要点");
    System.out.println("分身结论：" + result);

    // 主对话拿到结论后，可以继续基于它做别的事
    String answer = agent.chat("根据刚才的摘要，帮我写一句推广文案");
    System.out.println(answer);
}
```

`withMaxDepth(int)` 可以给分身单独设一个循环上限：

```java
summarizer.withMaxDepth(20);
```

## 跑起来会看到什么

- 分身在**自己的历史**里来回调工具、读文档，主对话的历史里**不会多出那些文件内容**；
- 分身的每一步事件（思考、调工具）仍会发到同一个事件中心，前端可以按 `agentId` / `parentAgentId` 把它们归到"分身"这一条支线；
- `execute(...)` 返回后，主历史里体现为"主对话拿到了一个结论"，然后继续往下走。

## 换个场景怎么改

### 适用场景

| 场景            | 为什么用分身                                                           |
| ------------- | ---------------------------------------------------------------- |
| 搜索代码 / 大范围找东西 | 搜索过程会翻大量文件，过程不进入主历史（内核自带的 `code_explorer`、`vfs_explorer` 即采用此方式） |
| 长文档摘要         | 原文留在分身上下文里，主对话只拿结论                                               |
| 多路并行调研        | 几路互不干扰，各查各的，最后合并                                                 |

### 并行执行多个子任务

各分身历史独立，可并发执行；线程池的选用原则见[并发执行与 ThreadLocal](/MiraAgent/guide/agent/concurrency.md)。

```java
import net.itzq.mira.modules.ai.agent.SubAgent;
import net.itzq.mira.modules.ai.utils.ThreadPoolUtil;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Future;

List<String> tasks = Arrays.asList(
        "总结 reads/ 目录下所有接口文档，给出对外能力清单",
        "总结 reads/ 目录下所有部署文档，给出上线步骤",
        "总结 reads/ 目录下所有故障复盘，给出常见问题清单");

// 建议用内核自带的线程池：它被 TTL 包装过，业务/工具里的 ThreadLocal 能正确带到子线程
ExecutorService pool = ThreadPoolUtil.getExecutorService();
List<Future<String>> futures = new ArrayList<>();
for (String task : tasks) {
    final String t = task;
    futures.add(pool.submit(new java.util.concurrent.Callable<String>() {
        @Override
        public String call() {
            SubAgent worker = new SubAgent(holder, "worker", "你是一个调研助手，只输出结论。");
            worker.addSubTools("vfs_glob", "vfs_grep", "vfs_file_read");
            return worker.execute(t);
        }
    }));
}

for (Future<String> f : futures) {
    System.out.println("—— 结论 ——\n" + f.get());
}
// 这个池子是全局共享的，用完不要 shutdown()
```

## 注意事项

- 子代理构造时若 `parentHolder.getTopAgent()` 为空，则无法建立父子关系；需先创建主 Agent，再创建子代理。
- 线程池选择：内核不通过 ThreadLocal 传递上下文，但业务代码或自研工具可能依赖它；此类情况使用 `ThreadPoolUtil.getExecutorService()`，不依赖 ThreadLocal 时普通线程池亦可。
- 分身继承的是"全局变量"这个对象的引用，对它的修改是共享的；`tempVariables` 是各自独立的。
- 分身的事件、停止请求都跟着主对话走：主 Agent 被停，分身也停。
- 任务简单时由主对话直接执行更快；引入子代理会带来额外的上下文与调度开销。

> 相关：[工具调用](/MiraAgent/guide/agent/tool-calling.md)、[长对话压缩（ContextCompactor）](/MiraAgent/guide/agent/compaction.md)、[内置工具一览](/MiraAgent/guide/agent/builtin-tools.md)
