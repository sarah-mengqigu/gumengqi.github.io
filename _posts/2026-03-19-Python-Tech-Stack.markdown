
# Python虚拟机
Python 虚拟机（PVM）模拟的是一台「基于栈的虚拟计算机」。
它模拟了一套：
1. 一个栈（stack）
2. 一套字节码指令集（比如 LOAD_NAME、CALL_FUNCTION）
3. 一套运行时环境（对象、命名空间、异常、GIL 等）

比如一行：
2 + 3
字节码和虚拟机行为是：
LOAD_CONST     2    → 把 2 压入栈
LOAD_CONST     3    → 把 3 压入栈
BINARY_ADD           → 从栈顶弹出两个数相加，结果压回栈
栈的变化：
[] → [2] → [2,3] → [5]

它是一个纯逻辑层面的虚拟机，只在代码里存在。

# Python解释器
xxx.py文件 是纯文本，Python 做的事只有四步：
**读文本 → 编译成字节码 → 丢给虚拟机跑 → 执行**

**1. 执行py文件**
python test.py
发生的第一件事：

Python 解释器启动
python 本身是一个可执行程序（C 语言写的）。
它一运行，就做第一件事：
打开 test.py，按文本读进来
就像记事本打开一样，只是：

• 它不认中文、不认注释
• 它只认 Python 语法规则

读到的内容类似：
['def', 'hello', '(', ')', ':', 'print', '(', '"hi"', ')']
这一步叫 词法分析 + 语法分析。

**2. 编译成字节码**

Python 不是直接跑文本，也不是直接编译成机器码。
它会把代码翻译成一套中间指令，叫：
字节码 bytecode

比如：
print("hello")
会变成类似：
LOAD_NAME     print
LOAD_CONST    'hello'
CALL_FUNCTION 1
POP_TOP
这就是字节码。

同时会生成：
\_\_pycache\_\_/test.cpython-3xx.pyc
这是字节码缓存文件，下次直接读，不用重新编译。

**3. 虚拟机（PVM）执行字节码文件**

你电脑的 CPU 不认 Python 字节码。
但 Python 自带一个虚拟机（PVM），它认识字节码。

PVM 本质就是一个超大循环：
while 还有指令:
    取下一条字节码
    执行它
比如：

• LOAD_NAME print → 找到 print 函数
• LOAD_CONST 'hello' → 把字符串放进栈
• CALL_FUNCTION 1 → 调用函数




**几个你可能疑惑的点**

① 为什么不用编译成 exe 也能跑？
因为 Python 自带虚拟机（PVM），
它相当于一个跨平台的翻译官，在每个系统上把字节码翻译成系统能懂的指令。

② 为什么双击 .py 也能跑？
操作系统看到 .py，
去找注册表/配置里绑定的 python.exe，
本质还是：
python test.py
③ 为什么 Python 比 C 慢？

因为：
• C → 直接编译成机器码，CPU 直接跑
• Python → 文本 → 字节码 → 虚拟机模拟执行 → 再到系统
多了两层翻译。


# Python包管理

## pip 包管理
当执行命令
pip install requests
pip 依次干这 5 件事：

