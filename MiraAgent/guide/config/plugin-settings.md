> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 插件配置项

编排任务里每个节点插件都有一组运行开关，例如超时时长、是否开启思考、使用哪套话术。这些值不写死在插件代码里，而通过配置快照的 `plugins` 块承载。配置快照的 `plugins` 块承载每个插件的一组设置；本节说明其结构及插件端的读取接口。

## 场景

编排任务通常包含多个节点插件，分别承担取数、AI 处理、判断等职责。每个插件各自需要一组开关，并在不同部署环境中取不同值。这些设置随整套配置一起导出与导入，插件在运行时直接读取，无需在构造阶段逐个手写。

## `plugins` 块的结构

`plugins` 是一个映射：**插件 id → 这个插件的一组设置**。

```json
"plugins": {
  "ai-chat-node": {
    "version": "1.0.0",
    "settings": {
      "llmTimeoutSeconds": 60,
      "enableThinking": true,
      "systemHint": "回答要简短"
    }
  },
  "data-fetch-node": {
    "version": "1.0.0",
    "settings": {
      "endpoint": "https://internal.example.com/api",
      "maxRows": 200
    }
  }
}
```

外层的 `ai-chat-node` 是插件 id；里面 `PluginSettings` 有两个字段：

| 字段         | 含义                            |
| ---------- | ----------------------------- |
| `version`  | 插件版本，导入时用来判断兼容性               |
| `settings` | 设置键值，一个 `Map<String, Object>` |

每个设置项的形态由插件用 `PluginSettingDef` 描述：`key`、`type`（`string` / `number` / `boolean` / `secret` / `enum`）、`label`、`description`、`defaultValue`、`options`（enum 用）、`required`、`placeholder`。其中 `type=secret` 的项会被当成凭据，导出时按凭据策略处理。这份定义由插件提供，界面据此渲染出表单；**值**则统一落在上面这个 `plugins` 块里。

## 读写设置的几个方法

都挂在配置快照 `rt.declaration()` 上：

| 方法                                       | 作用                                     |
| ---------------------------------------- | -------------------------------------- |
| `setPluginSetting(pluginId, key, value)` | 写一个设置值；插件或设置容器不存在时会自动创建                |
| `setPluginSettings(pluginId, map)`       | 批量写                                    |
| `pluginSettingsOf(pluginId)`             | 取整个设置容器（`PluginSettings`），不存在返回 `null` |
| `pluginSetting(pluginId, key)`           | 读字符串值；未配置或键不存在返回 `null`                |
| `pluginSetting(pluginId, key, def)`      | 读字符串值，空则回落 `def`                       |
| `pluginSettingLong(pluginId, key, def)`  | 按整数读，缺失或解析失败回落 `def`                   |
| `pluginSettingBool(pluginId, key, def)`  | 按布尔读，缺失回落 `def`                        |

以下为取值细节：

- **`pluginSetting` 一律返回字符串形态**（内部用 `String.valueOf`），所以数字/布尔存进去也读得出来。
- **`pluginSettingLong`**：值不是合法整数时，打一条日志并返回默认值，不抛异常。
- **`pluginSettingBool`**：`"1"` 或能解析为 true 的字符串算真，其他字符串算假；缺失才回落默认。
- **写入 `null`**：`setPluginSetting(pluginId, key, null)` 等于**删除**这个键，不是写空值。

## 最小可用代码

