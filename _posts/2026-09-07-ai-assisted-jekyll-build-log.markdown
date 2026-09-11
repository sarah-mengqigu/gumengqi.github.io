---
layout: post
title: "Github Pages诞生日志：见证AI辅助的两个能力阶段"
date: 2026-09-09
categories: log
excerpt: >-  
    上半年还在对话式人工部署，下半年用Agent已经可以搞定一切了！
---

这个博客是一个 Jekyll 静态站点，托管在 GitHub Pages 上。它的搭建过程本身就是一段“AI 辅助工程实践”的样本：**第一次（2026 年 3 月）由 Gemini 以对话形式指导我手动部署；第二次（2026 年 9 月）由 Codex（接入 DeepSeek V4 Flash）自动诊断并修复环境**。两次面对的是同一个根因——非常新的 macOS 版本带来的 Ruby 生态兼容性问题，但处理方式、路径和最终结果完全不同。

本文按时间线展开，尽量保留关键命令、报错原文和解决思路，方便日后复用。

## 1. 背景与硬件环境

| 项目 | 配置 |
|---|---|
| 设备 | MacBook Air（M2，8 核 / 8 GB 统一内存） |
| 系统 | macOS 26.3（Tahoe，arm64） |
| 原始环境 | 系统自带 Ruby 2.6.10（Apple 遗留，早已 EOL）、无 rbenv/rvm Ruby、Homebrew 版本过旧 |


## 2. 第一次构建（2026-03）：Gemini 指导，Docker 兜底

### 2.1 过程

3 月我创建了本项目，希望通过对话式 AI（Gemini）指导，在 Mac 上手动部署 Jekyll 并发布到 GitHub Pages。过程大致是：

1. 初始化 Jekyll 站点骨架（Gemfile、_config.yml、_posts 等）；
2. 在**本机**执行 `bundle install` 安装 Jekyll —— 失败；
3. 反复排查 Ruby 版本、原生 gem 编译问题 —— 始终无法绕开；
4. 最终放弃本机 Ruby 路线，改用 **Docker 容器**运行 Jekyll 官方镜像 `jekyll/jekyll:4.2.2`，站点才正常跑起来；
5. 通过终端推送文章到 GitHub，完成 Pages 部署。

### 2.2 根因

项目开发前，很不明智地升级了mac操作系统版本，导致安装 ruby 的时候，因为操作系统版本过高，ruby 没有兼容最新的版本号

- Apple 在 macOS 中只随系统提供 Ruby 2.6.10，且不再升级；
- Jekyll 4.2.x 及它的依赖链（`eventmachine`、`sassc`、`ffi` 等原生扩展）在 Ruby 2.6 + 新版 SDK/clang 下无法编译；
- 系统 gem 目录位于 SIP 保护区，装 gem 还涉及权限问题。

### 2.3 兜底方案：Docker

`docker-compose.yml` 的核心内容：

```yaml
services:
  jekyll:
    image: jekyll/jekyll:4.2.2
    platform: linux/amd64  # 解决架构不匹配警告
    volumes:
      - .:/srv/jekyll
    ports:
      - "4000:4000"
    command: jekyll serve --watch --force_polling --livereload --trace
    environment:
      - JEKYLL_ENV=development
```

官方镜像内置了正确的 Ruby 版本与依赖，等于把“环境问题”整体打包隔离。`Gemfile.lock` 里至今还留着 `x86_64-linux-musl` 平台标记——这正是当时在 Linux（alpine）容器内生成锁文件的痕迹。

### 2.4 小结

- ✅ 站点能跑、能部署；
- ❌ 本机 Ruby 环境始终未解决，日常预览被 Docker 绑定，主题定制与调试都不太顺手；
- 📌 这个“历史遗留问题”留到了 9 月。

## 3. 第二次构建（2026-09）：Codex + DeepSeek V4 Flash，根因修复

### 3.1 目标

9 月重启项目，目标明确：

1. **摆脱 Docker**，让 Jekyll 直接在 macOS 26.3 本机跑通；
2. **套用 Hydejack 主题**（开源 Jekyll 模板）；

这次由 Codex（接入 DeepSeek V4 Flash 模型）以“agent 自动执行”的方式完成诊断与修复，整个过程可以拆成一条清晰的“问题-修复链”。

### 3.2 环境审计

开工前先摸清家底（全部为实测）：

| 项目 | 实测结果 |
|---|---|
| 系统 Ruby | 仅 `/usr/bin/ruby` 2.6.10（EOL） |
| rvm | 已安装（1.29.12），但 `~/.rvm/rubies` 为空 —— 一个 Ruby 都没装 |
| Homebrew | 4.1.5（2023 年的老版本），任何命令都报 `unknown or unsupported macOS version: "26.3"` |
| Node | 未安装（先装了官方 Node v24，用于另一个 CLI 工具） |
| Docker | 已安装（上次方案的依赖） |

