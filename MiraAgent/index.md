> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# Mira Agent Core

轻量智能体内核

> 在 Java 应用内搭建具备对话、工具调用、会话记忆与流程编排能力的智能体

[快速开始](/guide/start/getting-started) | [Github](https://github.com/atzq/MiraAgent)

## Features

- [**依赖最小化**](/guide/index): 不依赖 Spring AI / LangChain4j 或任何 Web 框架，是可独立引入的运行时引擎。界面、业务与部署由使用方决定。
- [**对话与流式输出**](/guide/agent/basic-chat): 支持一问一答、多轮与逐字流式；运行过程以事件形式输出，由应用决定如何呈现。
- [**工具调用循环**](/guide/agent/tool-calling): 模型触发工具调用时，参数解析、执行、结果回填与下一轮循环由内核完成。
- [**会话记忆与回放**](/guide/data/persistence): 预留持久化扩展点，可接入 MySQL、Redis 或本地文件；消息与过程事件可按时间线回放。
- [**能力可插拔**](/guide/extensions/skills): 支持按需加载技能包、接入外部 MCP 服务、将业务 HTTP 接口注册为工具。
- [**运行预校验**](/reference/orchestration/execution-resolver): 节点所需的模型、工具与必需参数是否齐备，在执行前即可得出结论。
- [**一个进程多套运行时**](/guide/runtime/multi-instance): 支持在同一进程内运行多套互不干扰的运行时，各自持有独立的模型、工具、配置与数据目录。
- [**会话附件与工作空间**](/guide/data/vfs): 上传文件被转成纯文本存入会话附件包；也可组合特定工具，让 AI 读写指定项目目录。
- [**跨进程执行**](/guide/config/export-import): 运行时可导出 JSON 快照并在别处复原，支持沙箱与远程分布式执行；编排工作流可仅依赖运行时无头运行。
