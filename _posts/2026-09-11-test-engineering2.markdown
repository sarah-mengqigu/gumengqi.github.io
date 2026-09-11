---
layout: post
title: "一种基于 DDT(Data-Driven-Test) 的可 AI 端到端辅助的 API 测试方案"
date: 2026-09-11
categories: Test-Engineering
---


传统API测试一个接口一个测试函数，参数写死在代码里，断言散落在各处。这种方式在规模扩大后会暴露三个问题：新增用例要改代码，接口调整要改代码，AI 基于此生成的测试代码也存在同样的问题。

这篇文章介绍一种不同的思路：**把测试逻辑固化为稳定的执行框架，把测试数据独立为声明式的 JSON 用例，让 AI 在数据层参与协作，而非在代码层自由发挥。**

这套方案带来三个直接变化：

- **新增用例 = 新增 JSON 文件**，不需要碰 Python 代码
- **接口变更只需调整数据**，执行框架保持稳定
- **AI 生成 JSON 比生成代码更可控**，严格 schema 校验能拦住大部分错误

我们以 [pytest、PyGithub、JSON 和 Allure 为例](https://github.com/sarah-mengqigu/pytest_mqplugin)，构建了一套声明式、低成本扩展、适合 AI 端到端辅助的 API 测试方案。核心不是某个工具或技巧，而是一种**"代码归代码，数据归数据，AI 归数据"**的分层协作模式。

如果你正在做 pytest 二次开发、SDK 测试平台建设，或者正在探索 AI 如何真正落地测试工程，这套方案或许能提供一些参考。

## 一、为什么需要新的 API 测试方案

传统 API 测试通常采用"一个接口一个测试函数"的写法：

```python
def test_search_repositories():
    result = github_client.search_repositories(
        "pytest",
        sort="stars",
        order="desc",
    )
    assert result.totalCount > 0
```

这种方式在接口数量较少时没有问题，但随着测试规模扩大，会逐渐暴露三个核心问题：

1. **代码与数据耦合**：新增测试数据需要修改 Python 代码，正向/负向/边界用例难以统一管理。
2. **重复劳动**：每个接口都重复编写调用逻辑、异常处理和报告逻辑。
3. **AI 难以协作**：AI 很难稳定生成符合规范的测试代码，测试失败后需人工重新阅读代码、复现请求和分析响应。

在 Data-Driven Testing（DDT）思想下，可以把"测试逻辑"和"测试数据"分离：

```text
固定测试逻辑 + 多个 JSON 用例 = 多个独立 API 测试
```

这样做不仅适合人工维护，也天然适合 AI 辅助生成、修改和分析测试用例。

需要强调的是，这里的"AI 端到端辅助"并不是让 AI 直接执行或替代测试，而是让 AI 参与完整链路：

```text
需求理解 → JSON 用例生成 → 用例评审 → 测试执行 → 失败分析 → 报告总结 → 用例修复建议
```

真正的网络请求、状态码断言和凭证管理仍然由确定性的 Python 代码完成。


## 二、整体架构

整个方案的数据流如下：

```text
┌──────────────────────────┐
│       JSON 用例           │
│  声明 request_name        │
│  parameters / expectation │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│      data_driven.py       │
│  读取、校验、转换 JSON     │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│         ApiCase           │
│   一个 Python 数据对象     │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│ pytest.mark.parametrize   │
│  一个 JSON 变成一个测试    │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│ test_github_api_cases.py  │
│        通用执行器          │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│   GithubClient.invoke()   │
│   动态调用 SDK 方法        │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│        PyGithub           │
│      真实 API 请求         │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│      ApiCallResult        │
│      code + data          │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│     assert + Allure       │
│  断言状态码并生成报告       │
└──────────────────────────┘
```

**架构核心思路**：JSON 描述"测什么" → pytest 组织执行 → Service 层封装 SDK 调用 → ApiCallResult 统一结果 → Allure 沉淀证据 → AI 参与生成与分析。


## 三、测试目录设计与 JSON 用例格式

### 3.1 测试目录设计

```text
tests/
├── api/
│   └── github/
│       ├── cases/
│       │   ├── search_repositories_200.json
│       │   └── search_repositories_empty_query_422.json
│       └── test_github_api_cases.py
```

### 3.2 JSON 用例格式

每个 JSON 文件代表一个独立测试用例：

```json
{
  "request_name": "search_repositories",
  "parameters": {
    "query": "pytest",
    "sort": "stars",
    "order": "desc"
  },
  "expectation": {
    "code": 200
  }
}
```

| 字段 | 含义 |
|------|------|
| `request_name` | 要调用的服务方法名称 |
| `parameters` | 传给服务方法的关键字参数 |
| `expectation` | 期望结果 |
| `expectation.code` | 期望 HTTP 状态码 |

文件名会成为 pytest case ID：

```text
search_repositories_200.json → test_github_api_case[search_repositories_200]
```

这种设计有几个优点：

1. 只要新增 JSON，就自动新增测试用例。
2. 不需要修改 Python 测试执行器。
3. 用例可以作为独立的测试资产进行评审和版本管理。
4. AI 生成 JSON 比生成完整 Python 测试代码更稳定。
5. JSON 易于进行 schema 校验和自动化检查。


## 四、加载 JSON 用例

项目中的 `data_driven.py` 负责把 JSON 转成 Python 对象。

### 4.1 定义数据结构

```python
@dataclass(frozen=True, slots=True)  # frozen 保证不可变，slots 减少内存占用
class ApiExpectation:
    code: int

@dataclass(frozen=True, slots=True)
class ApiCase:
    case_id: str          # 来自 JSON 文件名
    request_name: str     # SDK 方法名称
    parameters: dict[str, Any]
    expectation: ApiExpectation
    source: Path          # 来源文件，便于定位问题
```

### 4.2 扫描 JSON 文件

```python
directory = Path(case_directory)
case_paths = sorted(directory.glob("*.json"))
```

当前实现只扫描 `cases/*.json`，不会扫描 `cases/子目录/*.json`。如果未来用例数量很多，可以改成 `directory.rglob("*.json")`，从而递归发现所有层级的 JSON 用例。

### 4.3 JSON 转成对象

```python
raw_case = json.loads(
    case_path.read_text(encoding="utf-8")
)
```

如果 JSON 语法错误，会捕获异常：

```python
except json.JSONDecodeError as error:
    raise ValueError(...) from error
```

`from error` 会保留原始异常链，方便排查。

### 4.4 严格字段校验

加载器只允许三个顶层字段：

```python
_reject_unknown_keys(
    case,
    {"request_name", "parameters", "expectation"},
    f"case {case_path.name}",
)
```

这一点对 AI 生成用例尤其重要。AI 可能会产生拼写错误或不存在的字段，严格 schema 可以防止错误字段被静默忽略，从而避免"测试看起来执行了，但实际上没有测试目标接口"的情况。

### 4.5 字段类型校验

```python
if type(code) is not int:
    raise ValueError(...)
```


## 五、通用数据驱动执行器

`test_github_api_cases.py` 是测试服务执行器。它不针对某一个接口写死测试，而是执行对象服务的所有 API 用例。

### 5.1 收集全部用例

```python
API_CASES = load_api_cases(
    Path(__file__).parent / "cases"
)
```

### 5.2 pytest 参数化

```python
@pytest.mark.parametrize(
    "api_case",
    API_CASES,
    ids=lambda case: case.case_id,
)
```

这是 DDT 的核心。如果 `API_CASES` 中有两个用例：

```text
search_repositories_200
search_repositories_empty_query_422
```

pytest 会自动生成：

```text
test_github_api_case[search_repositories_200]
test_github_api_case[search_repositories_empty_query_422]
```

`ids=lambda case: case.case_id` 决定测试 ID。

### 5.3 fixture 注入 client

```python
def test_github_api_case(
    github_client: GithubClient,
    api_case: ApiCase,
) -> None:
```

### 5.4 执行接口请求

```python
result = github_client.invoke(
    api_case.request_name,
    api_case.parameters,
)
```

测试代码不需要知道要调用哪个具体方法——JSON 中的 `request_name` 动态决定了调用的 SDK 方法。

### 5.5 校验状态码

```python
assert result.code == api_case.expectation.code
```


## 六、服务封装层

`github.py` 负责真正调用 PyGithub。

### 6.1 动态调用 SDK 方法

```python
method = getattr(self, request_name)
```

这样 JSON 中的 `request_name` 就能动态控制调用哪个方法。如果当前类没有这个方法，还会进入：

```python
def __getattr__(self, name: str) -> Any:
    return getattr(self._github_client, name)
```

从而把请求转发给底层 PyGithub 客户端。

### 6.2 参数字典展开

```python
data = method(**dict(parameters))
```

### 6.3 统一成功结果

```python
return ApiCallResult(
    code=200,
    data=data,
)
```

对于正常的 GET API，如果 SDK 没有抛出异常，就认为请求成功，统一返回 200。如果以后测试 POST 创建接口、期望 201，或者 DELETE 接口、期望 204，需要扩展这一部分。

### 6.4 统一异常结果

```python
except GithubException as error:
    return ApiCallResult(
        code=error.status or 500,
        data=error.data,
    )
```

`GithubException` 是 PyGithub 对 HTTP 错误的封装。如果服务端没有提供状态码，就使用 500 作为兜底。


## 七、两个关键工程细节

### 7.1 懒加载：确保请求真正发出

PyGithub 有很多懒加载对象。例如 `github_client.get_repo("owner/repo")` 可能只是创建一个对象，并没有真正访问 GitHub——只有访问属性时才发起请求。

如果测试执行器只调用 `get_repo()` 方法，就可能在还没有发请求的情况下提前认为测试成功。因此增加了：

```python
def _materialize_response(data: Any) -> None:
    if isinstance(data, PaginatedList):
        _ = data.totalCount
    elif isinstance(data, CompletableGithubObject):
        _ = data.raw_data
```

作用分别是：

- `PaginatedList`：读取 `totalCount`，触发分页接口请求。
- `CompletableGithubObject`：读取 `raw_data`，触发对象补全请求。

这样 422、404、403 等异常才能真正被捕获。

### 7.2 负向用例：绕过 SDK 的 response validation

一些 SDK 构建不完整的服务，在发送错误参数的 SDK 调用时不会做 response validation 检验，可以直接沿用请求返回 200 的机制测试返回 4xx 的情况。

但是有些服务（如 GitHub 发布的 SDK）有 response validation 检验。要测试接口 4XX，就需要通过在 `invoke()` 时绕过 SDK，把请求直接作为 request 发送到服务。

JSON：

```json
{
  "request_name": "search_repositories",
  "parameters": {
    "query": "",
    "sort": "stars",
    "order": "desc"
  },
  "expectation": {
    "code": 422
  }
}
```

执行过程：

1. 加载器把 JSON 转换成 `ApiCase`，pytest 生成 `test_github_api_case[search_repositories_empty_query_422]`。
2. `github_client.invoke()` 调用 `search_repositories`，空 query 走特殊分支，直接构造请求。
3. `_materialize_response()` 触发真实 GitHub 请求，返回 422。
4. PyGithub 抛出 `GithubException`，`invoke()` 转换成 `ApiCallResult(code=422)`。


## 八、AI 如何参与端到端流程

这套方案之所以适合 AI，是因为它把不稳定、复杂的 Python 实现固定下来，把易变的测试数据独立成 JSON。配套安装 skill：`api-sdk-tester`，让 AI Agent 按照 API 测试规范约束，进行全面的测试覆盖。

| 阶段 | AI 能做什么 | 人类保留什么 |
|------|-------------|--------------|
| 用例生成 | 根据接口文档/SDK 生成 JSON 用例 | 确认业务预期 |
| 用例评审 | 检查字段完整性、参数合法性 | 审批负向用例状态码 |
| 失败分析 | 根据 Allure 结果判断失败类型 | 判断是否属于缺陷 |
| 修复建议 | 提出 JSON 修改建议 | 决定是否接受修改 |
| 回归决策 | 标记高频失败用例 | 决定是否纳入正式回归集 |

**严格 schema 阻止错误生成**：如果 AI 生成了 `"reuquest_name"`（拼写错误），加载器会直接报错 `Unknown field(s): reuquest_name`。这比生成完整 Python 测试代码更安全，因为错误范围被限制在数据结构中。


## 九、扩展方向与总结

### 9.1 扩展方向

- **数据能力增强**：递归扫描 JSON、支持变量替换和环境配置、引入 JSON Schema/Pydantic 校验
- **测试场景扩展**：支持接口依赖和前后置动作、区分 Mock 与 Live 测试、支持响应字段校验
- **工程化集成**：为不同 SDK 建立独立服务实现、接入 CI/CD 和自动分析流程

### 9.2 总结

```text
用固定代码承载执行逻辑
用 JSON 描述测试数据
用 pytest 组织测试
用 Service 封装 SDK
用 ApiCallResult 统一结果
用 Allure 沉淀执行证据
用 AI 辅助生成、分析和总结
```

它解决的不只是"如何写一个 API 测试"，而是：

> 如何让 API 测试更容易扩展、更容易维护，并且能够稳定地与 AI 协作。

对于 pytest 二次开发、SDK 测试平台和未来的多服务 API 测试体系来说，这是一种结构清晰、成本可控、可逐步演进的实现方式。