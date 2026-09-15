---
layout: post
title: “一次测试左移的全流程实践：数据集目录结构检查脚本开发"
date: 2026-09-15
categories: Test-Engineering
excerpt: AI平台数据集100%覆盖检查的自动化方案
---

# 数据集文件目录标准化检查技术方案总结

## 1. 背景与需求

在模型开发平台提供的数据集管理场景中，往往需要自动化验证目录结构、文件命名是否符合预定义规范。主要需求包括：
- 检查目录嵌套层级是否与规范模板一致
- 验证特定目录下的文件名是否符合命名规则
- 支持本地文件系统及云存储（OBS）的检查
- 一些特定文件，特定格式的约束

本文基于实际技术选型和问题解决过程，整理了可用的工具、配置方法及关键问题处理。

---

## 2. 规范模版格式

文件目录结构可以使用YAML\JSON格式文件来表示，JSON文件在算法脚本里读取时更方便，故选用JSON文件。

Lerobot是具身数据的主流格式，里面记录了数采过程中，不同位姿下，各角度相机记录的视觉文件和一些元数据。

一个遵循 v3.0 规范的 LeRobot 数据集，其根目录下的标准组织如下：

    meta/ (元数据目录)：包含所有描述数据集的核心文件。

        info.json：全局元数据描述文件，定义数据集的核心信息（如版本、机器人类型、频率、总帧数等）和所有特征的完整模式（schema）。

        stats.json：存储所有特征的归一化统计量（如均值、标准差、最大值、最小值），用于模型训练时的数据标准化。

        tasks.parquet：任务定义文件，用 Parquet 格式存储。

        episodes/：存储 Episode 元数据的目录，其文件结构同样是分块的 Parquet 文件。

    data/ (数据目录)：存放机器人观测和动作数据的 Parquet 文件。这些文件被组织成多个分块目录。

    videos/ (视频目录)：存放机器人摄像头录制的 MP4 视频文件，按不同的视频流（如 observation.images.laptop）和分块目录组织


设计AssetDataset数据结构，是概括了meta-data-videos标准组织的一个单元。
脚本要先按照厂商-动作-位姿逐层访问到内部的obs对象，然后按照AssetDataset数据结构对LeRobot数据集内部继续访问。

以JSON为例，lerobot机器人数据可以表示为如下：
```json
{
    "Adversarial": {
        "HangCupsOnRack": {
            "hang_the_cup_on_rack_left_arm": "AssetDataset",
            "hang_the_cup_on_rack_right_arm": "AssetDataset"
        },
        ...
    }
}
```

以上JSON文件作为资产目录Spec文件作为关键约束，是测试自动化检查的依据来源。


## 3. 工具选型

检索开源工具，找到了下面两种方案：

### 3.1 validir

**特点**：通过 YAML 模板声明式定义目录结构，支持 `allow_extra`、`check_hidden` 等标志，支持通配符匹配。

### 3.2 repo-structure（推荐作为 validir 的替代）

**特点**：通过 YAML 规则文件定义目录约束，灵活支持 `require`（必须存在）、`allow`（允许存在）、`forbid`（禁止存在）、`companion`（关联文件）及递归检查。

以上两种方案都是要将OBS中的资产下载到本地的，实际测试要达到100%覆盖检查就意味着，将所有资产都下载到执行机，对网络开销和本地存储的需求都是不合理的。

### 3.3 云存储（OBS）上的目录检查

OBS（对象存储服务）本质是扁平存储，其“目录”是通过对象 Key 中的分隔符（如 `/`）模拟。检查其结构有以下几种方案。

### 3.4 方案对比

| 方案 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| **挂载工具**（obsfs/s3fs/rclone） | 将 OBS 桶挂载为本地文件系统，然后使用本地工具 | 无需修改现有检查脚本 | obsfs 已于2024年7月停止维护；网络延迟影响性能 |
| **obsutil 命令行** | 调用 OBS API 列出对象，结合脚本解析 | 简单直接，适合一次性或批量检查 | 需解析文本输出，灵活性较低 |
| **Python SDK**（esdk-obs-python） | 编程方式调用 API，实现任意复杂校验逻辑 | 灵活、高效，可深度集成 | 需编写代码 |

结合优缺点分析，排除掉前两个的方案，只能自研一套通过逐级访问OBS目录的算法了

## 算法方案：DFS + OBS SDK

集成关键接口：

```python
from obs import ObsClient

obs_client = ObsClient(
    access_key_id='YOUR_AK',
    secret_access_key='YOUR_SK',
    server='https://obs.cn-north-1.myhuaweicloud.com'
)

def list_all_objects(bucket_name, prefix=''):
    marker = None
    while True:
        resp = obs_client.listObjects(bucket_name, prefix=prefix, marker=marker)
        if resp.status < 300:
            for obj in resp.body.contents:
                print(obj.key)   # 对象的完整路径
            if resp.body.is_truncated:
                marker = resp.body.nextMarker
            else:
                break
        else:
            raise Exception(f"Error: {resp.errorCode}")

list_all_objects('my-bucket', prefix='data/')
```

使用正则匹配，逐级获取obs路径下的对象元数据，再从元数据中取得下一级路径，递归下去：

```python
import re
pattern = re.compile(r'/asset-dataset-bucket-0/asset/[^/]+/$')
```

## Pytest框架集成方案

### 1. 资产配置管理

平台资产保存在服务管理账号的OBS桶内，资产列表保存在测试OBS桶内。