1. 去 PyPI 查 requests 最新版
2. 读取 requests 的依赖：比如 urllib3、certifi、charset-normalizer 等
3. 递归查所有依赖的依赖（依赖树）
4. 下载所有需要的包
5. 丢到 site-packages
6. 把包名+版本写到 site-packages/*.dist-info 里


依赖扁平化管理
① site-packages 目录
所有包实际都放在这里：
lib/pythonX.X/site-packages/
② .dist-info 元数据文件夹

每个包都会带一个，比如：
requests-2.31.0.dist-info/
里面有：
• METADATA：依赖要求（如 urllib3>=1.21.1,<3）
• INSTALLER：记录是 pip 安装的
• RECORD：安装了哪些文件
pip 就是读这些文件来知道当前环境有什么。

缺陷：
• 多个包依赖同一个库不同版本时候， pip 不会智能解决，只会装最后那个

解决方案：用虚环境隔离不同项目的依赖
A 依赖 requests\=\=2.20
B 依赖 requests\=\=2.30
你先装 A → requests 2.20
再装 B → 直接升级成 2.30
A 可能就炸了。

👉 pip 不做依赖冲突保障，这就是为什么要 venv + requirements.txt


pip 如何保存依赖？
pip freeze > requirements.txt
完全是快照，不是智能解析。

pip uninstall requests
• 读 .dist-info/RECORD
• 把里面列的文件一个个删掉
• 删掉 .dist-info 文件夹

⚠️ 但：
pip 不会自动卸载没用的依赖（孤儿包）
比如你卸载 requests，urllib3 还留在那。


## setup.py

### 「安装说明书 + 打包脚本」

1. 让别人可以用 pip install . 安装你的包
2. 把你的代码打包成 .tar.gz / .whl，发到 PyPI（也就是 pip 官方源）

它告诉 Python：
• 这个包叫什么名字
• 版本是多少
• 包含哪些模块/文件夹
• 依赖哪些包
• 有哪些可执行命令（比如你输 django 就能跑）

2. 最简单的 setup.py 长这样

```python
from setuptools import setup

setup(
    name="my_tool",          # 包名
    version="0.0.1",          # 版本
    packages=["my_tool"],     # 要包含的文件夹
    install_requires=["requests>=2.25"],  # 依赖
    entry_points={
        "console_scripts": [
            "mycmd=my_tool.main:run"  # 命令行指令
        ]
    }
)

```
3. 用它能做什么？

（1）本地安装
pip install .
pip 就会读 setup.py，然后：

• 把你的代码复制到 Python 的 site-packages
• 自动安装 install_requires 里的依赖
• 有 entry_points 就会生成系统命令

（2）打包
> python setup.py sdist
> \# 或者现代方式
> pip install build
> python -m build

会生成：

• dist/xxx-0.0.1.tar.gz

别人就可以：
pip install dist/xxx-0.0.1.tar.gz

（3）上传到 PyPI（让全世界 pip 安装）
twine upload dist/*
然后别人就能：
pip install my_tool



### setup.py 工作原理

1. setup.py 基于 setuptools 这个库
2. 你写的配置 → 传给 setup() 函数
3. setuptools 负责：

◦ 复制文件
◦ 处理依赖
◦ 生成可执行命令
◦ 打包

本质：
setup.py = 包的安装+打包配置文件


5. 和 requirements.txt 区别（必懂）

• requirements.txt：我这个项目需要装啥
• setup.py：我这个包是谁、要啥、怎么装

现代项目已经慢慢用 pyproject.toml 代替 setup.py 了，但原理完全一样。


# Python虚环境
核心一句话：venv 本质就是「复制/链接一份 Python 解释器 + 改包查找路径」，从根源上做到环境隔离。

当你运行：
python -m venv myenv
它只会干 3 件事：
1. 在 myenv 里创建一套迷你 Python 目录结构
◦ myenv/bin/
◦ myenv/Scripts/
◦ myenv/lib/pythonx.x/site-packages
2. 复制或软链接 python、pip 等可执行文件
不是完整复制整个 Python，只是** lightweight 副本/链接**，指向系统 Python，但运行时会认自己的目录。
3. 生成关键文件：pyvenv.cfg
里面写死：
home = /usr/bin/python3.x  # 原始Python位置
include-system-site-packages = false  # 关键！不读系统包


venv 实现隔离靠三点：
1. 每个环境一套独立 site-packages
2. 独立的 python/pip 入口
3. 启动时只加载自己的包路径，不加载系统包

它不是虚拟机、不是容器、不是沙箱，
就是Python 级别的路径隔离。


# Conda Poetry
作为 Python 使用者，你可能会发现直接使用 pip 和 python -m venv 有时不足以应对复杂的项目需求——比如需要管理不同 Python 版本、处理非 Python 依赖的库（如科学计算）、确保团队成员依赖完全一致、或者准备发布自己的库。这时候就需要更强大的工具。

## 一、Conda & Miniconda：环境与包的一站式管理

1. 什么是 Conda？
Conda 是一个开源的包管理系统和环境管理系统，它并非 Python 专属，但广泛应用于 Python 数据科学和机器学习领域。它可以：

· 创建隔离的 Python 环境（类似 venv）。
· 安装不仅包括 Python 包，还包括任何语言的软件（如 C 库、R 包等）。
· 自动处理复杂的依赖关系，尤其是非 Python 依赖（如 CUDA、BLAS 等）。

2. Miniconda 与 Anaconda 的关系

· Anaconda：是一个大型发行版，预装了 conda 和 150 多个科学计算包（如 numpy、pandas、jupyter），适合新手或不想手动安装的人，但体积很大（约 3GB）。
· Miniconda：是 Anaconda 的最小启动版，只包含 conda 和它的依赖项，以及 Python。你可以用 conda 命令按需安装需要的包，体积小、灵活。通常推荐使用 Miniconda。

3. 安装 Miniconda
· 访问 Miniconda 官网 下载对应系统的安装程序。
· 安装过程中可以选择是否将 conda 加入 PATH（建议选是）。
· 安装后打开终端（或 Anaconda Prompt），输入 conda --version 验证。

4. 常用命令

环境管理
```bash
# 创建一个名为 myenv 的新环境，指定 Python 版本为 3.9
conda create --name myenv python=3.9

# 创建环境的同时安装一些包
conda create --name myenv python=3.9 numpy pandas

# 激活环境（Linux/macOS）
conda activate myenv
# Windows 下命令相同

# 退出当前环境
conda deactivate

# 列出所有环境
conda env list

# 删除环境（及其中所有包）
conda env remove --name myenv
```

包管理

```bash
# 安装包（默认从 conda 默认仓库下载）
conda install numpy

# 指定安装版本
conda install numpy=1.21

# 从不同的渠道（如 conda-forge）安装
conda install -c conda-forge pandas

# 卸载包
conda remove numpy

# 列出当前环境所有已安装包
conda list

# 更新包
conda update numpy

# 更新 conda 自身
conda update conda
```

导出与重建环境

```bash
# 导出当前环境的所有包信息到文件（包含具体版本和来源）
conda env export > environment.yaml

# 从 environment.yaml 创建一模一样的环境
conda env create -f environment.yaml
```

结合 pip 使用

Conda 环境中也可以使用 pip 安装一些 conda 仓库没有的包，但要注意混用时可能产生依赖冲突：

```bash
conda activate myenv
pip install some-package
```

---

## 二、Poetry：现代 Python 依赖管理与打包工具

Poetry 是一个专注于 Python 项目的工具，它统一了依赖管理、打包和发布。与传统的 pip + requirements.txt + setup.py 不同，Poetry 使用一个 pyproject.toml 文件来管理项目元数据和依赖，并提供锁文件确保确定性安装。

1. 安装 Poetry

推荐使用官方安装脚本（macOS/Linux）：

```bash
curl -sSL https://install.python-poetry.org | python3 -
```

Windows 可以在 PowerShell 中运行：

```powershell
(Invoke-WebRequest -Uri https://install.python-poetry.org -UseBasicParsing).Content | python -
```

安装后确保 poetry 的 bin 目录在 PATH 中（安装脚本会提示）。

2. 创建新项目

```bash
# 创建一个名为 myproject 的项目，包含基本目录结构
poetry new myproject
cd myproject
```

目录结构：

```
myproject/
├── pyproject.toml
├── README.md
├── myproject/
│   └── __init__.py
└── tests/
    └── __init__.py
```

pyproject.toml 是核心配置文件，里面包含项目名称、版本、作者以及依赖声明。

3. 在现有项目中初始化

如果你已有项目，可以在项目根目录运行：

```bash
poetry init
```

它会交互式地引导你填写信息并生成 pyproject.toml。

4. 添加依赖

```bash
# 添加生产依赖
poetry add requests

# 添加开发依赖（如 pytest）
poetry add --dev pytest

# 指定版本
poetry add django@^3.2
```

执行后，Poetry 会自动更新 pyproject.toml 和生成 poetry.lock 文件（锁定精确版本）。后续其他开发者只需运行 poetry install 就会安装 poetry.lock 中指定的版本，保证环境一致。

5. 安装所有依赖

```bash
# 根据 pyproject.toml 和 poetry.lock 安装所有依赖（包括开发依赖）
poetry install

# 只安装生产依赖（不安装 dev 依赖）
poetry install --no-dev
```

6. 进入虚拟环境

Poetry 会自动为每个项目创建独立的虚拟环境（默认在 {cache-dir}/virtualenvs 下）。要进入该环境运行命令，有两种方式：

```bash
# 方式一：使用 poetry run 执行命令
poetry run python myscript.py

# 方式二：激活虚拟环境（类似 source venv/bin/activate）
poetry shell
```

7. 查看依赖树

```bash
poetry show        # 列出所有依赖
poetry show --tree # 以树形展示依赖关系
```

8. 更新依赖

```bash
# 更新所有依赖到符合 pyproject.toml 约束的最新版本
poetry update

# 更新指定包
poetry update requests
```

9. 构建与发布

如果你要打包项目并发布到 PyPI：

```bash
# 构建 wheel 和源码包
poetry build

# 发布（需要先配置 PyPI 认证）
poetry publish
```

---

## 三、对比与选择建议

特性 Conda / Miniconda Poetry
主要用途 科学计算、数据科学、多语言依赖管理 纯 Python 项目开发、库打包与发布
环境隔离 支持（跨语言） 支持（Python 虚拟环境）
包来源 默认从 conda 仓库（anaconda.org） 默认从 PyPI
非 Python 包 原生支持（如 C 库、R 包） 不支持（需通过系统包或 pip 处理）
依赖解析 强大，能处理复杂二进制依赖 专注于 Python 依赖，解析严格
锁文件 environment.yaml（不锁定精确版本） poetry.lock（精确锁定版本）
项目配置 无统一标准，常结合 requirements.txt 使用 pyproject.toml（PEP 621 标准）
发布到 PyPI 不直接支持（需额外用 setuptools） 原生支持（poetry build/publish）

什么时候用 Conda / Miniconda？

· 项目涉及科学计算库（如 numpy、pandas、scikit-learn），这些库常有非 Python 依赖（如 BLAS、LAPACK），用 conda 安装可以自动处理。
· 需要使用不同 Python 版本，且希望系统层面隔离（conda 可以安装任何版本的 Python）。
· 需要管理非 Python 工具（如 R、Java 库）。
· 团队中数据科学家多用 Jupyter，conda 能方便地创建内核。

什么时候用 Poetry？

· 开发一个标准的 Python 库或 Web 应用，依赖都来自 PyPI。
· 希望有现代、清晰的依赖管理（锁文件确保环境一致，避免“在我机器上能跑”）。
· 需要发布到 PyPI，希望简化打包流程。
· 喜欢 pyproject.toml 的简洁和标准化。

---

## 四、实际工作流示例

场景：数据分析项目（用 Miniconda）

```bash
# 1. 创建环境，指定 Python 3.9，并安装常用包
conda create --name mydata python=3.9 numpy pandas matplotlib jupyter

# 2. 激活环境
conda activate mydata

# 3. 安装额外的包（如 scikit-learn）
conda install scikit-learn

# 4. 导出环境
conda env export > environment.yaml

# 5. 别人复现环境
conda env create -f environment.yaml
```

场景：Web 应用开发（用 Poetry）

```bash
# 1. 初始化项目
poetry new mywebapp
cd mywebapp

# 2. 添加依赖
poetry add fastapi uvicorn

# 3. 添加开发依赖
poetry add --dev pytest pytest-cov

# 4. 编写代码，运行
poetry run uvicorn mywebapp.main:app --reload

# 5. 测试
poetry run pytest

# 6. 生成 requirements.txt（供部署用，如果需要）
poetry export -f requirements.txt --output requirements.txt
```





