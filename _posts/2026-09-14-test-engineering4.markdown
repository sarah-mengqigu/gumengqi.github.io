---
layout: post
title: "以状态机建模的测试工程能力提升实践"
date: 2026-09-13
categories: Test-Engineering
excerpt: LLM模型擅长快速实现算法开发，可以通过建模，将业务问题转换成算法问题。这中间的桥梁，就是算法和大模型都易解析的建模描述文件（YAML文件）
---

## 一、关于生命周期管理业务的测试建模：状态机

### 1.1 一个真实的痛点场景

以一个云计算系统的裸金属服务器生命周期管理为例。服务器的状态包括：上架、交付、安装系统、运行中、已关机、已冻结、维护中、已下架等。系统定义了这些状态之间的状态机流转。

测试工程师面临的困境是：

- 状态多，转换路径组合爆炸，人工枚举测试路径力不从心
- 状态机频繁变化，每次变化都要回溯修改测试用例
- 异常路径（比如安装失败后重试、冻结后异常恢复）容易被遗漏

测试设计时虽然按照状态转换图进行了详尽的转换路径设计，但每次状态机的修改，都需要重新审视修改用例，判断哪些用例是新增的，哪些用例是需要修改的，以及哪些用例是需要删除的。如果改动点多，用例维护成本也会随之增加，人为遗漏和犯错的几率上升，已经超过了重新设计的认知和时间成本。

AI引入测试辅助后，我们开始重新审视以上过程的优化空间。模型擅长快速实现算法开发，可以通过建模，将业务问题转换成算法问题。这中间的桥梁，就是算法和模型都易解析的YAML文件

### 1.2 用YAML描述系统状态机

结构化的YAML文档既是对业务状态转换的描述又是一张有向连通图的描述

```yaml
states:
  - init
  - running
  - stopped
  - error
  - terminated

transitions:
  - from: init
    to: running
    action: CreateInstance Succeed
    description: OS 安装成功

  - from: init
    to: error
    action: CreateInstance Failed
    description: OS 安装失败

  - from: running
    to: stopped
    action: PowerDown
    description: 关机成功

  - from: running
    to: running
    action: PowerRebooting
    description: 重启成功

  - from: running
    to: terminated
    action: DeleteInstance Succeed
    description: 卸载OS成功

  - from: running
    to: error
    action: DeleteInstance Failed
    description: 卸载OS失败

  - from: stopped
    to: running
    action: PowerOn
    description: 开机成功

  - from: stopped
    to: error
    action: ChangeOS/Reinstall/ModifyIP Failed
    description: 切换OS/重装OS/修改IP失败

  - from: stopped
    to: running
    action: ChangeOS/Reinstall/ModifyIP Succeed
    description: 切换OS/重装OS/修改IP成功

  - from: stopped
    to: terminated
    action: DeleteInstance Succeed
    description: 卸载OS成功

  - from: error
    to: running
    action: ChangeOS/Reinstall/ModifyIP Succeed
    description: 切换OS/重装OS/修改IP成功

  - from: error
    to: terminated
    action: DeleteInstance Succeed
    description: 重试卸载OS成功
```
在看护产品相关业务质量的整个保障活动中，YAML文档提供了最基础的信息来源，这种文件的优势在于：

- **沟通语言统一**：不论是产品团队、开发团队还是质量团队，不论是测试关注业务还是关注执行，都能高效地从中提取信息
- **机器友好**：可以直接以DDT的范式集成到自动化框架，也可以作为AI辅助测试实践中SDT的Spec文件
- **可版本控制**：和代码一起进Git，变更可追溯，团队可评审，是可持续演进的优质资产
  
---

## 二、状态机背后的图论

### 2.1 状态机就是有向图

状态机在数学上天然对应**有向图**：

- **节点（Node）**：各个生命周期状态（init、running、stopped、error、terminated）
- **边（Edge）**：状态之间的转换（如 running → stopped 由 PowerDown 触发）

当同一个状态对之间存在多条转换（比如 stopped → running 既可以由“开机成功”触发，也可以由“配置变更成功”触发），这就构成了**多重有向图（MultiDiGraph）**。

### 2.2 图论能回答哪些测试问题

一旦把状态机抽象为图，成熟的图论概念就能派上用场：

