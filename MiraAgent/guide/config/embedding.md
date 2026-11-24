> For AI agents: the complete documentation index is available at /MiraAgent/llms.txt, the full documentation bundle is available at /MiraAgent/llms-full.txt.

# 接入向量模型

做语义检索、知识库召回，第一步是把文本变成一串数字（向量），再用这串数字算相似度。内核提供向量化能力与一张注册表：接入向量模型后，可按别名取得任意文本的向量。本节说明接入、查询的方式，以及自行计算相似度的方法。

## 场景

以知识库场景为例：用户提问后，先从库中取出若干最相关片段，再交由大模型回答。实现"语义相近"而非"字面相同"的匹配，需要先把文本转为向量。向量模型与对话模型一致，支持配置多个并随时切换。

## 一条向量模型记录有哪些字段

`EmbeddingApiConfig` 的一行就是一个可用的向量模型：

| 字段               | 含义                                                   | 默认    |
| ---------------- | ---------------------------------------------------- | ----- |
| `alias`          | 别名，注册与查找都靠它，必须非空                                     | 无     |
| `apiUrl`         | **完整的接口地址**，如 `https://api.openai.com/v1/embeddings` | 无     |
| `apiKey`         | 密钥                                                   | 无     |
| `model`          | 模型名，如 `text-embedding-3-small`                       | 无     |
| `dimension`      | 向量维度                                                 | 1536  |
| `connectTimeout` | 连接超时（毫秒）                                             | 30000 |
| `readTimeout`    | 读取超时（毫秒）                                             | 60000 |

`apiUrl` 与对话模型的 `apiHost` + `apiEndpoint` 不同：向量模型此处使用**一个完整地址**。

## 注册与查找

注册表是 `rt.embeddingRegistry()`，常用方法：

| 方法                                  | 作用                                                |
| ----------------------------------- | ------------------------------------------------- |
| `registerProvider(config)`          | 按一条 `EmbeddingApiConfig` 注册（内部创建一个 HTTP 提供者）      |
| `registerProvider(alias, provider)` | 注册实现的提供者实例                                        |
| `getProvider(alias)`                | 按别名取提供者，没注册会抛异常；`getProvider(alias, false)` 只警告不抛 |
| `embed(alias, text)`                | 按别名把文本转成向量                                        |
| `embed(text)`                       | 用默认提供者转向量                                         |
| `setDefaultProvider(alias)`         | 设置默认提供者                                           |
| `unregisterProvider(alias)`         | 注销                                                |
| `aliases()`                         | 列出已注册的别名                                          |

通过配置快照注册：`declaration().setEmbeddings(list)` 配合 `declaration().applyAllInstance()`。

## HttpEmbeddingProvider 的能力

注册表按配置创建出来的 `HttpEmbeddingProvider` 会：

- 向 `apiUrl` 发一个 POST，请求体是 `{"input": 文本, "model": 模型名, "dimensions": 维度}`，带 `Authorization: Bearer <apiKey>` 头。**这是 OpenAI 兼容格式**。
- 文本超过约 32000 字符时自动截断并打日志。
- 调用失败（网络错误、非 200、返回空）时**返回 null**，不抛异常。
- 提供 `getDimension()` 返回配置的维度。

## 最小可用代码

示例注册一个向量模型，把三句话转成向量，再计算余弦相似度：

