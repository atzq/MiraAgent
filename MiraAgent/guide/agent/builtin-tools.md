> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 内置工具一览

内核自带工具按用途分四组。Bash 的安全限制单独说明。

## 场景

已支持让 AI 调用工具时，可在此查看现成工具的清单、调用方式与副作用（改动所处理文件的工具）。

## 文件系统类

这一组操作的是真实目录（`holder.workspacePath` 指向的地方）。

| 工具名             | 干什么                                     | 关键参数                                                                                                                                                                                                       | 读/写 |
| --------------- | --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --- |
| `Bash`          | 执行一条 Shell 命令，返回 stdout 与 stderr 合并后的输出 | `command`(必填)、`description`(用途说明)、`timeout`(秒，默认 300)、`workdir`                                                                                                                                            | 写   |
| `Edit`          | 用一段新文本替换文件里的一段旧文本                       | `filePath`、`oldString`、`newString`、`replaceAll`(默认 false)                                                                                                                                                  | 写   |
| `Read`          | 读文件内容，可按行范围读                            | `filePath`、`offset`(起始行，默认 1)、`limit`(行数，默认 2000)                                                                                                                                                          | 读   |
| `Write`         | 新建文件或整篇覆盖写入                             | `filePath`、`content`                                                                                                                                                                                       | 写   |
| `Glob`          | 按通配符找文件，例如 `**/*.java`                  | `pattern`(必填，相对路径模式)、`path`(可选起始目录)                                                                                                                                                                        | 读   |
| `Grep`          | 按正则搜索文件内容                               | `pattern`(必填)、`path`、`glob`、`outputMode`(`content`/`files_with_matches`/`count`)、`contextBefore`/`contextAfter`/`contextAround`、`showLineNumber`、`caseInsensitive`、`type`、`headLimit`、`offset`、`multiline` | 读   |
| `list_files`    | 列出目录树结构                                 | `path`、`maxDepth`(默认 3)、`maxResults`(默认 200)、`offset`                                                                                                                                                      | 读   |
| `code_explorer` | 起一个子代理在目录里大范围搜索，只把结论带回来                 | `query`(必填，任务描述)                                                                                                                                                                                           | 读   |

## 会话附件类（VFS）

这一组操作的是"这次会话自己的附件空间"（`holder.vfsId` 指向的虚拟文件系统），跟真实目录隔开。设计中用户上传的文件放这里，非必须。

| 工具名              | 干什么              | 关键参数                                                     | 读/写 |
| ---------------- | ---------------- | -------------------------------------------------------- | --- |
| `vfs_glob`       | 在会话附件里按通配符找文件    | `pattern`(必填)、`path`(VFS 路径，默认根目录 `/`)                   | 读   |
| `vfs_grep`       | 在会话附件里按正则搜内容     | `pattern`(必填)、`path`、`glob`、`outputMode`、上下文行等（同 `Grep`） | 读   |
| `vfs_file_read`  | 读会话附件里的文件        | `filePath`、`offset`、`limit`                              | 读   |
| `vfs_file_write` | 写会话附件里的文件        | `filePath`、`content`                                     | 写   |
| `vfs_file_edit`  | 编辑会话附件里的文件（替换片段） | `filePath`、`oldString`、`newString`、`replaceAll`          | 写   |
| `vfs_list_files` | 列出会话附件的目录结构      | `path`、`maxDepth`、`maxResults`、`offset`                  | 读   |
| `vfs_explorer`   | 起子代理探索会话附件，只带回结论 | `query`(必填)                                              | 读   |

## 技能类

技能就是技能目录下的一份份 `*.zip` 包，**文件名就是它的名字（slug）**，包内放一本 `SKILL.md`。这一组让 AI 按需加载。

| 工具名               | 干什么                                                      | 关键参数                             | 读/写 |
| ----------------- | -------------------------------------------------------- | -------------------------------- | --- |
| `use_skill`       | 加载某个技能的说明书正文                                             | `skill_name`(技能 slug)            | 读   |
| `read_skill_file` | 读技能包里的某个具体文件                                             | `skill_name`、`file_path`(包内相对路径) | 读   |
| `expand_skill`    | 把技能包完整解压到 `<dataDir>/skills-expand/<slug>/`，方便跑脚本或看二进制资源 | `skill_name`                     | 写   |

技能工具只在"本次会话有激活技能"时才会被挂上白名单。

## MCP

| 工具名        | 干什么              | 关键参数                                                            | 读/写       |
| ---------- | ---------------- | --------------------------------------------------------------- | --------- |
| `mcp_call` | 调用外部 MCP 服务暴露的工具 | `tool_name`(必填，格式 `mcp__{服务名}__{工具名}`)、`arguments`(可选，JSON 字符串) | 取决于外部那个工具 |

