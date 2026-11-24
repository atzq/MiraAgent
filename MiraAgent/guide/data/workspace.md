> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 工作空间（Workspace）

工作空间将 AI 的读文件、改文件、跑命令约束在指定的一个真实项目目录内。

## 场景

编程助手是一个典型场景：指定一个代码目录（如 `D:/projects/demo`），AI 在该目录内读文件、改文件、搜代码、跑构建命令。文件操作被约束在该目录内，不访问目录外的磁盘内容，代码内容也不必每次都放入提示词。

内核通过给会话上下文设置 `workspacePath` 约束文件操作范围，代码类工具（Bash、Read、Write、Edit、Glob、Grep 等）均基于该路径工作。

## 最小可用代码

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;
import net.itzq.mira.modules.runtime.KernelRuntime;

public class WorkspaceDemo {
    public static void main(String[] args) throws Exception {
        KernelRuntime rt = KernelRuntime.builder()
                .name("code-assistant")
                .dataDir("D:/code-assistant/data")
                .build()
                .start();

        AgentContextHolder holder = AgentContextHolder.builder()
                .userId("u1")
                .historyId(IdGen.uuid())
                .modelAlias("deepseek-v3")
                .prompt("你是一个严谨的编程助手，改动前后都要说明原因。")
                // 关键：AI 能碰的真实项目目录
                .workspacePath("D:/projects/demo")
                .runtime(rt)
                .build();

        AutoAgent agent = new AutoAgent(holder);
        // 全量工具：Bash / Edit / Read / Write / Glob / Grep / list_files / code_explorer
        agent.mountFullCodeTools();

        // 跑一轮"改文件"的对话
        agent.chatStream("把 README.md 里的项目名改成 demo-service").await();
    }
}
```

### 内核里还有一层"文件存储"抽象

上面的 `workspacePath` 是 AI 直接操作的真实目录。内核另外提供 `Workspace` 抽象，用于把文件存入内核管理的存储位置。两者都称为工作空间，但落点不同：

- `FileWorkspace`：把虚拟路径落到**真实磁盘**，目录是 `<dataDir>/ab/cd/<sessionId>/storage`。
- `VFS`：把内容落到**一个 zip 文件**（附件场景常用，见「会话附件」篇）。
- `LocalFileSpace`：直接以指定的**任意真实目录**为根，接口里用 `/` 分隔的虚拟路径，内部映射到真实文件。

`WorkspaceConfig` 控制落盘和展示：

```java
import net.itzq.mira.modules.runtime.KernelRuntime;
import net.itzq.mira.modules.workspace.WorkspaceConfig;

WorkspaceConfig wc = new WorkspaceConfig();
wc.setDataDir("D:/my-app/data"); // 文件落盘的根目录
wc.setMapDir("/workspace");      // 映射根目录（展示用）；不设时映射路径 == 真实路径

KernelRuntime rt = KernelRuntime.builder()
        .name("my-app")
        .dataDir("D:/my-app/data")
        .vfsConfig(wc)
        .build()
        .start();
```

`mapDir` 只影响"映射路径"这个概念：虚拟路径 `/data/a.txt` 的真实路径是 `<存储根>/data/a.txt`，映射路径是 `/workspace/data/a.txt`。`mapDir` 留空时，映射路径就等于真实路径。

以真实目录为根直接读写：

```java
import net.itzq.mira.modules.workspace.LocalFileSpace;
import net.itzq.mira.modules.workspace.Workspace;

Workspace wk = LocalFileSpace.open("D:/projects/demo"); // 目录必须已存在，不会自动创建
wk.write("/notes/todo.md", "待办：补测试");
System.out.println(wk.readString("/notes/todo.md"));
System.out.println(wk.listTree());
wk.close();
```

## 跑起来会看到什么

- AI 那轮对话里会连续出现工具调用：先用 `Read` 读 `README.md`，再用 `Edit`（或 `Write`）改内容，改完可能用 `Grep`/`Glob` 复查。
- 指定目录下的文件被修改：`D:/projects/demo/README.md`。
- AI 执行 `Bash` 时，输出中包含"工作目录 / 命令 / 退出码"以及命令输出；工作目录默认即 `workspacePath`。
- `LocalFileSpace` 那段会打印 `待办：补测试` 和一份目录树。

## 换个场景怎么改

1. **只读模式**：把 `agent.mountFullCodeTools()` 换成 `agent.mountReadOnlyTools()`，或白名单直接用 `AutoAgent.readOnlyTools()`——它不含 Bash、Edit、Write，AI 只能看不能改。
2. **限制 AI 仅访问子目录**：将 `workspacePath` 指向子目录即可，例如 `D:/projects/demo/src`。
3. **多用户各自一个目录**：每个用户的运行时给不同 `dataDir`，`workspacePath` 各自指定；同一个运行时下也可以按会话给不同目录。
4. **避免写入真实磁盘**：改用 VFS，内容落 zip，见「会话附件」篇。
5. **换一个"存储根"**：改 `WorkspaceConfig.dataDir`；仅需调整展示路径时改 `mapDir`。

## 注意事项

- **Bash 需要 workspacePath**：`workspacePath` 为空时，Bash 工具会直接拒绝执行并返回"当前未设置工作空间路径，禁用 Bash 工具"。
- **workdir 必须是已存在的目录**：Bash 的 `workdir` 参数给的是相对工作空间的路径或绝对路径，指向不存在的目录会报错；每条命令开启新的 shell，无法通过 `cd` 维持命令间的状态。
- **降权执行（可选）**：配置快照里的 `AgentConfig.bashSandBoxUser` 可以让 Bash 命令以指定用户身份执行；不设置时默认取环境变量 `SANDBOX_USER`，取不到就不降权。
- **危险命令只告警**：内核会识别 `rm -rf`、`git push --force`、`DROP TABLE` 这类模式并写日志，但默认不阻断；需在调用前拦截时，可用工具调用前钩子（`EventHook`）实现。
- **阻塞命令会被拒**：`sleep` 这类阻塞命令会被直接拦下。
- **越界防护**：存储层对虚拟路径做 `../` 逃逸检查，路径尝试超出存储根时抛出"非法路径（越界）"。

> 相关：[内置工具一览](/MiraAgent/guide/agent/builtin-tools.md) · [会话附件（VFS）](/MiraAgent/guide/data/vfs.md) · [接入一个模型](/MiraAgent/guide/config/models.md)
