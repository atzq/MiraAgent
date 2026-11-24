> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 会话附件（VFS）

会话附件将用户上传的文件抽取为纯文本后存储，AI 可在本次会话内读取、搜索。

> 内核只提出了一种附件的简易存储方案，构建了一个小型虚拟”文件系统“充当”知识库“，不涉及把上传的文件变成文本。文本转换通常借助视觉模型、光学字符识别（OCR）、MinerU、Docling等框架辅助实现。

## 场景

以带附件上传的助手为例：用户在对话中提交一个 PDF、一个 Word 或一张图片，需要先将其抽取为 markdown（或任意纯文本），因为 AI 无法直接读取原始二进制字节。抽取后产生两个问题：

1. 这些文本存于何处？全部放入系统提示词会使提示词过长，且多数情况下并不需要。
2. 不同用户、不同会话的附件需彼此隔离。

内核负责此事的组件是 VFS：一次会话对应一个 zip 文件，zip 内为一个普通文本文件系统。AI 可列目录、按关键字搜索、按路径读取文件。

## 最小可用代码

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;
import net.itzq.mira.modules.runtime.KernelRuntime;
import net.itzq.mira.modules.vfs.VFS;
import net.itzq.mira.modules.vfs.model.SearchResult;

import java.util.Arrays;
import java.util.List;

public class VfsDemo {
    public static void main(String[] args) throws Exception {
        // 1. 起运行时：dataDir 就是 VFS 的落盘根目录
        KernelRuntime rt = KernelRuntime.builder()
                .name("my-app")
                .dataDir("D:/my-app/data")
                .build()
                .start();

        // 2. 一次会话一个 id，必须是 IdGen.uuid()（32 位、无连字符）
        String sessionId = IdGen.uuid();

        // 3. 取会话文件系统。此刻磁盘上还没有 zip，只是算好了路径
        VFS vfs = rt.vfs().loadVfs(sessionId);
        try {
            // 4. 存附件：把抽取好的纯文本写进去
            //    addDocument 返回虚拟路径（docId），后续都按这个路径读
            String md = "# 季度报告\n本季度营收同比增长 12%。";
            String docPath = vfs.addDocument(
                    "report.md",           // 文件名
                    md.getBytes("UTF-8"),  // 源字节；传 null 时用 textChunks 拼出来的文本代替
                    "/docs",               // 目录
                    Arrays.asList(md));    // 分段文本，这里只有一段
            System.out.println("已存入: " + docPath);

            // 也可以直接按路径写一段文本
            vfs.write("/docs/note.md", "这是会议备注。");

            // 5. 读回来
            System.out.println("内容: " + vfs.readString("/docs/report.md"));

            // 6. 看目录树
            System.out.println("目录树:\n" + vfs.listTree());

            // 7. 按正则搜关键字，返回命中的文件列表
            List<SearchResult> hits = vfs.grep("营收", 10);
            for (SearchResult r : hits) {
                System.out.println("命中: " + r.getFilePath() + " 来源=" + r.getSource());
            }
        } finally {
            // 8. 释放底层 zip 句柄
            vfs.close();
        }

        // 9. 让 AI 用起来：把同一个 id 写进会话上下文，挂 VFS 工具
        AgentContextHolder holder = AgentContextHolder.builder()
                .userId("u1")
                .historyId(sessionId)      // 历史 id
                .vfsId(sessionId)          // 文件 id：一个会话对应一个
                .modelAlias("deepseek-v3")
                .runtime(rt)
                .build();

        AutoAgent agent = new AutoAgent(holder);
        agent.mountToolsIfAbsent(AutoAgent.getDefaultVfsReadTools());
        agent.chatStream("帮我看看 /docs/report.md 里营收涨了多少").await();
    }
}
```

### 说明几件事

- zip 的落盘路径规则是 `<dataDir>/ab/cd/<sessionId>.zip`：取 sessionId 的前 2 位做第一级目录、第 3\~4 位做第二级目录，避免所有 zip 堆在一个目录里。上面的例子会落到 `D:/my-app/data/<前2位>/<第3~4位>/<sessionId>.zip`。
- `vfsId` 与 `historyId` 使用同一个 sessionId 是常见做法。两者也可分开——内核不强制相等，调用方保证"一个会话对应一个 id"时，工具即可定位同一份文件。
- AI 侧通过 `holder.getVfsId()` 获取 id。挂载 `vfs_file_read`、`vfs_list_files`、`vfs_grep`（需要写入时再挂载 `vfs_file_write`、`vfs_file_edit`、`vfs_glob`、`vfs_explorer`）后，AI 即可用这些工具读取附件。id 为空时，工具直接返回"虚拟文件系统未初始化"。

## 跑起来会看到什么

- 控制台依次打印 `已存入: /docs/report.md`、文件内容、目录树，以及 `命中: /docs/report.md 来源=grep`。
- 目录树里能看到 `/`、`/docs/`、`/docs/report.md`、`/docs/note.md`。
- 磁盘上出现一个 zip：`<dataDir>/ab/cd/<sessionId>.zip`。
- 时间点：`loadVfs` 步骤不产生任何文件；直到首次 `addDocument` 或 `write`，zip 才真正创建。
- 最后那轮对话里，可看到 AI 调用 `vfs_file_read` 之类的工具，把文件内容读进上下文再回答。

## 换个场景怎么改

| 目标                | 改哪里                                                                                      |
| ----------------- | ---------------------------------------------------------------------------------------- |
| 一个用户有多个会话         | 每个会话各自 `IdGen.uuid()` 当 sessionId；用数据表存储"用户 ↔ 会话 ↔ sessionId"的对应关系                       |
| 附件只读，不许 AI 改      | 工具白名单中不挂 `vfs_file_write`、`vfs_file_edit`；仅保留 `AutoAgent.getDefaultVfsReadTools()` 返回的工具 |
| 附件也要能被 AI 编辑      | 白名单再加上 `vfs_file_write`、`vfs_file_edit`                                                  |
| 换个落盘位置            | 改运行时的 `dataDir`（或用 `WorkspaceConfig` 单独给 VFS 指定数据目录）                                     |
| 若需让 AI 直接操作真实项目目录 | 属于另一类工作空间，见「工作空间」篇                                                                       |

## 注意事项

- **sessionId 格式**：必须是 32 位无连字符的十六进制字符串，用 `IdGen.uuid()` 生成；传别的（比如带连字符的 UUID）会直接抛异常。
- **懒加载**：读不存在的文件会抛 `NoSuchFileException`，不会自动创建空文件。不确定是否存在时，可先用 `vfs.exists(path)` 判断；`listTree()` 在没有 zip 时返回空字符串。
- **覆盖写**：`addDocument` 和 `write` 都是覆盖语义，同一个虚拟路径再写一次会替换掉旧内容。
- **用完需关闭**：VFS 实现了 `Closeable`，使用后应调用 `close()`，或用 try-with-resources 管理。
- **路径用正斜杠**：虚拟路径统一以 `/` 开头（如 `/docs/report.md`），内部会把反斜杠和多余的斜杠归一化。

> 相关：[内置工具一览](/MiraAgent/guide/agent/builtin-tools.md) · [工作空间（Workspace）](/MiraAgent/guide/data/workspace.md) · [内核运行时装配](/MiraAgent/guide/runtime/kernel-runtime.md)