强连通分量（SCC）：表示一组状态之间可以两两互相到达。注意，不要求两两直接相连，只要求存在路径。在状态机中，SCC对应状态循环——例如 running ↔ stopped 可以互相转换，构成一个强连通分量。

强连通分量有两种形态：

- 完全两两连通：所有节点之间都有双向直接边
- 环状连通：节点串成一个环，任意两点通过环上路径互相可达

无论哪种形态，只要在一个SCC内，就代表这些状态构成了一个“可以自由进出的功能模块”。这对测试策略至关重要：**SCC内部的转换需要循环测试，SCC之间的转换需要边界测试**。

欧拉路径：遍历图中每条边恰好一次的路径。在状态机测试中，这对应**覆盖所有状态转换的最短测试路径**。

欧拉路径的存在条件（有向图）：

- 欧拉回路：所有顶点入度=出度，且图强连通
- 欧拉路径：恰好一个顶点出度=入度+1（起点），恰好一个顶点入度=出度+1（终点），其他顶点入度=出度

寻找欧拉路径使用Hierholzer算法，时间复杂度O(|E|)，是图论中的经典算法。

DFS路径：深度优先搜索产生的路径，倾向于走尽可能长的路径，覆盖尽可能多的状态。

度中心性：节点的入度和出度。入度中心性高的状态是“受欢迎状态”（很多状态指向它），出度中心性高的状态是“决策状态”（它可以转向很多其他状态）。

**有了以上知识，我们就可以对照业务设计YAML文件结构，保证文件内定义的字段准确描述了所有的业务关系及变化。同时在编写YAML文件时，以图论为数学依据，可以提前识别出业务设计上的缺陷**

### 2.3 矩阵视角：多重有向图的数学表示

图论分析背后是矩阵运算。多重有向图可以用多种矩阵表示：

**权值邻接矩阵**：矩阵元素 M[i][j] = 从状态i到状态j的边数量。这是**最适合测试用例输出**的表示，因为：

- 保留了多重边信息
- 兼容矩阵运算（求幂、求传递闭包）
- 支持覆盖度分析

布尔邻接矩阵：只表示“是否存在转换”，适合基本可达性分析，但丢失多重边信息。

概率转移矩阵：用于马尔可夫链分析、稳态分布计算、基于风险的测试。

张量表示：三维张量（源状态×目标状态×动作类型），保留完整动作信息，适合细粒度分析，但计算复杂度高。

边列表：最详细的边信息，直接用于测试用例生成，但不适合矩阵运算。

Incidence矩阵：适合网络流分析，但矩阵通常很稀疏。

**有了以上知识，我们可以寻找合适的python数学工具计算出测试路径，无需从头实现算法。**

---

## 三、图论到算法：AI Coding全搞定

### 3.1 NetworkX：图算法的标准库

使用现有**NetworkX**——Python生态中成熟的图论库。

在NetworkX中，我们可以：

- 用 `nx.MultiDiGraph()` 创建多重有向图
- 用 `add_edge(from, to, action=..., description=...)` 记录状态转换及其语义信息
- 用 `nx.strongly_connected_components(G)` 直接计算强连通分量
- 用 `nx.eulerian_circuit(G)` 直接生成欧拉路径
- 用 `nx.pagerank(G)` 直接计算PageRank
- 用 `nx.in_degree_centrality(G)` 和 `nx.out_degree_centrality(G)` 计算中心性


### 3.2 如何从NetworkX获取action信息

MultiDiGraph将action存储在边的属性中。获取方式：

```python
# 获取从某状态出发的所有出边（含action）
for u, v, data in G.out_edges("running", data=True):
    print(f"{u} -> {v}: {data['action']}")

# 处理多重边（获取keys）
for u, v, key, data in G.edges("stopped", keys=True, data=True):
    print(f"键{key}: {u} -> {v}: {data['action']}")
```

关键区别：

- `G.out_degree(node)` 返回出度（边数量）
- `G.out_edges(node, data=True)` 返回出边详情（含action）

在状态机测试中，我们同时需要两者：

- 用**度**做统计分析和中心性计算
- 用**edges**获取具体的action生成测试用例

### 3.3 AI Coding：让AI写算法，业务事实验证算法

到这里，一个完整的分析脚本就呼之欲出了。

比如，你可以这样描述需求：

