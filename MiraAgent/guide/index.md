> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 总览

Mira Agent Core 是一个纯 Java、最小化依赖的循环模式智能体内核，可嵌入现有 Java 应用，负责大模型对话、工具调用、会话文件读写与历史消息管理，不依赖 Spring 或其他框架。

## 环境依赖

运行环境：JDK8

内核的运行时依赖如下：

| 依赖                                                      | 版本     | 用途       |
| ------------------------------------------------------- | ------ | -------- |
| freemarker                                              | 2.3.35 | 提示词模板渲染  |
| transmittable-thread-local                              | 2.14.5 | 跨线程上下文传递 |
| jackson-core / databind / annotations / datatype-jsr310 | 2.15.2 | JSON 序列化 |
| slf4j-api                                               | 1.7.32 | 日志       |
| hutool-http                                             | 5.8.46 | HTTP 客户端 |

Lombok 仅在编译期使用（`provided`）；JUnit 4 只用于测试（`test`）。

## 核心构成

内核里只有四类资产，理解它们即理解全部：

```text
配置快照      —— 这个系统里"有什么"：有哪些模型、哪些工具、系统提示词是什么
调用参数      —— 这一次"怎么跑"：问什么、用哪个模型、挂哪个会话
事件          —— 运行过程中的直播：思考、调工具、正文、结束、报错
扩展接口      —— 由调用方实现的几个接口：存历史、解析工具包
```

内核负责把这些串起来跑；应用负责给用户界面、做业务判断、决定配置内容。

## 功能索引

| 目标                     | 看这一篇                                                               |
| ---------------------- | ------------------------------------------------------------------ |
| 了解适用条件与边界              | [内核适用场景](/MiraAgent/guide/start/introduction.md)                   |
| 搞清各部分谁调用谁              | [整体结构](/MiraAgent/guide/start/architecture.md)                     |
| 跑通第一次对话                | [跑通第一次对话](/MiraAgent/guide/start/getting-started.md)               |
| 应用启动时先把内核准备好           | [内核运行时装配](/MiraAgent/guide/runtime/kernel-runtime.md)              |
| 一个进程里跑多套互不干扰的运行时       | [一个进程里跑多套运行时](/MiraAgent/guide/runtime/multi-instance.md)          |
| 跟大模型一问一答、多轮追问          | [基础对话](/MiraAgent/guide/agent/basic-chat.md)                       |
| 逐字流式输出、推给前端            | [流式输出](/MiraAgent/guide/agent/streaming.md)                        |
| 让 AI 调用工具              | [工具调用](/MiraAgent/guide/agent/tool-calling.md)                     |
| 看内核自带哪些工具              | [内置工具一览](/MiraAgent/guide/agent/builtin-tools.md)                  |
| 写自己的工具                 | [自定义工具](/MiraAgent/guide/agent/custom-tools.md)                    |
| 拆出子智能体分工干活             | [子代理（SubAgent）](/MiraAgent/guide/agent/sub-agent.md)               |
| 并发跑任务、跨线程取 ThreadLocal | [并发执行与 ThreadLocal](/MiraAgent/guide/agent/concurrency.md)         |
| 让用户中途停止、之后还能继续         | [停止与续跑](/MiraAgent/guide/agent/stop-resume.md)                     |
| 在对话前后插入自己的逻辑           | [工具执行前钩子（EventHook）](/MiraAgent/guide/agent/event-hook.md)         |
| 上下文太长时自动压缩             | [长对话压缩（ContextCompactor）](/MiraAgent/guide/agent/compaction.md)    |
| 上传的文件给 AI 阅读           | [会话附件（VFS）](/MiraAgent/guide/data/vfs.md)                          |
| 让 AI 操作指定的项目目录         | [工作空间（Workspace）](/MiraAgent/guide/data/workspace.md)              |
| 会话有记忆、能回放              | [消息持久化](/MiraAgent/guide/data/persistence.md)                      |
| 了解一份配置里到底有什么           | [配置中心](/MiraAgent/guide/config/config-center.md)                   |
| 配置多个模型随时切换             | [接入一个模型](/MiraAgent/guide/config/models.md)                        |
| 用向量模型做检索               | [接入向量模型](/MiraAgent/guide/config/embedding.md)                     |
| 导出整套配置、换个地方导入运行        | [配置导出与导入](/MiraAgent/guide/config/export-import.md)                |
| 给插件型功能存自己的设置           | [插件配置项](/MiraAgent/guide/config/plugin-settings.md)                |
| 让 AI 按需加载"技能说明书"       | [技能包（Skills）](/MiraAgent/guide/extensions/skills.md)               |
| 接外部 MCP 工具服务           | [接入 MCP 工具服务](/MiraAgent/guide/extensions/mcp.md)                  |
| 把业务 HTTP 接口变成工具        | [HTTP 工具](/MiraAgent/guide/extensions/http-tools.md)               |
| 让 AI 看懂图片              | [图片输入](/MiraAgent/guide/extensions/multimodal.md)                  |
| 画一张流程图，让 AI 按流程自动跑     | [编排：让 AI 按流程跑](/MiraAgent/reference/orchestration/overview.md)     |
| 每个节点跑之前先校验配置           | [执行前先校验](/MiraAgent/reference/orchestration/execution-resolver.md) |
| 照着清单搭一个完整上层应用          | [上层应用接入清单](/MiraAgent/reference/checklist.md)                      |

## 推荐阅读顺序

1. [内核适用场景](/MiraAgent/guide/start/introduction.md) —— 先看边界与适用条件
2. [整体结构](/MiraAgent/guide/start/architecture.md) —— 各组件关系与调用方向
3. [跑通第一次对话](/MiraAgent/guide/start/getting-started.md) —— 复制代码即可运行
4. 之后按上方功能索引挑选所需篇章

> 相关：[上层应用接入清单](/MiraAgent/reference/checklist.md)、[内核适用场景](/MiraAgent/guide/start/introduction.md)、[整体结构](/MiraAgent/guide/start/architecture.md)
