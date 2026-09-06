## 大模型训练流程

```
海量原始数据
    ↓
数据清洗 / 去重 / 过滤 / 配比
    ↓
Tokenizer
    ↓
模型架构设计
    ↓
大规模预训练（Pretraining）
    ↓
Base Model
    ↓
持续预训练 / 领域增强
    ↓
SFT / Instruction Tuning
    ↓
Preference / Reward Training
    ↓
Reasoning 强化学习
    ↓
Tool-use / Agent Training
    ↓
Safety / Alignment
    ↓
评测 / Red Team
    ↓
再次训练
    ↓
最终模型
```

## 大模型处理流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Client / Application                      │
│                                                                     │
│  你的代码：                                                           │
│  model + messages/input + tools + stream + ...                      │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               │ HTTP Request
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          API Gateway / Server                       │
│                                                                     │
│  认证 → 参数校验 → 限流 → 路由 → 请求预处理                              │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         LLM Serving Layer                           │
│                                                                     │
│  ① 上下文组装                                                        │
│  ② Tokenization                                                     │
│  ③ Token IDs                                                        │
│  ④ Embedding / 输入表示                                              │
│  ⑤ Transformer 前向计算                                              │
│  ⑥ Attention + FFN + 多层 Transformer                               │
│  ⑦ 输出 Logits                                                      │
│  ⑧ Sampling                                                         │
│  ⑨ 得到下一个 Token                                                  │
│  ⑩ 不断重复 ⑤~⑨                                                    │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Detokenization                               │
│                                                                     │
│  Token IDs → Token → 文本                                            │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Response Layer                              │
│                                                                     │
│  普通：完整 JSON                                                      │
│  流式：SSE / Streaming Events                                        │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
                            Client
```

**上下文组装**

进入 LLM serving layer 后，会把 API 参数转换成内部统一的数据结构。内部可能 `System / Developer Context + User Context + Tool Definitions + Conversation History + Generation Config`，最终形成 Prompt / Context ，也就是模型真正看到的上下文。

**Tokenization：文本变成 Token**

"什么是 Elasticsearch？"，文本不会直接原封不动交给 Transformer，会经过：`Tokenizer -> Tokens -> Token IDs`，概念上可能变成：`["什么", "是", " Elastic", "search", "？"]`, 再映射成 [18372, 421, 9321, 18273, 45]，这里的 Token ID 只是示意，真实 ID 取决于模型的 tokenizer。

现代大语言模型普遍采用**子词分词 (Subword Tokenization)**：将常见词（如 "agent"）保留为完整词元，将不常见词（如 "Tokenization"）拆分成多个有意义的子词片段（如 "Token"、"ization"），既控制了词表大小，又能通过组合子词理解与生成新词，**字节对编码 (Byte-Pair Encoding, BPE)** 是最主流的子词分词算法之一。

**Token Embedding**

Token ID 本身只是一个整数，模型需要把它映射到高维向量：`Token ID -> Embedding Lookup -> 向量`, 例如：`18372 -> [0.12, -0.83, 0.45, ..., 0.21]` 真实模型里可能是几千维甚至更多。于是：`Token IDs -> Embedding Vectors`。

**Transformer**

现代 LLM 大多基于 Transformer 架构，可以把一个 Transformer Block 简化为：`输入 -> Self-Attention -> Residual / LayerNorm -> Feed Forward Network -> Residual / LayerNorm -> 输出`, 模型有很多层：

```
Embedding
    ↓
Transformer Block 1
    ↓
Transformer Block 2
    ↓
Transformer Block 3
    ↓
...
    ↓
Transformer Block N
```

**Self-Attention**

这是模型理解上下文的重要机制。Attention 会让不同 Token 之间产生关联，会计算 Query Key Value，然后得到：每个 Token 对其他 Token 的关注程度。

**FFN / MLP**

Attention 之后通常进入 Feed Forward Network，`Attention -> FFN / MLP -> 下一层` 它负责进一步进行非线性变换和特征提取。整个 Transformer Block 可能在几十层甚至上百层反复执行。

**模型最后输出**

当最后一层处理完成以后： ```Hidden States -> LM Head -> Logits```, 例如词表里有 50000 个 token, 那么模型会给每一个候选 Token 一个分数, 这个叫：`Logits`, 它还不是最终概率。

```
"是"       2.1
"一个"     5.3
"可以"     1.7
"搜索"     4.8
"系统"     2.0
...
```

**Softmax：得到概率**

把 logits 转成概率 `Logits -> Softmax -> Probability`, 现在模型知道：下一个 Token 最有可能是什么。

```
"是"       0.05
"一个"     0.42
"可以"     0.03
"搜索"     0.31
"系统"     0.06
...
```

**Sampling：决定下一个 Token**

temperature 和 top_p 会影响从概率分布里怎么选 Token。

Token 生成以后，最终要转回文本 `Token IDs -> Tokenizer / Detokenizer -> 文本`，这个过程叫：Detokenization。


## 为什么模型反复计算

因为 LLM 通常不是一次计算整句话直接出来，而是：

```text
预测一个 Token
↓
再预测一个 Token
↓
再预测一个 Token
↓
...
```

例如：

```text
输入：
什么是 Elasticsearch？