> “帮我用Python + NetworkX开发一个状态机测试框架，需求：
> 1. 从YAML加载状态机
> 2. 支持DFS/欧拉/贪心/混合路径生成策略
> 3. 输出测试用例骨架（预置条件、步骤、预期结果）
> 4. CLI接口，输出JSON

LLM会在几分钟内生成一个完整可用的算法，在用已有的测试用例在交付版本上验证脚本正确性。在智能化更高的AI Agent上，可以实现算法脚本自动bugfix和修改适配。


### 3.4 一个自包含的分析脚本

最终形态是一个自包含的Python脚本 `state_machine_analyzer.py`，它：

- 支持CLI调用：`python state_machine_analyzer.py analyze --yaml state.yaml`
- 支持模块导入：`from state_machine_analyzer import StateMachineAnalyzer`
- 输入是YAML，输出是JSON
- 提供`schema`命令供LLM自主探索能力
- 提供`full`命令一键完成全流程

整个脚本的头部有完整文档，LLM直接读取就能理解其能力。这是“脚本即文档，能力即可见”的设计哲学。

---

## 五、状态图模型作为Spec的SDT（Spec-Driven-Test）实践

至此，我们有了YAML模型，有了NetworkX算法，有了自包含的Python脚本。这三者组合起来，形成一个完整的**SDT（Spec-Driven-Test）实践**——**把模型本身作为规格说明（Spec），驱动测试的生成与评估**。

它包含三个核心环节：

### 5.1 状态机分析

第一个环节是对状态机本身进行数理分析，回答“这个状态机的结构如何”。

**输出内容包括**：

- **基础统计**：总状态数、总转换数、初始状态、终止状态
- **连通性**：是否强连通、强连通分量分组
- **状态详情**：每个状态的入度、出度、PageRank、所属SCC
- **关键状态识别**：最重要的状态（PageRank最高）、最频繁的转换
- **复杂度指标**：图密度、平均分支因子

**现实指导意义**：

- 在开发初期的设计阶段，识别状态设计缺陷（是否有重复的边，没有连通的节点）
- 大型SCC（多状态循环）是复杂功能模块，需要重点测试循环逻辑和边界条件
- 高PageRank状态是核心枢纽，需要重点测试和监控
- 初始/终止状态需要特别的验证逻辑

用一条命令即可完成：

```bash
python state_machine_analyzer.py analyze --yaml state.yaml
```

### 5.2 测试路径生成

第二个环节是生成测试路径，回答“应该测试哪些状态转换序列”。

支持**五种路径生成策略**，各有优劣：

**DFS路径**：

- 特征：单条长路径，倾向走尽可能多的状态
- 优势：执行效率高、测试用例简单清晰、资源消耗少
- 劣势：可能错过重要循环路径、循环测试不充分、边界条件覆盖不足
- 适用场景：冒烟测试、回归测试主路径

**SCC循环路径**：

- 特征：为每个强连通分量生成循环路径
- 优势：充分测试状态循环和复杂交互、覆盖所有可能的循环路径、更容易发现循环相关缺陷
- 劣势：可能产生多条测试用例、执行时间较长、需要更多资源
- 适用场景：完整性测试、性能测试、边界测试

**欧拉路径**：

- 特征：遍历每条边恰好一次（当图满足欧拉条件时）
- 优势：路径最短、测试用例数量最少、覆盖所有转换一次
- 劣势：条件严格（许多实际状态机不满足）
- 适用场景：转换覆盖测试

**贪心覆盖路径**：

- 特征：优先选择覆盖未访问边/节点的下一步
- 优势：适用于任意图、覆盖效率较高
- 劣势：不保证最优解
- 适用场景：通用场景

**混合策略（推荐）**：

- 组合DFS和SCC路径，按边覆盖度排序，贪心去重
- 在覆盖度和路径数量之间取得平衡

**从测试覆盖角度的重要结论**：

| 维度 | SCC路径 | DFS路径 |
|------|---------|---------|
| 深度覆盖 | 高（循环充分） | 中 |
| 缺陷发现（循环/资源泄漏/并发） | 高 | 低 |
| 测试效率 | 中 | 高 |
| 测试用例数量 | 多 | 少 |
| 冒烟测试 | 不适合 | 适合 |
| 完整性测试 | 适合 | 不适合 |

**推荐实践：混合使用两类路径**——DFS覆盖主线，SCC覆盖循环；根据测试阶段动态调整策略。

用一条命令即可生成：