### 3.3 复现问题（用证据说话）

先在 Ruby 2.6.10 下对项目跑一次 `bundle install`，装不上：

```bash
bundle _2.3.25_ install --dry-run   # 2.3 不支持 --dry-run，改为真实安装到临时目录
```

结果非常典型：

```text
An error occurred while installing eventmachine (1.2.7), and Bundler cannot
continue.
In Gemfile:
  jekyll-admin was resolved to 0.11.1, which depends on    # ← 注意：被降级了
```

结论：
1. **解析阶段就漂移**：锁定文件里 `jekyll-admin 0.12.0` 在 Ruby 2.6 下不被接受，被降级到 0.11.1；
2. **编译阶段硬失败**：`eventmachine 1.2.7`（2018 年的老 C++ 扩展）无法在 2026 年的 SDK/clang 下编译。

**必须装现代 Ruby（3.x）**。

### 3.4 修复链（按顺序）

#### 3.4.1 rvm 编译安装 Ruby 3.3.12

第一步就遇到两个坑：

- `rvm install 3.3` 会先执行 `requirements_osx_brew_update_system`，即**调用 Homebrew 装依赖**——但 Homebrew 已坏，rvm 卡死。解法：`--autolibs=disabled` 跳过。
- rvm 把别名 “3.3” 解析到一个不存在的下载地址（404）。解法：**指定精确版本**。

```bash
rvm install 3.3.12 --autolibs=disabled
rvm use 3.3.12
ruby -v   # ruby 3.3.12 (2026-07-16 revision ...) [arm64-darwin25]
```

编译约几分钟后成功。但马上发现新 Ruby 是“残缺”的——`openssl`、`psych` 扩展缺失：

```text
cannot load such file -- openssl (LoadError)
cannot load such file -- psych (LoadError)
```

原因：macOS 新版 SDK **不再自带 OpenSSL/libyaml 头文件**，源码编译 Ruby 时这两个扩展被自动跳过。而 RubyGems/Bundler 走 HTTPS、解析 YAML 都依赖它们。

#### 3.4.2 升级 Homebrew（治本）——  **这一步是human in the loop 强行插入的动作**

Homebrew 报错 `unknown or unsupported macOS version: "26.3"` 的根因是版本太老（4.1.5 不支持 macOS 26）。麻烦的是：**`brew update` 自己也起不来**——版本检查发生在 Homebrew Ruby 启动阶段，任何命令都会先崩。

升级后 Homebrew 恢复正常，于是：

```bash
brew install libyaml          # 为 Ruby 补 psych
# 用已装的 openssl@3 为 Ruby 补 openssl
```

#### 3.4.3 重编缺失的 Ruby 扩展

进入 Ruby 源码里对应的扩展目录，用 `extconf.rb` 重新生成 Makefile 并编译安装：

```bash
# psych（需要 libyaml）
cd ~/.rvm/src/ruby-3.3.12/ext/psych
ruby extconf.rb --with-libyaml-dir=/opt/homebrew/opt/libyaml && make && make install

# openssl（需要 openssl@3）
cd ~/.rvm/src/ruby-3.3.12/ext/openssl
ruby extconf.rb --with-openssl-include=/opt/homebrew/opt/openssl@3/include \
                --with-openssl-lib=/opt/homebrew/opt/openssl@3/lib && make && make install

ruby -ropenssl -e 'puts OpenSSL::OPENSSL_VERSION'   # OpenSSL 3.1.2 ✅
ruby -rpsych -e 'puts Psych::VERSION'               # psych 5.1.2 ✅
```

#### 3.4.4 修复原生 gem 编译：`CXX=false`

重新 `bundle install`，`eventmachine` 又失败，而且失败得“很安静”——`make` 直接返回 Error 1 却没有编译错误。手动执行编译命令后发现真相：

```text
c++ ... -o binder.o -c binder.cpp   # 实际执行的是 false ...?
clang++: error: no such file or directory: 'false -I...'
```

原来 Makefile 里的编译器变量是 `CXX = false`。查 `rbconfig`：

```text
RbConfig::CONFIG["CXX"]  # => "false"
```

**根因**：rvm 编译 Ruby 时只传了 `CC=gcc`，没有探测到 C++ 编译器，导致 `CXX` 落成了 `false`，所有需要 C++ 的 gem 都会拿到一个假的编译器。

修复：直接改 `rbconfig.rb` 中对应条目：

