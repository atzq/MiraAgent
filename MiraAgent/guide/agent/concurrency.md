> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 并发执行与 ThreadLocal

内核的对话运行在线程池中。自行开启线程时涉及线程池选用，以及 ThreadLocal 跨线程丢失的问题。

## 场景

工具方法从 ThreadLocal 读取当前用户。单线程执行无异常；并发执行（如同时运行多个子代理）时，工具读到 `null`，报"拿不到当前用户"。

## ThreadLocal 丢失的原因

业务侧常见的写法：把当前用户、租户、traceId 放进线程，深层代码随时能取，不用一路改方法签名。

```java
/** 业务侧的用户上下文：放 ThreadLocal，哪一层代码都能取到 */
public final class UserContext {

    private static final ThreadLocal<String> USER_ID = new ThreadLocal<String>();

    public static void set(String userId) {
        USER_ID.set(userId);
    }

    public static String get() {
        return USER_ID.get();
    }

    public static void clear() {
        USER_ID.remove();
    }
}
```

ThreadLocal 是"跟线程绑定"的：主线程里 `set` 的值，只有主线程读得到。任务交给别的线程执行时，新线程的 ThreadLocal 是空的——所以工具里 `UserContext.get()` 拿到 `null`。

（`InheritableThreadLocal` 只在"新建线程"的那一刻继承一次；线程池里的线程是复用来的，照样不行。）

## 用内核提供的线程池

内核提供了 `ThreadPoolUtil`，它内部的线程池用 TTL（TransmittableThreadLocal）包了一层：提交任务时，会把提交线程的 ThreadLocal 一起带过去。

```java
import net.itzq.mira.modules.ai.utils.ThreadPoolUtil;

ThreadPoolUtil.getExecutorService().submit(new Runnable() {
    @Override
    public void run() {
        // 这里 UserContext.get() 读得到提交线程 set 的值
    }
});
```

配合子智能体并发执行的完整示例：

```java
import net.itzq.mira.modules.ai.agent.SubAgent;
import net.itzq.mira.modules.ai.utils.ThreadPoolUtil;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Future;

UserContext.set("u1");                                  // 主线程里存好当前用户
ExecutorService pool = ThreadPoolUtil.getExecutorService();

List<String> tasks = Arrays.asList("查一下今天的订单量", "查一下今天的退款量", "查一下今天的客诉量");
List<Future<String>> futures = new ArrayList<>();
for (String task : tasks) {
    final String t = task;
    futures.add(pool.submit(new java.util.concurrent.Callable<String>() {
        @Override
        public String call() {
            SubAgent worker = new SubAgent(holder, "worker", "你是数据助手，只输出结论。");
            worker.addSubTools("queryOrderCount", "queryRefundCount", "queryComplaintCount");
            return worker.execute(t);
        }
    }));
}

for (Future<String> f : futures) {
    System.out.println("—— 结论 ——\n" + f.get());
}
UserContext.clear();                                    // 线程池的线程是复用的，用完清掉
```

被调用的工具里就能正常取到用户了：

```java
@Tool(name = "queryOrderCount", description = "查询当前用户今天的订单量")
public String queryOrderCount() {
    String userId = UserContext.get();                  // 并发跑也拿得到
    if (userId == null) {
        return ToolCallResult.error("拿不到当前用户，检查是不是用了自己的线程池");
    }
    return ToolCallResult.success(orderService.countToday(userId) + " 单");
}
```

## 是否需要 ThreadPoolUtil

多数情况下无需使用。

| 情况                                           | 推荐                                  |
| -------------------------------------------- | ----------------------------------- |
| 任务里要靠 ThreadLocal 取东西（用户上下文、traceId、租户 id 等） | 使用 `ThreadPoolUtil`                 |
| 业务侧编写的工具（`Tool`）内部依赖 ThreadLocal             | 使用 `ThreadPoolUtil`；工具会被内核调度到其他线程执行 |
| 仅并发执行若干子任务且数据经方法参数传入                         | 无需使用，普通线程池亦可                        |
| 单线程串行调用（最常见）                                 | 不存在跨线程，无需使用                         |

内核不强制要求使用；是否使用取决于业务代码是否依赖 ThreadLocal。参考：内核自身仅在"流式对话"处用到它——`BasicAgent` / `BasicClient` 把一轮对话的执行体提交到该线程池；任务体携带 `contextHolder` 对象进入，全内核没有用 ThreadLocal 传上下文的代码。

## 跑起来会看到什么

- 用 `ThreadPoolUtil` 提交：工具的 `UserContext.get()` 拿到 `u1`，三个子任务都正常返回各自的结论。
- 改用 `Executors.newFixedThreadPool(3)` 提交时，同样的代码在工具中取到 `null`，返回"拿不到当前用户"。

## 换个场景怎么改

| 目标                            | 改哪儿                                              |
| ----------------------------- | ------------------------------------------------ |
| 在工具里拿会话上下文，而不是自己存 ThreadLocal | 工具方法参数里加一个 `AgentContextHolder`，内核自动传入会话上下文      |
| 仅并发执行若干子任务且数据经参数传入            | 用哪个池子都行，普通池子也能跑                                  |
| 已经有别处给的线程池，不方便换               | 用 TTL 的 `TtlExecutors` 把现有池子包一层再提交（依赖里已经带了），效果一样 |
| traceId 之类的链路信息跨线程传递          | 同样放 ThreadLocal，用 `ThreadPoolUtil` 提交即可          |

## 注意事项

- `ThreadPoolUtil` 的线程池全局共享；使用后不应 `shutdown()`，否则其他处无法再提交任务。
- ThreadLocal 使用后应调用 `remove()`；线程池线程复用，未清理会令上一个任务的值遗留到下一个任务，造成串号。
- 显式传参优先于 ThreadLocal：工具方法多接一个 `AgentContextHolder` 参数最稳妥；ThreadLocal 仅适用于无法修改方法签名的深层代码。
- 子智能体的并发：每个分身历史独立，可并发跑；分身的停止请求和事件都跟着主对话走。

> 相关：[子代理（SubAgent）](/MiraAgent/guide/agent/sub-agent.md) · [自定义工具](/MiraAgent/guide/agent/custom-tools.md) · [内核工具类](/MiraAgent/reference/utilities.md)
