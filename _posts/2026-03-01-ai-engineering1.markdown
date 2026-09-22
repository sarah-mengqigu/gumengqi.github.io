---
layout: post
title: 当Agent学会自己找知识，RAG还剩下什么？
date: 2026-03-01
categories: AI-Engineering
excerpt: RAG 正在从一个“生成增强技术”，重新靠近它的第一性原理——Information Retrieval，沉淀成 Agent 背后的信息检索基础设施
---

如果把大语言模型看成一个“知道很多东西、但知识封存在参数里的大脑”，RAG（Retrieval-Augmented Generation，检索增强生成）最初解决的问题其实很朴素：

**能不能让模型在回答问题之前，先去外部世界找一找？**

2020 年，Lewis 等人在论文 *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks* 中正式提出 RAG 这一框架。论文将预训练语言模型的参数化记忆，与外部的非参数化记忆结合起来，并通过检索器从 Wikipedia 等知识库中寻找相关内容，再交给生成模型完成回答。([arXiv][1])

此后几年，RAG 几乎成为 LLM 应用开发的标准技术路线之一。

但到了 Agent 开始真正能够读文件、搜索、调用工具、执行代码之后，一个有意思的问题出现了：

> **如果 Agent 自己已经会找东西了，我们还需要专门做 RAG 吗？**

这个问题，恰好可以成为理解 RAG 演化的一条线索。

---

# 第一章：RAG 是什么？

## 1. 从“模型记忆”到“外挂记忆”

LLM 的知识主要存在于模型参数中。

这带来三个天然问题。

第一，**知识会过时**。

模型训练完成以后，现实世界仍然在变化。

第二，**模型很难精确控制自己的知识来源**。

即使模型“知道”某件事情，也很难回答：

> 这个结论来自哪一份文档？

第三，**模型并不适合承担所有外部知识的存储任务**。

企业内部的几十万份文档、客户数据、产品手册、法规文件，不可能为了让模型知道它们，就全部重新训练模型。

RAG 的思路于是非常直接：

```text
                    LLM
                     ↑
                     │
                Retrieved Context
                     ↑
                     │
User Query → Retriever → Knowledge Base
```

模型不再只依赖自己的参数记忆，而是在推理过程中临时获得一部分外部知识。

所以 RAG 的核心并不是“向量数据库”。

真正的思想是：

> **把模型的参数化知识，与外部可更新、可检索的非参数化知识结合起来。**

这也是最初 RAG 论文的核心定义。([arXiv][1])

---

## 2. 传统 RAG 到底怎么工作？

后来工程界把这个思想逐渐发展成了一条相当固定的流水线：

```text
             离线阶段

原始文档
   ↓
Document Parsing
   ↓
Chunking
   ↓
Embedding
   ↓
Vector Index
```

用户提问时：

```text
             在线阶段

用户问题
   ↓
Query Embedding
   ↓
Vector Search
   ↓
Top-K Chunks
   ↓
Context Assembly
   ↓
LLM
   ↓
Answer
```

其中最重要的是中间那一步：

> **把巨大的知识空间压缩成少量与问题相关的上下文。**

例如公司有 100 万个文本片段。

用户问：

> “我们的 API 网关超时配置是多少？”

LLM 不需要看到 100 万个 chunk。

RAG 的任务是：

```text
1,000,000 chunks
       ↓
   retrieval
       ↓
      20
       ↓
    rerank
       ↓
       5
       ↓
      LLM
```

所以从本质上说：

**RAG 是一个搜索空间压缩器。**

---

## 3. Chunk 为什么成为 RAG 的第一个难题？

文档太大，不能直接作为一个向量。

于是需要把它切成 chunk。

最简单的办法：

```text
每 500 tokens
切一次
```

但很快就会发现，chunk 并不是纯粹的工程参数。

切得太小：

> 语义不完整。

切得太大：

> 一个向量里面塞进了太多不同的信息，检索粒度又变粗。

于是开始出现：

* overlap chunking
* semantic chunking
* parent-child chunking
* hierarchical chunking
* contextual chunking
* late chunking

其中一个很有代表性的方向是 **Late Chunking**。后面章节后展开介绍

这其实暴露了 RAG 的一个根本问题：

> **检索单位究竟应该是什么？**

Chunk 从来不是天然存在的东西。

它只是为了让机器能够检索而人为制造出来的单位。

---

## 4. 从 Vector Search 到真正的信息检索

早期 RAG 很容易被简化成：

```text
Embedding
   ↓
Vector DB
   ↓
Top-K
```

但真正投入生产以后，人们很快发现：

**语义相似 ≠ 真正相关。**

于是现代 RAG 往往逐渐变成：