资产列表可以用JSON文件保存：
```json
{
    "SO-101": "obs://asset-dataset-bucket-0/asset/239018/203001/49921",
    "Lele": "obs://asset-dataset-bucket-0/asset/239018/203001/48810",
    "Sony": "obs://asset-dataset-bucket-0/asset/239018/223001/10990",
    ...
}
```

测试环境配置服务管理员账号的AK/SK和测试账号的AK/SK。测试账号保存在全局配置，管理员账号分region保存在region环境配置中。

**AK/SK是敏感数据，需要在传输中作安全加密**

global.toml：
[obs]
ak=**************
sk=**************

cn-north-1.toml
[obs]
ak=**************
sk=**************


### 2. 分层设计
Config：测试框架维护资产列表，和资产目录Spec文件，每轮迭代新增的资产追加到列表内，测试框架自动从测试OBS桶下载列表。
Service：测试框架集成OBS SDK，通过SDK访问资产对应的OBS路径。
Fixture：定义一个fixture_asset_check函数，传入指定的资产OBS路径，根据资产目录Spec文件，递归遍历资产OBS路径，检查结果计入结果字典。再定义一个fixture_res_dict，统计结果字典“所有叶子节点整数值的总和”和“叶子节点总数“
Tests：测试用例迭代检查整个资产列表：迭代调用fixture_asset_check，每次检查把结果记录到内存，所有资产检查结束后，迭代全量结果字典，调用fixture_res_dict判定每个资产检查结果，这样就不会因为中间某个资产检查未通过中断测试。

### 3. 断言设计
除了访问OBS路径需要递归，对重的结果字典也是多层嵌套的，我们的预期是每个资产的结果字典内每层检查都通过，所以对结果的断言也要用DFS算法逐层检查。

检查项设计为三个枚举值：
0：（初始默认值）路径存在，但格式内容不正确
-1：路径不存在
1： 路径存在，且格式内容正确

最终统计“所有叶子节点整数值的总和”是否与“叶子节点总数”相等。


```python
def sum_leaf_ints(obj):
    """
    递归遍历 obj，累加所有 int 类型的叶子节点值。
    支持嵌套 dict 和 list。
    """
    total = 0
    if isinstance(obj, dict):
        for value in obj.values():
            total += sum_leaf_ints(value)
    elif isinstance(obj, list):
        for item in obj:
            total += sum_leaf_ints(item)
    elif isinstance(obj, int):
        total += obj
    # 其他类型（str, float, None等）忽略
    return total

# 测试
check_res = {
    "data": {"data_chunk": -1},
    "meta": {
        "meta_chunk": 0,
        "meta_info_json": 1,
        "meta_stats_json": -1,
        "meta_tasks_parquet": -1,
    },
}

result = sum_leaf_ints(check_res)
print(result)  # 输出: -2

```

## 测试前移

### 测试自动化 + 持续测试 + 质量门禁前移
将以上自动化工程接入开发CI/CD，资产发布的卡点之后。实现了**测试自动化 + 持续测试 + 质量门禁前移**

1. 检查时机前移
- **原来**：版本转测后 → 测试人员人工检查 → 发现问题 → 退回开发 → 修复 → 重新转测
- **现在**：开发提交代码/触发流水线 → 自动检查脚本运行 → 立即反馈 → 开发当场修复

检查点从“转测后”前移到了“开发过程中”，这正是“左移”。

1. 人工检查 → 自动化持续检查
人工检查只能在特定节点进行，成本高、易遗漏、反馈慢。  
脚本接入 CI/CD 后，每次变更都能自动执行，实现**持续测试**，这是测试左移的重要支撑手段。

2. 测试团队的角色前移
测试团队不再只是“最后把关”，而是把质量规则、检查逻辑提前输出给开发流水线，参与到研发过程中，推动“质量内建”。

3. 降低修复成本
数据集目录结构问题如果在转测后发现，可能已经影响了后续训练、评测等流程；在 CI 阶段拦截，修复成本极低。

---


## ”售后“——维护关键点

技术创新推动流程优化，需要配合定义好”售后“维护工作，保证测试前移实践稳定高效运行

### 1. 规则要稳定、可维护
- 数据集目录规范可能随业务演进，检查脚本的规则（如 `repo-structure` 的 YAML）要版本化、可配置。
- 建议把规则文件与代码一起纳入版本管理。

### 2. 与开发团队达成共识
- 测试左移不是测试团队单方面“加卡点”，而是与开发协作。
- 要让开发理解：检查是为了尽早发现问题，而不是增加负担。
- 最好由开发在流水线中主动接入，测试团队提供规则和工具支持。

### 3. 反馈要快、要清晰
- CI 中检查失败时，报错信息要明确指出哪个目录、哪个文件不符合规范。
- 可以生成报告（如 `repo-structure report`）附在流水线产物中。

### 4. 避免“形式化左移”
- 如果脚本只是偶尔运行，或者失败后没人处理，那就只是“把人工检查换了个地方”，没有真正左移。
- 需要配合质量门禁：检查不通过则流水线失败，阻止合并或构建。

### 5. 环境一致性
- 如果数据集存放在 OBS 等云存储，CI 中要能访问（通过 SDK 或挂载），并处理好凭证管理。
- 本地检查与 CI 检查应使用同一套规则和工具版本。

### 6. 分层检查策略
- 快速检查（目录结构、命名规范）放在 CI 每次提交时运行。
- 更耗时的全量数据校验可以放在 nightly 构建或转测前。
- 这样既保证反馈速度，又不遗漏深度检查。

---