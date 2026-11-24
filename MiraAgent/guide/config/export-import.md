> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 配置导出与导入

在开发机配好模型、工具、提示词后，可导出为一份 JSON 文件；在无界面服务器或另一环境中读回即可运行。本节说明导出时密钥的处理方式、导入时校验报告的读取方式，以及只导入部分配置的方法。

## 场景

开发机调试好整套配置后，需将应用部署到一台无界面服务器。配置以文件形式从开发机带到服务器，服务器读文件即可运行，无需在服务器上重新配置。密钥不应以明文形式存放在文件中。

## 两种密钥策略

导出用 `rt.exportDeclaration(CredentialPolicy)`，参数决定密钥怎么落盘：

| 策略                           | 效果                       | 用在哪         |
| ---------------------------- | ------------------------ | ----------- |
| `CredentialPolicy.INLINE`    | 密钥**原文**内联               | 本机自用、自己存数据库 |
| `CredentialPolicy.REFERENCE` | 密钥换成 `${ENV:<密钥末8位>}` 占位 | 分发给别人、进代码仓库 |

传入 `null` 等同于 `INLINE`。`REFERENCE` 只保留原值的**末 8 位**作为提示，不会泄露完整密钥。

导入端的处理：内核**不会自动把 `${ENV:...}` 解析成真实值**。当探测到凭据仍为未解析的引用形态时，在报告中记一条 warning，但不阻断执行。真实值需在导入前自行替换（见"换个场景怎么改"）。

## 导入与校验报告

`rt.importDeclaration(json)` 做两件事：

1. 把 JSON 里出现的配置块写进这个运行时的配置快照，并**立即应用进本运行时**（模型、向量模型会注册好，默认别名生效）。
2. 返回一个 `ValidationReport`。

报告字段含义：

| 方法 / 字段         | 含义                                                |
| --------------- | ------------------------------------------------- |
| `isPass()`      | 无致命、且无缺失项时为 true，可以继续跑                            |
| `getMissing()`  | 缺失清单，每项含 `getKind()` / `getRef()` / `getDetail()` |
| `getWarnings()` | 警告清单（字符串），不阻断                                     |
| `isFatal()`     | 是否致命                                              |
| `isEmpty()`     | 缺失和警告都为空                                          |
| `summary()`     | 人类可读的一行摘要，适合打日志                                   |

`getMissing()` 里 `kind` 的取值有：`model`、`tool`、`prompt`、`credential`、`invocation`、`config`。比如导出的 `toolProviders` 里有个 `builtin` 工具，但目标运行时没注册它，就会记一条 `kind=tool` 的缺失。

**只导入部分配置为内置能力**：导入逐块检查，JSON 中未出现的块保持原值不变。增删改均可：将无需变动的块从 JSON 中删除即可。

## 最小可用代码

```java
import net.itzq.mira.core.utils.IdGen;
import net.itzq.mira.modules.runtime.KernelRuntime;
import net.itzq.mira.modules.config.AgentConfig;
import net.itzq.mira.modules.config.CredentialPolicy;
import net.itzq.mira.modules.config.GlobalConfigManager;
import net.itzq.mira.modules.config.ValidationReport;
import net.itzq.mira.modules.ai.agent.AgentContextHolder;
import net.itzq.mira.modules.ai.agent.AutoAgent;
import net.itzq.mira.modules.ai.client.config.ModelApiConfig;

import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

public class ExportImportDemo {

    public static void main(String[] args) throws Exception {

        // ============ 第一步：开发机上配好并导出 ============
        KernelRuntime dev = KernelRuntime.builder()
                .name("dev")
                .dataDir("D:/dev/data")
                .build();
        KernelRuntime.installDefault(dev);
        dev.start();

        GlobalConfigManager decl = dev.declaration();
        ModelApiConfig model = ModelApiConfig.builder()
                .alias("deepseek-v3")
                .providerName("deepseek")
                .apiModelName("deepseek-chat")
                .apiHost("https://api.deepseek.com")
                .apiEndpoint("/v1/chat/completions")
                .apiKey("sk-真实密钥")
                .build();
        decl.setModels(new ArrayList<ModelApiConfig>(Arrays.asList(model)));
        AgentConfig agentConfig = new AgentConfig();
        agentConfig.setDefaultModel("deepseek-v3");
        decl.setAgentConfig(agentConfig);
        decl.applyAllInstance();

        // 导出成 REFERENCE：密钥只留末 8 位
        String json = dev.exportDeclaration(CredentialPolicy.REFERENCE);
        Files.write(Paths.get("D:/decl.json"), json.getBytes(StandardCharsets.UTF_8));
        System.out.println("已导出到 D:/decl.json");
        dev.close();

        // ============ 第二步：另一台机器读文件并导入 ============
        String loaded = new String(
                Files.readAllBytes(Paths.get("D:/decl.json")), StandardCharsets.UTF_8);

        KernelRuntime server = KernelRuntime.builder()
                .name("server")
                .dataDir("/data/app")
                .build();
        KernelRuntime.installDefault(server);
        server.start();

        ValidationReport report = server.importDeclaration(loaded);
        System.out.println(report.summary());

        if (!report.isPass()) {
            for (ValidationReport.Missing m : report.getMissing()) {
                System.out.println("缺少: kind=" + m.getKind()
                        + ", ref=" + m.getRef() + ", detail=" + m.getDetail());
            }
            server.close();
            return;
        }

        // 校验通过，直接开始对话
        AgentContextHolder holder = AgentContextHolder.builder()
                .userId("u1")
                .historyId(IdGen.uuid())
                .runtime(server)
                .modelAlias("deepseek-v3")
                .build();
        String answer = new AutoAgent(holder).chat("你好，做个自我介绍");
        System.out.println("回答 = " + answer);

        server.close();
    }
}
```