```text
                  Query
                    ↓
       ┌────────────┼────────────┐
       ↓            ↓            ↓
    Dense         Sparse      Metadata
    Search        Search       Filter
       ↓            ↓            ↓
       └────────────┼────────────┘
                    ↓
                 Fusion
                    ↓
                 Rerank
                    ↓
             Context Assembly
                    ↓
                   LLM
```

Dense retrieval 擅长语义匹配。

BM25 等 sparse retrieval 擅长精确词项匹配。

Metadata filter 可以处理时间、部门、权限、文档类型等结构化条件。

Reranker 再从候选结果中判断哪些真正与问题相关。

到了这个阶段，RAG 已经越来越不像“向量数据库应用”，而更像一个完整的信息检索系统。

ColBERT 是一个很有代表性的例子：它没有把一个 passage 压成单一向量，而是保留 token-level representation，并通过 late interaction 对 query 与 passage 进行更细粒度的匹配。([arXiv][3])

---

# 第二章：Agent 出现了——渐进式披露吞并 RAG 的一部分领地

RAG 曾经有一个非常重要的隐含前提：

> **LLM 不会自己寻找知识，所以我们必须在它回答之前，把知识检索出来。**

Agent 出现之后，这个前提开始松动。

一个 Agent 不再只是：

```text
输入 → LLM → 输出
```

而是：

```text
输入
 ↓
Agent reasoning
 ↓
选择工具
 ↓
读取文件
 ↓
搜索
 ↓
继续读取
 ↓
执行代码
 ↓
获得结果
 ↓
继续 reasoning
```

于是出现了一个非常有意思的替代方案：

> **为什么一定要先把文档切 chunk、embedding、召回？**

如果 Agent 可以直接浏览文件系统，它也许可以自己找到需要的文档。

---

## 1. Progressive Disclosure：把知识空间变成可导航空间

Anthropic 在 2025 年公开介绍 Agent Skills 时，把这种思想明确化了。

一个 Skill 不需要把所有知识一次性塞进上下文，而是：

```text
Skill metadata
      ↓
发现相关 Skill
      ↓
读取 SKILL.md
      ↓
发现 reference
      ↓
读取 reference
      ↓
继续执行
```

这就是 **progressive disclosure（渐进式披露）**。

Skill 首先只提供名称和描述；当 Agent 判断它相关时，再读取完整 `SKILL.md`；如果还需要更详细的信息，再继续读取目录里的其他文件。([Anthropic][4])

于是，一个巨大的知识空间不必被一次性压缩成向量。

它可以被组织成：

```text
skills/
├── authentication/
│   ├── SKILL.md
│   └── oauth.md
│
├── database/
│   ├── SKILL.md
│   └── migration.md
│
└── deployment/
    ├── SKILL.md
    └── kubernetes.md
```

Agent 自己决定：

> 我要先看哪个目录？

这和传统 RAG 的思路其实很不一样。

RAG 是：

> **机器先替你找。**

Progressive disclosure 是：

> **让 Agent 自己导航。**

Anthropic 对 context engineering 的进一步讨论也明确指出，Agent 可以通过自主探索逐步发现相关上下文，而不是一次性把所有信息塞进 context；代价则是运行时探索会比预计算检索更慢，也更依赖 Agent 的工具使用能力和导航策略。([Anthropic][5])

---

## 2. 为什么 Vibe Coding 特别容易让人“感觉 RAG 没那么重要了”？

代码项目恰好是 Progressive Disclosure 的理想场景。

假设一个项目：

```text
project/
├── README.md
├── src/
├── tests/
├── docs/
├── scripts/
└── skills/
```

用户说：

> “给这个项目增加 OAuth。”

Agent 可以：

```text
查看项目结构
 ↓
寻找 auth 相关文件
 ↓
阅读 README
 ↓
读取 skill
 ↓
查看现有认证代码
 ↓
搜索依赖
 ↓
修改
 ↓
运行测试
```

这里其实没有必要先做：

```text
所有代码
 ↓
chunk
 ↓
embedding
 ↓
vector DB
 ↓
top-k
```

因为**代码库本身就是一个可导航的信息空间**。

这也是为什么在今天的 Agentic coding / vibe coding 场景里，很多开发者会产生和你一样的体感：

> “我怎么没感觉 RAG 有那么必要？”

不是 RAG 失效了，而是 **Agent 的 filesystem + search + tools + progressive disclosure 接管了部分原本属于 RAG 的工作。**

---

## 3. 那么 RAG 被留下了什么？

这里反而出现了一个更清晰的边界。

### 第一类：海量业务数据

例如：

* 客户历史数据
* 海量工单
* 交易记录
* 历史日志
* 企业知识库
* 用户行为记录

问题不在于 Agent 不会读。

而在于：

> **数据空间实在太大。**