```java
import net.itzq.mira.modules.runtime.KernelRuntime;
import net.itzq.mira.modules.config.GlobalConfigManager;
import net.itzq.mira.modules.config.PluginSettings;

import java.util.HashMap;
import java.util.Map;

public class PluginSettingsDemo {

    public static void main(String[] args) {
        KernelRuntime rt = KernelRuntime.builder()
                .name("plugin-demo")
                .dataDir("D:/my-app/data")
                .build();
        KernelRuntime.installDefault(rt);
        rt.start();

        GlobalConfigManager decl = rt.declaration();

        // 1) 写一组插件设置（比如界面上的"保存"按钮触发）
        Map<String, Object> settings = new HashMap<>();
        settings.put("llmTimeoutSeconds", 60);
        settings.put("enableThinking", true);
        settings.put("systemHint", "回答要简短");
        decl.setPluginSettings("ai-chat-node", settings);

        // 2) 插件执行时读出来用
        String hint = decl.pluginSetting("ai-chat-node", "systemHint", "你是一个助手");
        long timeout = decl.pluginSettingLong("ai-chat-node", "llmTimeoutSeconds", 30L);
        boolean thinking = decl.pluginSettingBool("ai-chat-node", "enableThinking", false);

        System.out.println("提示词   = " + hint);
        System.out.println("超时(秒) = " + timeout);
        System.out.println("思考模式 = " + thinking);

        // 3) 拿整个容器看看
        PluginSettings ps = decl.pluginSettingsOf("ai-chat-node");
        System.out.println("插件版本 = " + ps.getVersion());
        System.out.println("全部设置 = " + ps.getSettings());

        // 4) 读取一个没配过的键，走默认值
        System.out.println("兜底值   = " + decl.pluginSetting("ai-chat-node", "notSet", "默认"));

        rt.close();
    }
}
```

## 跑起来会看到什么

```text
提示词   = 回答要简短
超时(秒) = 60
思考模式 = true
插件版本 = null
全部设置 = {enableThinking=true, systemHint=回答要简短, llmTimeoutSeconds=60}
兜底值   = 默认
```

`插件版本` 为 null，因为直接写设置值，没有设置 `version`。带上版本可手动构造一个 `PluginSettings` 对象：

```java
PluginSettings ps = new PluginSettings();
ps.setVersion("1.0.0");
ps.getSettings().put("systemHint", "回答要简短");
rt.declaration().getPlugins().put("ai-chat-node", ps);
```

## 换个场景怎么改

**让运维在界面上改这些值**

流程是：界面读 `pluginSettingsOf(pluginId)` 回显 → 用户改 → 保存时调 `setPluginSetting(...)` 或 `setPluginSettings(...)` 写回配置快照。

写完之后，把"保存"接到应用的统一"应用配置"出口上即可：

```java
// 设置面板保存后
decl.setPluginSettings("ai-chat-node", settings);
// 走统一的应用出口（复用配置中心的"应用"动作）
decl.applyAllInstance();
```

插件设置在插件执行时直接读配置快照，值写入后下一次执行即生效；`applyAllInstance()` 负责把模型与向量模型重新注册进运行时。两者一同调用无副作用，保存后统一调用一次即可。

**其他常见改法**

| 目标       | 改哪儿                                          |
| -------- | -------------------------------------------- |
| 多个插件各自一组 | 换 `pluginId` 再写一遍，互不影响                       |
| 一个插件改一项  | `setPluginSetting(pluginId, key, value)`     |
| 删掉一项     | `setPluginSetting(pluginId, key, null)`      |
| 敏感值（密钥类） | 在插件的设置定义里把该项 `type` 设为 `secret`，导出时自动按凭据策略处理 |

## 注意事项

- **插件 id 保持一致**：`plugins` 按 id 归类，导出与导入均依赖它对齐。
- **读之前判空**：`pluginSettingsOf(pluginId)` 在插件没配过时返回 `null`；`pluginSetting(...)` 系列则安全地返回默认值。
- **类型转换由调用方负责**：设置值底层为 `Object`，`pluginSettingLong` / `pluginSettingBool` 代为转换，转换失败仅回落默认值、不报错，因此无法用于发现配置写错。

> 相关：[节点配置：这个节点需要什么](/MiraAgent/reference/orchestration/node-blueprint.md)、[配置中心](/MiraAgent/guide/config/config-center.md)
