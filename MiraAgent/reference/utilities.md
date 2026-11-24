> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 内核工具类

内核自带的一批小工具类，按用途分组说明。

## 提示词模板（PromptLoader + PropsMap）

系统提示词包含动态片段（如"本次可用的技能清单"）时，采用模板加变量的写法。

| 方法                                               | 用途                           |
| ------------------------------------------------ | ---------------------------- |
| `PromptLoader.readFileString(path)`              | 读取模板文本（先找文件，再找 classpath 资源） |
| `PromptLoader.createTemplate(content)`           | 把文本变成模板                      |
| `PromptLoader.processTemplate(template, params)` | 用变量渲染模板                      |
| `PropsMap.put(key, value)`                       | 放入模板变量                       |

```java
import net.itzq.mira.core.utils.PromptLoader;
import net.itzq.mira.core.utils.PropsMap;

String templateText = PromptLoader.readFileString("prompt/agent_system.md");
PropsMap params = new PropsMap();
params.put("skills", "- 周报生成：根据本周记录生成周报");
params.put("skillCount", "1");

String prompt = PromptLoader.processTemplate(
        PromptLoader.createTemplate(templateText), params);
holder.setPrompt(prompt);   // 写回上下文
```

> 变量为 null 时，`PromptLoader.prompt(path, params)` 会把它替换成空串，避免模板渲染报错。

> 相关：[上层应用接入清单](/MiraAgent/reference/checklist.md)、[配置中心](/MiraAgent/guide/config/config-center.md)