如果有几百万份记录，让 Agent 自己：

```text
搜索
→ 阅读
→ 判断
→ 再搜索
→ 再阅读
```

成本会迅速上升。

所以 RAG / Search 的价值变成：

> **提前把巨大的搜索空间压缩成候选集合。**

---

### 第二类：需要证据链的知识

比如：

* 法律法规
* 公司制度
* 合同
* 医疗指南
* 金融政策
* 技术规范

这时候问题也不是：

> “Agent 能不能读？”

而是：

> **“你能不能证明这个答案来自哪里？”**

因此：

```text
用户问题
 ↓
检索权威文档
 ↓
定位具体条款
 ↓
回答
 ↓
引用原文 / 来源
```

RAG 在这里承担的不只是 knowledge retrieval，而是：

**grounding + provenance + evidence。**

---

## 4. 还有一种容易被误认为 RAG 的东西：Memory

Agent 的长期记忆也需要 retrieval，但 Memory Retrieval 并不等于 RAG。

例如：

> “我上个月为什么放弃那个方案？”

可以使用：

```text
SQL / KV
```

也可以：

```text
Vector Search
```

还可以：

```text
Knowledge Graph
```

或者：

```text
时间 + metadata
```

所以：

> **Memory Retrieval 是问题，RAG 只是其中一种实现。**

这也说明，Agent 时代真正重要的问题已经从：

> “我要不要 RAG？”

变成：

> **“这种信息应该以什么形式存储？Agent 应该如何找到它？”**

---

# 第三章：RAG 走向 Agentic RAG

如果说第一阶段的 RAG 是：

> **我替 LLM 找知识。**

那么第二阶段的 RAG 是：

> **我把检索能力做得更强。**

而现在正在出现的第三阶段，则更接近：

> **让 Agent 决定什么时候检索、检索什么、怎么检索，以及什么时候停止检索。**

这就是所谓的 **Agentic RAG**。

已有的 Agentic RAG 研究综述也将这一变化概括为：Agent 不再被限制在固定的 retrieval pipeline 中，而是可以利用 planning、reflection、tool use 等机制动态调整检索策略，并在检索与推理之间反复迭代。([arXiv][6])

---

## 1. 哪些东西已经沉淀成基础设施？

经过几年的发展，RAG 最底层的一些东西其实已经非常成熟。

### 向量索引

HNSW、IVF、PQ、DiskANN 等 ANN 技术已经形成相当成熟的基础设施。

对于普通应用开发者来说：

> **自己发明一个更快的 vector index，通常已经不是 RAG 应用最值得投入的地方。**

---

### Vector Database

向量存储本身也已经基础设施化。

它越来越像：

```text
PostgreSQL
Redis
Object Storage
Vector Search
```

中的一种基础能力。

真正复杂的问题已经不再只是：

> “怎么把 vector 存进去？”

而是：

> “这个 vector 到底代表什么？”

---

### Embedding API / Model

Embedding 已经高度服务化。

调用模型得到 representation，然后建立索引，已经不再是 RAG 的主要创新点。

当然 embedding 模型本身仍然在进步，但应用开发者真正需要关注的往往已经从：

> “哪个 embedding API？”

转向：

> **“我的信息如何表示，才能被正确检索？”**

---

### 基础 RAG Pipeline

这种：

```text
Load
→ Chunk
→ Embed
→ Store
→ Retrieve
→ Generate
```

已经基本成为一种工业设计模式。


---

# 2. 当下真正值得关注的，是 Retrieval Quality

RAG 现在真正困难的地方越来越集中在：

> **“到底应该召回什么？”**

而不是：

> “怎么把向量找出来？”

这两个问题差别很大。

例如用户问：

> “公司为什么在 2024 年放弃 Kafka？”

最简单的 semantic search 可能找到：

```text
Kafka 使用文档
Kafka 部署指南
Kafka 性能测试
Kafka 与 Pulsar 对比
2024 架构决策会议纪要
```

真正有价值的可能只有最后两份。

所以未来的检索系统越来越需要：

```text
Query Understanding
        ↓
Query Expansion
        ↓
Hybrid Retrieval
        ↓
Metadata Filtering
        ↓
Reranking
        ↓
Context Reconstruction
```

---

# 3. Chunking 仍然没有结束

这是一个很值得关注的方向。

传统思路是：

> 文档 → chunk → embedding

但现在越来越多人开始重新思考：

> **为什么必须先把文本切碎，再独立理解？**

Late Chunking 就是一个代表。

它先利用长上下文 embedding 模型理解整个文档，再生成 chunk representation，使局部片段能够保留更多全局上下文。([arXiv][2])

另一个方向则是 multi-vector / late interaction。