```java
import net.itzq.mira.modules.runtime.KernelRuntime;
import net.itzq.mira.modules.config.AgentConfig;
import net.itzq.mira.modules.config.GlobalConfigManager;
import net.itzq.mira.modules.ai.client.embedding.EmbeddingApiConfig;
import net.itzq.mira.modules.ai.client.embedding.EmbeddingRegistry;

import java.util.ArrayList;
import java.util.List;

public class EmbeddingDemo {

    public static void main(String[] args) {
        KernelRuntime rt = KernelRuntime.builder()
                .name("embed-demo")
                .dataDir("D:/my-app/data")
                .build();
        KernelRuntime.installDefault(rt);
        rt.start();

        EmbeddingApiConfig emb = EmbeddingApiConfig.builder()
                .alias("bge-m3")
                .apiUrl("https://api.example.com/v1/embeddings")
                .apiKey(System.getenv("EMBEDDING_API_KEY"))
                .model("bge-m3")
                .dimension(1024)
                .build();

        GlobalConfigManager decl = rt.declaration();
        decl.setEmbeddings(new ArrayList<>(java.util.Arrays.asList(emb)));

        AgentConfig agentConfig = new AgentConfig();
        agentConfig.setDefaultEmbedding("bge-m3");
        decl.setAgentConfig(agentConfig);

        decl.applyAllInstance();   // 注册进本运行时

        EmbeddingRegistry registry = rt.embeddingRegistry();
        System.out.println("已注册向量模型 = " + registry.aliases());

        float[] v1 = registry.embed("bge-m3", "如何申请报销");
        float[] v2 = registry.embed("bge-m3", "报销流程怎么走");
        float[] v3 = registry.embed("bge-m3", "今天天气不错");

        if (v1 == null || v2 == null || v3 == null) {
            System.out.println("向量调用失败，检查 apiUrl / apiKey");
            rt.close();
            return;
        }
        System.out.println("报销 vs 报销流程 相似度 = " + cosine(v1, v2));
        System.out.println("报销 vs 天气     相似度 = " + cosine(v1, v3));

        rt.close();
    }

    // 余弦相似度：值越接近 1 越相似
    private static double cosine(float[] a, float[] b) {
        double dot = 0.0, na = 0.0, nb = 0.0;
        for (int i = 0; i < a.length; i++) {
            dot += a[i] * b[i];
            na += a[i] * a[i];
            nb += b[i] * b[i];
        }
        if (na == 0.0 || nb == 0.0) {
            return 0.0;
        }
        return dot / (Math.sqrt(na) * Math.sqrt(nb));
    }
}
```

## 跑起来会看到什么

- 应用配置时日志出现 `注册 Embedding: bge-m3`。
- 控制台：

```text
已注册向量模型 = [bge-m3]
报销 vs 报销流程 相似度 = 0.9137
报销 vs 天气     相似度 = 0.2041
```

语义相近的两句话相似度明显更高，这就是知识库检索的基本原理。

## 关于内核能力的说明

内核提供的是**向量化能力与注册表**：把一个别名映射到一个"文本 → 向量"的提供者，外加一个 `EmbeddingProvider` 接口（两个方法：`embed(text)` 和 `getDimension()`）。

**检索业务需自行实现。** 内核源码里没有"存向量、查最近邻、拼上下文"的现成逻辑——那部分属于应用：怎么存（内存、文件、向量库）、怎么查（暴力算余弦还是一棵索引）、查几条、怎么拼进提示词，均由调用方决定。上文的余弦相似度示例即最简形态的检索逻辑。

## 换个场景怎么改

**换成自建 embedding 服务**

若服务返回的格式非 OpenAI 兼容，不修改 `HttpEmbeddingProvider`，改为实现 `EmbeddingProvider` 后注册：

```java
import net.itzq.mira.modules.ai.client.embedding.EmbeddingProvider;

class MyEmbeddingProvider implements EmbeddingProvider {
    @Override
    public float[] embed(String text) {
        // 调用实际部署的服务，返回 float[]
        return new float[]{ /* ... */ };
    }

    @Override
    public int getDimension() {
        return 768;
    }
}

// 注册自定义实例，别名照旧
rt.embeddingRegistry().registerProvider("my-embed", new MyEmbeddingProvider());
// 之后和内置的一样用：rt.embeddingRegistry().embed("my-embed", "文本");
```

**只改接口地址**：如果服务仍是 OpenAI 兼容，只改 `apiUrl` / `model` / `dimension` 即可。

| 待替换项      | 改哪儿                                                                                                  |
| --------- | ---------------------------------------------------------------------------------------------------- |
| 换服务商（仍兼容） | `EmbeddingApiConfig` 的 `apiUrl`、`model`、`dimension`                                                  |
| 换默认向量模型   | `agentConfig.setDefaultEmbedding("别名")` + `applyAllInstance()`，或 `registry.setDefaultProvider("别名")` |
| 完全自定义协议   | 实现 `EmbeddingProvider`，用 `registerProvider(alias, provider)` 注册                                      |

## 注意事项

- **`embed` 失败返回 null**：调用方需先判空再使用；直接访问 `vec.length` 会触发空指针。
- **`dimension` 需准确**：它随请求发给服务端（`dimensions` 字段），同时也是 `getDimension()` 的返回值；填错会导致向量维度与预期不符。
- **`apiUrl` 为完整地址**；只填主机名会导致请求路径错误。
- **切换向量模型需重算**：不同模型的向量不在同一空间，更换模型要求对整库重新向量化。

> 相关：[接入一个模型](/MiraAgent/guide/config/models.md)、[会话附件（VFS）](/MiraAgent/guide/data/vfs.md)、[技能包（Skills）](/MiraAgent/guide/extensions/skills.md)