```bash
python state_machine_analyzer.py paths --yaml state.yaml --strategy hybrid --max-paths 5
```

### 5.3 从路径到用例：LLM的语义化

通过算法脚本生成的路径作为AI辅助测试用例生成的输入：

> 基于以下测试路径信息，生成详细的测试用例：
>
> 路径: init → running → stopped → running → terminated
> 边信息:
> - init → running: CreateInstance Succeed (OS安装成功)
> - running → stopped: PowerDown (关机成功)
> - stopped → running: api_power_on (开机成功)
> - running → terminated: DeleteInstance Succeed (卸载OS成功)
>
> 请为这个路径生成包含以下内容的测试用例：
> 1. 测试用例名称
> 2. 预置条件
> 3. 测试步骤
> 4. 预期结果
> 5. 优先级

LLM会生成一份结构化的用例文档：

```
### TC-001: 服务器正常关机重启流程

**优先级**: P0
**业务场景**: 运维人员对运行中的服务器执行关机维护，维护完成后重新开机

**预置条件**:
- 服务器实例已创建且处于"运行中"状态
- 服务器已安装 CentOS 7.9 操作系统
- 运维平台可正常访问

**测试步骤**:
1. 在运维平台选择目标服务器，点击"关机"
   - 预期：接收到 PowerDown 请求
2. 等待关机操作完成
   - 预期：状态从"运行中"变为"已关机"
3. 在运维平台点击"开机"
   - 预期：接收到 api_power_on 请求
4. 等待开机操作完成
   - 预期：状态从"已关机"变回"运行中"

**验证点**:
- 状态流转: running → stopped → running
- 关键动作: PowerDown, api_power_on
- 数据一致性: 关机前后服务器配置保持不变
```

这就是SDT的完整闭环：

- **Python负责数理分析**：状态机结构、路径生成、覆盖度评估
- **LLM负责语义生成**：把技术路径翻译成业务语言用例
- **YAML作为Spec**：YAML既是规格说明，又是测试生成的输入，又是回归验证的基准


### 5.4 测试覆盖度评估

第三个环节是评估测试的覆盖完整性，回答“哪些状态和转换还没被覆盖”。

**输出内容包括**：

- **覆盖度摘要**：总状态数/已覆盖数、总转换数/已覆盖数、覆盖率百分比
- **覆盖缺口**：未覆盖的状态列表、未覆盖的转换列表
- **改进建议**：基于未覆盖情况，自动推荐应采用的策略

**现实指导意义**：

- 未覆盖的高频转换是测试盲区，需要补充用例
- 覆盖度不足时，可切换策略或增加路径数量

用一条命令即可评估：

```bash
python state_machine_analyzer.py coverage --yaml state.yaml --paths paths.json
```

### 5.5 另一种建模

本文作者另一篇实践[《从组合爆炸到精准覆盖：IaaS解决方案测试的场景空间建模与工程化实践》](https://sarah-mengqigu.github.io/gumengqi.github.io/test-engineering/2026/09/09/test-engineering1.html
)同样也使用了MBT的方法。

区别于本文中状态机模型的业务，针对的是复杂场景组合的建模。但建模作为Spec的SDT范式是一脉相承的：

算法负责数理分析：状态机结构分析、路径生成、场景空间分析、双维覆盖度评估
LLM负责语义生成：把“路径×场景”翻译成业务语言的、可交付的测试用例
模型作为Spec：
- 状态机YAML是时序维度的规格说明
- 场景平面YAML是空间维度的规格说明
- 两者共同构成完整的解决方案规格，既是测试生成的输入，又是回归验证的基准
---


# 六 我是怎么落地的

算法脚本的落地实际上通过提示词和模型对话就能实现，但是还需要人和真实业务对齐，调试脚本，把脚本集成到框架一系列工作。那不妨……

![prompt]({{ "assets/img/blog/create skill state machine.png" | relative_url }})

直接把上面的章节文字丢给AI Agent，Agent开启框架项目目录，调用/skill-creator，让Agent根据该篇文章自主规划，

在codex执行的结果，可以看到Agent是严格按照本篇文章规划落地的:
![agent_response]({{ "assets/img/blog/create skill state machine reps.png" | relative_url }})


**那么也就是说，在Agent时代，可复用的不再是只是工程代码，高质量的技术文章也可以 ：）**