ColBERT 的思路就是不强迫整个 passage 变成一个向量，而保留更细粒度的 token-level representations，再在 query 与 document 之间做交互。([arXiv][3])

这背后其实是同一个问题：

> **单向量是不是过度压缩了语言？**

---

# 4. RAG 正在从“Vector Search”变成“Information Retrieval”

如果未来继续发展，RAG 很可能越来越不像：

```text
Vector DB
+
LLM
```

而越来越像：

```text
             Information Space
                    │
       ┌────────────┼─────────────┐
       ↓            ↓             ↓
    Keyword       Vector       Structure
    Search        Search        Search
       │            │             │
       └────────────┼─────────────┘
                    ↓
                 Rerank
                    ↓
             Context Builder
                    ↓
                  Agent
                    ↓
        ┌───────────┼───────────┐
        ↓           ↓           ↓
      Search       Tool       Reasoning
        ↓           ↓           ↓
        └───────────┼───────────┘
                    ↓
                  Answer
```

也就是说：

**RAG 正在从一个“生成增强技术”，重新靠近它的第一性原理——Information Retrieval。**

只是以前搜索系统把结果给人看。

现在搜索系统把结果给 Agent 看。

---

# 5. 最终的变化：RAG 不再是主角，而是 Agent 的一种能力

如果把这段历史压缩成四个阶段，大概是：

```text
第一阶段

LLM
 ↓
我知道很多
```

↓

```text
第二阶段

LLM
 ↓
RAG
 ↓
外部知识

“模型不会的，我替它找。”
```

↓

```text
第三阶段

Agent
 ↓
Skills / Files / Search / Tools
 ↓
Progressive Disclosure

“模型自己可以去找。”
```

↓

```text
第四阶段

Agent
 ↓
决定是否需要检索
 ↓
决定去哪里检索
 ↓
选择 Search / Vector / DB / API
 ↓
反复检索 + Reasoning
 ↓
形成答案

“模型不仅会找，还会决定怎么找。”
```

这就是 Agentic RAG 更值得关注的地方。

它并不是简单地：

> **“给 RAG 加一个 Agent。”**

而是 RAG 的角色发生了变化：

> **从一个固定的信息注入管道，变成 Agent 可以调用、组合、规划和反思的一种认知工具。**

---

# 总结：RAG 没有消失，它只是退回了自己真正擅长的地方

回头看我们最开始的问题：

> **Agent 时代还需要 RAG 吗？**

答案可能不是“需要”或者“不需要”。

更准确的说法是：

**Agent 正在吞掉 RAG 原来负责的一部分工作。**

当知识是：

* 少量的
* 结构清晰的
* 可以通过文件系统导航的
* Agent 能够逐步发现的
* 与具体工作流程绑定的

那么 Skills、filesystem、search 和 progressive disclosure 往往已经足够。

但当问题变成：

> **“从数百万条信息里找到相关的几十条。”**

或者：

> **“告诉我这个结论，并给出能够追溯的权威证据。”**

RAG / Information Retrieval 依然不可替代。

所以今天再理解 RAG，我觉得最重要的已经不是背诵：

> chunk → embedding → vector DB → top-k → LLM

而是建立一个更大的判断框架：

> **什么信息应该放进 context？**
> **什么信息应该成为 Agent 的 Skill？**
> **什么信息应该进入 Memory？**
> **什么信息需要 Search？**
> **什么信息必须通过 Retrieval 提供证据？**
> **什么时候应该让 Agent 自己探索，什么时候应该让索引替它缩小搜索空间？**

从这个角度看，RAG 的历史其实并不是一个“技术被新技术淘汰”的故事。

更像是：

> **RAG 最初试图替 LLM 找知识；Agent 出现以后，开始学会自己找知识；而 RAG 最终沉淀成 Agent 背后的信息检索基础设施。**

这可能才是 RAG 从 **Retrieval-Augmented Generation** 走向 **Agentic Retrieval** 最有意思的一步。


[1]: https://arxiv.org/abs/2005.11401?utm_source=chatgpt.com "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks"
[2]: https://arxiv.org/abs/2409.04701?utm_source=chatgpt.com "Late Chunking: Contextual Chunk Embeddings Using Long-Context Embedding Models"
[3]: https://arxiv.org/abs/2004.12832?utm_source=chatgpt.com "ColBERT: Efficient and Effective Passage Search via Contextualized Late Interaction over BERT"
[4]: https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills?utm_source=chatgpt.com "Equipping agents for the real world with Agent Skills \ Anthropic"
[5]: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents?utm_source=chatgpt.com "Effective context engineering for AI agents \ Anthropic"
[6]: https://arxiv.org/abs/2501.09136?utm_source=chatgpt.com "Agentic Retrieval-Augmented Generation: A Survey on Agentic RAG"