```bash
ruby -e 'pth=ARGV[0]; s=File.read(pth);
         s.sub!(/(CONFIG\["CXX"\]\s*=\s*)"false"/, %q{\1"/usr/bin/c++"});
         File.write(pth,s)' \
  ~/.rvm/rubies/ruby-3.3.12/lib/ruby/3.3.0/arm64-darwin25/rbconfig.rb
```

#### 3.4.5 修复原生 gem 编译：libc++ 头文件是“空壳”

继续 `bundle install`，`sassc`（老 libsass，纯 C++）报：

```text
./libsass/src/memory/allocator.hpp:8:10: fatal error: 'vector' file not found
```

排查过程很有意思：`clang++ -v` 明明把 `.../CommandLineTools/usr/include/c++/v1` 列在搜索路径里，`<vector>` 却找不到。逐一排查后发现：

- CLT 的 `usr/include/c++/v1` **只有 11 个文件**（空壳）；
- 完整的 189 个标准库头文件在 **SDK** 里：`/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/v1`；
- clang 默认没把 SDK 里的这份加进 C++ 搜索路径。

解法：给相关 gem 的编译显式加上 SDK 头文件路径：

```bash
bundle config set build.sassc        "--with-cxxflags=-I/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/v1"
bundle config set build.eventmachine "--with-cxxflags=-I/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/c++/v1"
```

（这些参数会写入项目 `.bundle/config`，属于“机器相关”配置，已加入 `.gitignore`。）

#### 3.4.6 收网：bundle install + 构建验证

```bash
bundle install          # Bundle complete! 55 gems ✅
bundle exec jekyll build
bundle exec jekyll serve
```

Jekyll 终于在 macOS 26.3 本机原生跑起来了——**Docker 正式“退役”**。

### 3.5 套用 Hydejack 主题

环境跑通后，把默认主题换成开源的 **Hydejack**（`jekyll-theme-hydejack`，GPL-3.0）。

关键改动：

| 文件 | 操作 |
|---|---|
| `Gemfile` | 移除 minima，加入 `jekyll-theme-hydejack ~> 9.1` + 插件组 |
| `_config.yml` | 清理历史遗留的重复键（两个 title、两个 theme），改为 Hydejack 配置 |
| `index.html` | 首页，`layout: blog` |
| `posts.md` | 归档页，`layout: list` |
| `about.md` | 关于页，`layout: about` |
| `404.md` | 404 页，`layout: not-found` |
| `_data/authors.yml` | 作者数据 |

启动jekyll：
```bash
bundle install          # 55 gems
bundle exec jekyll build
bundle exec jekyll serve
# /、/about/、/posts/、/feed.xml 全部 HTTP 200
```

## 4. 两次构建的对比

| 维度 | 2026-03（Gemini） | 2026-09（Codex + DeepSeek V4 Flash） |
|---|---|---|
| 执行方式 | AI 对话指导，手动执行 | AI agent 自动诊断与执行 |
| 环境策略 | 绕开本机（Docker 容器） | 根治本机（装 Ruby 3.3.12 + 修编译链） |
| 主题 | minima | Hydejack |
| 本机 Ruby | 仍不可用 | macOS 26.3 原生可用 |
| 依赖 Docker | 是 | 否 |
| 遗留问题 | Ruby 环境未解决 | 仅剩作者信息/baseurl 等配置项 |

## 5. 可复用的经验（排坑清单）

1. **永远别用 macOS 系统 Ruby 跑现代 Jekyll**——它是 2.6，早已 EOL。用 rvm/rbenv 装 3.x。
2. **Homebrew 过旧会“自杀式”无法更新**：报 `unknown or unsupported macOS version` 时，直接用 `git fetch + reset --hard` 升级本体（记得 master→main 迁移），国内可用 USTC 镜像。
3. **源码编译 Ruby 后要检查扩展**：`openssl`/`psych` 常因 SDK 缺头文件被跳过；用 Homebrew 装好依赖后进 `ext/` 目录重编即可，不用整颗重编 Ruby。
4. **rvm 编的 Ruby 若 `CXX=false`**：所有 C++ 原生 gem 都会编译失败且报错隐蔽；改 `rbconfig` 或重编 Ruby。
5. **CLT 的 libc++ 头文件可能是空壳**：报 `<vector>/<iostream> not found` 时，用 `-I .../SDKs/MacOSX.sdk/usr/include/c++/v1` 指向 SDK 里的完整头文件。
6. **GitHub 推送**：HTTPS 需要 PAT（密码认证已被禁）；更省心的是把 SSH 公钥加入 GitHub 后用 `git@github.com` remote。
7. **Docker 是好兜底，但不是终点**：环境根治后，本地开发、主题定制、调试的体验完全是两回事。



---

*本博客本身就是这个“第二次构建”过程的产物。*
