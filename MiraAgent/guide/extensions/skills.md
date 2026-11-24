> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 技能包（Skills）

一份"操作手册"可打包为技能，AI 在需要时自行读取，避免将手册全文放入系统提示词。

## 场景

示例：客服助手处理退款、改地址、开发票等业务，每个业务对应一份内部手册，合计数万字。全部放入系统提示词会带来长度、成本与混淆问题。

需求是：系统提示词中只声明存在哪些手册，由 AI 自行决定并读取所需手册。

内核中负责该能力的是 **技能**。一个技能即 **一个 zip 包**，内含 `SKILL.md`（开头为元信息，其后为正文）及所需参考资料。AI 先看到技能摘要，决定使用时调用 `use_skill` 读取正文。

## 最小可用代码

技能包以目录下的 `*.zip` 形式存在，每个 zip 对应一个技能，**文件名即其名字（slug）**：

- 文件名：`order-manual.zip`
- 包里：`SKILL.md`（可以直接放在 zip 根目录，也可以放在唯一的一层顶层目录里，两种打包方式都认）
- `SKILL.md` 开头是两个 `---` 包着的元信息，字段有：`name`（就是 slug，要和 zip 文件名一致）、`description`（多行描述用 `description: >`）、`description_zh`、`description_en`、`version`、`license`、`category`、`triggers`（触发词列表，每行 `- 词`）。两个 `---` 之后就是正文，也就是 AI 真正要读的内容。

将技能挂接到一次会话的完整代码如下：

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.core.utils.PropsMap;
import net.itzq.mira.core.utils.PromptLoader;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;
import net.itzq.mira.modules.ai.mcp.McpPrepared;
import net.itzq.mira.modules.ai.skills.SkillEntity;
import net.itzq.mira.modules.runtime.KernelRuntime;

import java.util.Arrays;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Set;

public class SkillsDemo {
    public static void main(String[] args) throws Exception {
        KernelRuntime rt = KernelRuntime.builder()
                .name("my-app")
                .dataDir("D:/my-app/data")
                .build()
                .start();

        // 1. 装载技能：扫描这个目录下的 *.zip（幂等，装过一次再调会跳过）
        rt.skills().init("D:/my-app/skills");

        AgentContextHolder holder = AgentContextHolder.builder()
                .userId("u1")
                .historyId(IdGen.uuid())
                .modelAlias("deepseek-v3")
                .runtime(rt)
                .build();

        // 2. 本次会话要用哪几本手册，按 slug 点名
        holder.setActiveSkillSlugs(Arrays.asList("order-manual"));

        // 3. 白名单 = 系统基线工具 + 技能配套工具（use_skill / read_skill_file / expand_skill）
        Set<String> whitelist = new LinkedHashSet<>(AutoAgent.systemBaselineTools(null));
        List<SkillEntity> activeSkills = AutoAgent.resolveActiveSkills(holder);
        McpPrepared noMcp = McpPrepared.empty();
        AutoAgent.mountSkillAndMcpTools(whitelist, activeSkills, noMcp);

        AutoAgent agent = new AutoAgent(holder);
        agent.mountToolsIfAbsent(whitelist.toArray(new String[0]));

        // 4. 把技能摘要渲染进系统提示词，AI 才知道"有这么几本手册可翻"
        PropsMap params = AutoAgent.buildSkillMcpPromptParams(activeSkills, noMcp);
        String prompt = PromptLoader.processTemplate(
                PromptLoader.createTemplate(
                        "你是一个严谨的助手。\n\n可用技能:\n${skills}\n\n可用 MCP 工具:\n${mcpTools}"),
                params);
        holder.setPrompt(prompt);

        // 5. 开聊
        agent.chatStream("帮我按手册处理一笔订单退款").await();
    }
}
```

## 跑起来会看到什么

- 启动时会打印技能装载情况，比如 `已加载技能: order-manual (订单退款手册)`、`SkillRepository 初始化完成，共注册 1 个技能`。
- 系统提示词里 `${skills}` 被替换成了技能的摘要行（slug + 描述），`${mcpTools}` 因为是空配置，被替换成空字符串。
- 对话时，AI 先调用 `use_skill`（参数 `skill_name`），随后获取 `<skill_content>...</skill_content>` 包裹的正文，并据此回答。
- 手册里如果还提到别的文件，AI 可以调 `read_skill_file`（参数 `skill_name` + `file_path`）读包里的单个文件；如果那个技能自带脚本、需要落到磁盘才能用，AI 会调 `expand_skill`，包会被解压到 `<dataDir>/skills-expand/<skill_name>/`，返回真实目录路径。
- `use_skill` 有校验：`skill_name` 必须在本次会话的 `activeSkillSlugs` 里，否则会被拒绝。

## 换个场景怎么改

| 目标               | 改哪里                                                                                                                               |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| 由 AI 按用户输入自动挑选手册 | 调用 `rt.skills().matchSkills(用户输入)` 获取触发词命中的技能列表，将 slug 传入 `holder.setActiveSkillSlugs(...)`                                       |
| 一次给多本手册          | `setActiveSkillSlugs(Arrays.asList("order-manual", "invoice-manual"))`，多本会一起进摘要                                                   |
| 手册里附带模板、脚本等资料    | 打包进 zip 的其它文件，AI 用 `read_skill_file` 读、用 `expand_skill` 解压                                                                        |
| 换技能存放目录          | 换成实际路径调 `rt.skills().init("...")`；若想让 `resolveActiveSkills` 自动加载，把配置快照里的技能目录（`AgentConfig.skillsDir`，默认 `data-skills/skills`）改成目录 |
| 一个进程里跑多套互不干扰的运行时 | 每套运行时各自 `rt.skills().init(自己目录)`，同名技能在两个目录里也不会串                                                                                   |
| 改了技能包想重新加载       | 先 `rt.skills().shutdown()` 释放句柄，再 `init(...)`；只改 zip 内容不重载是不生效的                                                                   |

## 使用要点

- **只认 zip**：技能目录里扫描的是 `*.zip`，散着的文件夹不会被识别。
- **slug 需一致**：`SKILL.md` 中的 `name` 与 zip 文件名须一致，工具按该 slug 查找包。
- **init 幂等**：已装载过的技能再次调用会被跳过。更换目录或重载需先调用 `shutdown()`。
- **未激活不可用**：仅进入 `activeSkillSlugs` 的技能可被 AI 通过 `use_skill` 调用；配套的 `use_skill` / `read_skill_file` / `expand_skill` 三个工具，仅在本次会话存在激活技能时自动加入白名单。
- **会话结束应释放**：`rt.skills().shutdown()` 清空注册表并关闭已打开的 zip 句柄。

> 相关：[接入 MCP 工具服务](/MiraAgent/guide/extensions/mcp.md) · [内置工具一览](/MiraAgent/guide/agent/builtin-tools.md) · [配置中心](/MiraAgent/guide/config/config-center.md)