## 跑起来会看到什么

- 开发机侧打印 `已导出到 D:/decl.json`，文件里的 `apiKey` 是 `${ENV:真实密钥}` 形态（保留末 8 位）。
- 服务器侧打印一行摘要：

```text
[OK] missing=0, warnings=1
  ~ 模型[deepseek-v3] 凭据为引用形态且未解析: ${ENV:真实密钥}
```

- `isPass()` 为 true（warning 不阻断），对话照常执行。

如果目标环境缺了某个 builtin 工具，摘要会变成 `[INCOMPLETE] missing=1, ...`，并打印出缺的那一项，`isPass()` 为 false。

## 换个场景怎么改

**容器里用环境变量注入密钥**

`REFERENCE` 导出的占位不会自动解析，所以在导入前把 `${ENV:...}` 换成环境变量值即可：

```java
import java.util.regex.Matcher;
import java.util.regex.Pattern;

// 把 ${ENV:xxx} 换成同名环境变量的真实值
static String resolveEnv(String json) {
    Pattern p = Pattern.compile("\\$\\{ENV:([^}]+)\\}");
    Matcher m = p.matcher(json);
    StringBuffer sb = new StringBuffer();
    while (m.find()) {
        String name = m.group(1);
        String val = System.getenv(name);
        m.appendReplacement(sb, val == null ? "" : Matcher.quoteReplacement(val));
    }
    m.appendTail(sb);
    return sb.toString();
}

// 导入前先解析，warning 就不会出现
ValidationReport report = server.importDeclaration(resolveEnv(loaded));
```

容器里把真实密钥设成同名环境变量即可，比如 `Environment: ENV_NAME=<密钥末8位对应的环境变量名>`。

**只导入一部分配置**

| 想只改   | 做法                         |
| ----- | -------------------------- |
| 只换模型  | 保留 JSON 的 `models` 块，删掉其余块 |
| 只改提示词 | 只留 `prompts` 块             |
| 只改默认值 | 只留 `agentConfig` 块         |

导入时没出现的块不会被清空，保持运行时原有值。

**多套运行时并存**：导入只作用于调用 `importDeclaration` 的那一个运行时，同一个进程里的其他运行时互不影响。

## 注意事项

- **`INLINE` 不宜入库**：密钥原文明文写入 JSON，仅适合本机或私有加密存储。
- **`REFERENCE` 需自行解析**：若不解析，模型会把 `${ENV:...}` 当作密钥发出，导致调用失败。
- **导入即应用**：导入后立即将模型与向量模型注册进本运行时。若需"先校验、确认无误再应用"，应先对 JSON 做校验，再决定是否导入。
- **缺失不静默补默认**：报告只说明缺什么，不会替调用方猜测。

> 相关：[配置中心](/MiraAgent/guide/config/config-center.md)、[接入一个模型](/MiraAgent/guide/config/models.md)、[一个进程里跑多套运行时](/MiraAgent/guide/runtime/multi-instance.md)