`mcp_call` 只在本次会话配置了 MCP 服务、且拿到了工具清单时才会被挂上。

## Bash 的安全限制（以源码为准）

Bash 是唯一能跑任意命令的工具，因此它身上的约束最多：

| 限制        | 具体行为                                                                                                                                                                                                                                                                                                                                                               |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 危险命令检测    | 命中 `rm -rf` / `rm -r`、`git reset --hard`、`git push --force`、`git push -f`、`git clean -f`、`git branch -D`、`git checkout --`、`git commit --amend`、`git --no-verify`、`DROP TABLE/DATABASE/SCHEMA`、`TRUNCATE`、`DELETE FROM`、Windows 的 `del /S /Q`、`format C:` 时，**记日志并把警告写进结果**，但不拒绝执行。真正拦截需在 hook 里自行处理（见[工具执行前钩子（EventHook）](/MiraAgent/guide/agent/event-hook.md)）。 |
| 阻塞式 sleep | 检测到起始处或紧跟分隔符的 `sleep N`（N ≥ 2 秒）时**直接阻止**，返回错误、不执行。                                                                                                                                                                                                                                                                                                                |
| 超时        | 默认 300 秒，最大 1800 秒（传更大的值会被截到 1800）。到点强制终止，返回已产生的部分输出。                                                                                                                                                                                                                                                                                                              |
| 输出上限      | 回给模型只保留**末尾 2000 行 / 64 KB**；被截断时把完整输出写到临时文件，并把该路径回给模型（可用 `Read` 读全文）。                                                                                                                                                                                                                                                                                             |
| 输出总量看门狗   | 单条命令输出超过 64 MB 会被强制终止。                                                                                                                                                                                                                                                                                                                                             |
| 降权执行      | 配置了沙箱用户（`AgentConfig.bashSandBoxUser`）时，命令以该用户身份执行；没配就在当前用户下跑。                                                                                                                                                                                                                                                                                                     |
| 优先用专用工具   | 命令以 `grep`/`rg`/`find`/`ls`/`cat`/`head`/`tail`/`sed`/`awk`/`echo` 开头时记一条警告，提示改用 `Glob`/`Grep`/`Read`/`Edit`/`Write`。                                                                                                                                                                                                                                              |
| 工作目录      | 默认用 `workspacePath`；可用 `workdir` 参数指定（相对路径按工作空间解析）。每条命令新开 shell，不保留 `cd`、变量、函数。                                                                                                                                                                                                                                                                                    |
| 交互输入      | stdin 立即关闭，需要交互输入的命令会立刻拿到 EOF 结束，而不是挂到超时。                                                                                                                                                                                                                                                                                                                          |
| 无工作空间     | `workspacePath` 为空时直接禁用 Bash。                                                                                                                                                                                                                                                                                                                                      |

命令输出只保留尾部：构建日志与报错信息通常在末尾，丢弃头部比丢弃尾部影响小。

## 换个场景怎么改

| 目标               | 改哪儿                                                                                               |
| ---------------- | ------------------------------------------------------------------------------------------------- |
| 只用其中几个工具         | 不挂整套，白名单点名：`agent.mountToolsIfAbsent("Read", "Glob", "Grep");`                                    |
| 只给会话附件（VFS）的工具   | 用 `AutoAgent.getDefaultVfsReadTools()` 拿只读的那 5 个，写类自己按需加 `vfs_file_write`/`vfs_file_edit`         |
| 判断当前 Agent 是不是只读 | `boolean readOnly = AutoAgent.isReadOnly(holder.getTools());`                                     |
| 拿到全量清单           | `AutoAgent.fullCodeTools()`（含写类）、`AutoAgent.readOnlyTools()`（不含写类）、`AutoAgent.writeTools()`（只看写类） |

## 注意事项

- 工具名大小写不敏感地参与白名单比对；按上文写法书写可避免大小写误配。
- `Bash` 的危险命令警告不是拦截。拦截需在 hook（`onBeforeToolCall`）返回 `HookResult.respond(...)` 或 `HookResult.suspend()`。
- `Edit` 的 `oldString` 在文件中必须唯一，否则调用失败；该约束用于避免改错位置。

> 相关：[工具调用](/MiraAgent/guide/agent/tool-calling.md)、[自定义工具](/MiraAgent/guide/agent/custom-tools.md)、[技能包（Skills）](/MiraAgent/guide/extensions/skills.md)