模型生成：
Elasticsearch
```

然后再：

```
什么是 Elasticsearch？
+
Elasticsearch
```

继续预测：

```
是
```

再继续：

```
什么是 Elasticsearch？
Elasticsearch 是
```

核心循环就是：

```
┌──────────────────────┐
│ Context               │
└──────────┬───────────┘
           ↓
     Transformer
           ↓
        Logits
           ↓
       Sampling
           ↓
      Token N+1
           ↓
   加入 Context
           │
           └───────────────┐
                           │
                           ▼
                      Transformer
                           ↓
                         ...
```

这个过程叫：**自回归生成（Autoregressive Generation）**

## KV Cache

这个是理解 LLM 推理性能非常重要的一步。如果每生成一个 Token，都把整个历史重新计算一次，会很慢，所以推理过程中会缓存 Attention 的 `K = Key V = Value`, 也就是 `KV Cache` 。

第一次：`Context -> 计算 K/V -> 缓存`, 下一次生成 Token 时：`新 Token + 已有 KV Cache ->  只计算新增部分`, 因此 `Prefill` 和 `Decode` 是推理阶段两个非常重要的概念。

**Prefill**

假设：用户输入 1000 个 Token, 模型第一次需要处理整个输入：

```
1000 tokens
 ↓
Transformer
 ↓
KV Cache
```

这部分叫：Prefill, 它的主要工作是处理已有上下文。

**Decode**

接下来模型开始一个一个生成：

```
Token 1001
Token 1002
Token 1003
...
```

每次主要计算新增 Token，并利用 KV Cache。这个叫：Decode

所以完整推理：

```
Prompt
  ↓
Tokenization
  ↓
Prefill
  ↓
KV Cache
  ↓
Decode
  ↓
Token
  ↓
Decode
  ↓
Token
  ↓
Decode
  ↓
...
```

这就是为什么 AI 输出能够“一个 token 一个 token”地流出来。



如果把整个 LLM API 请求压缩成 12 步:

```
1. Client 组装 Request
       ↓
2. HTTP 请求发送
       ↓
3. API Gateway 鉴权 / 限流 / 路由
       ↓
4. Context 组装
       ↓
5. Tokenization
       ↓
6. Token IDs → Embedding
       ↓
7. Transformer / Attention / FFN
       ↓
8. Logits
       ↓
9. Sampling
       ↓
10. 生成下一个 Token
       ↓
11. Decode → Detokenization
       ↓
12. HTTP / SSE Response
```

其中第 7～10 步会不断循环：

```
Transformer
   ↓
Logits
   ↓
Sampling
   ↓
Token
   ↓
再次进入 Decode
   ↓
Transformer
   ↓
...
```

## 蒸馏

蒸馏（Knowledge Distillation，知识蒸馏）：让一个“大而强的模型（Teacher）”教一个“小而快的模型（Student）”，把大模型的能力尽可能迁移给小模型。

```
    Teacher Model
大模型 / 强模型
        │
        │ 生成答案、概率、推理/偏好等“知识”
        ▼
    Distillation
    蒸馏
        │
        ▼
    Student Model
小模型 / 快模型
```


## 模型幻觉

**模型幻觉是 LLM 的固有缺陷，也是驱动后续所有能力增强的原动力。**

为了提高大语言模型的可靠性，研究人员从数据、模型、推理与生成三个层面来检测和缓解幻觉：

- **数据层面**：通过高质量数据清洗、引入事实性知识以及强化学习与人类反馈（RLHF）等方式，从源头减少幻觉。
- **模型层面**：探索新的模型架构，或让模型能够表达其对生成内容的不确定性。
- **推理与生成层面**：
  - **检索增强生成（RAG）**：生成前先从外部知识库检索相关信息，注入上下文引导模型生成基于事实的回答（详见阶段二）。
  - **多步推理与验证**：引导模型多步推理，在每一步自我检查或外部验证。
  - **引入外部工具**：允许模型调用搜索引擎、计算器、代码解释器等，获取实时信息或精确计算（详见阶段三）。

> 正因为「单靠输入提示」不够可靠，下一步转向「输入侧增强」：不只优化提问的表达，更主动地**组织喂给模型的上下文**。