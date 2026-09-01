# Termux-bai — 二次构建完整架构

## 1. 项目定位

Termux-bai 是基于 Termux 官方工程架构进行长期维护的二次构建项目。

当前 App 基线固定为 **Termux `v0.119.0-beta.3`**。该版本是 Termux-bai 第一阶段的官方上游基线，不代表项目以后不能升级。

核心原则：

- 保留 Termux 官方的核心工程边界和运行机制。
- 二次开发集中在独立的定制层，避免把定制逻辑散落到上游核心代码。
- App、Shared、Bootstrap、Packages、Repository、Plugins、CI/CD 分层维护。
- **最终采用三个职责独立的 GitHub 仓库：App、Packages、APT Repository。**
- 所有与上游的修改保持可追踪、可同步、可回退。
- 如果修改应用包名、前缀或其他运行时身份配置，则同步构建匹配的 Bootstrap 和软件包，禁止混用不匹配的官方二进制环境。
- 如果最终使用自定义 `TERMUX_APP_PACKAGE` / applicationId 形成独立发行版，则必须使用与该发行版身份匹配的自有 APT Repository，不能直接把官方 `com.termux` 软件仓库主机作为最终发行版仓库。
- 项目必须从第一天开始具备上游升级能力。

目标不是制作一个一次性修改版 APK，而是建立一个可以长期跟随 Termux upstream 演进的完整发行工程。

---

## 2. 三仓库总体架构

Termux-bai 最终采用三个独立 GitHub 仓库，每个仓库只负责一个明确的工程边界：

```text
Official Termux
│
├── termux-app
│       │
│       │ upstream
│       ▼
│   ① Termux-bai
│       │
│       └── Android App / 二改 / APK
│
└── termux-packages
        │
        │ upstream
        ▼
    ② termux-bai-packages
        │
        ├── 自动同步官方源码/配方
        ├── 应用 Termux-bai Patch
        ├── 构建 Bootstrap
        └── 构建 .deb
                │
                ▼
        ③ termux-bai-repo
                │
                ├── APT Repository
                ├── Packages 索引
                ├── Release metadata
                └── 签名
                │
                ▼
          Termux-bai 用户
```

三个仓库不是重复保存三份代码，而是分别承担：

| 仓库 | 核心职责 | 主要产物 |
|---|---|---|
| **Termux-bai** | Android App 二次构建与功能开发 | APK |
| **termux-bai-packages** | 官方 Packages 上游同步、适配、构建 | Bootstrap、`.deb` |
| **termux-bai-repo** | 最终软件包发行仓库 | APT Repository |

---

## 3. 仓库一：Termux-bai

### 3.1 定位

**Termux-bai** 是 Android App 本体仓库。

它不是专门负责拉取 `termux-packages` 的仓库，也不是 APT 软件仓库。

它的主要上游是官方 **`termux-app`**。

当前第一阶段基线：

```text
termux-app v0.119.0-beta.3
        ↓
    Termux-bai
```

### 3.2 主要功能

负责所有 Android App 层面的内容：

- applicationId / `TERMUX_APP_PACKAGE`
- App 名称、图标、品牌信息
- Terminal Activity
- TerminalView
- Session 管理
- TermuxService
- Intent
- Android 权限
- 通知
- 生命周期
- Settings
- Android 兼容处理
- 中文化
- 设置界面优化
- 菜单与页面结构优化
- 移动端交互优化
- Terminal UI 二改
- 字体与中文显示
- Terminal 渲染优化
- Keyboard / Extra Keys
- Theme
- Prompt
- WebUI 入口及 Android 侧集成
- Termux-bai 专属 Android 功能

### 3.3 上游同步

该仓库跟踪官方 `termux-app`，而不是直接跟踪 `termux-packages`。

```text
Official termux-app
        ↓
   Termux-bai
        ↓
  Version pin
        ↓
  Termux-bai patches
        ↓
     Build
        ↓
      APK
```

官方版本更新时，Termux-bai 建立新的版本基线，重新应用经过验证的二改 Patch，并进行完整构建测试。

---

## 4. 仓库二：termux-bai-packages

### 4.1 定位

**termux-bai-packages** 是 Termux-bai 的软件包与 Bootstrap 构建仓库。

它的主要上游是官方 **`termux-packages`**。

它负责回答一个核心问题：

> 官方 Termux 的软件包更新以后，Termux-bai 如何自动同步、修改并重新构建成属于 Termux-bai 发行版的软件包？

### 4.2 自动同步官方源码

这个仓库应该设计成自动拉取官方最新的 `termux-packages` 内容，而不是永久复制一个旧版本后人工维护。

推荐机制：

```text
Official termux-packages
        │
        │ 定时 / 手动同步
        ▼
termux-bai-packages
        │
        ├── 同步官方 package recipes
        ├── 同步官方构建脚本
        ├── 同步官方 patches
        ├── 应用 Termux-bai patches
        ├── 应用发行版配置
        └── 冲突检测
                │
                ▼
             Build
                │
        ├── Bootstrap
        └── .deb packages
```

### 4.3 Termux-bai 修改方式

原则上不直接把官方源码大量改成不可追踪的版本。

优先采用：

```text
Official upstream
       ↓
Version / commit pin
       ↓
Termux-bai patch layer
       ↓
Build
```

这样官方更新时可以重新应用 Termux-bai 的修改。

如果 Patch 与官方新版本发生冲突：

```text
同步
 ↓
Patch
 ↓
冲突
 ↓
CI 失败
 ↓
人工处理
 ↓
重新构建
```

**禁止在存在未解决冲突时自动发布正式 Repository。**

### 4.4 主要功能

- 自动同步官方 `termux-packages`
- 保存官方 packages 构建基线
- 管理 Termux-bai Packages Patch
- 构建 Bootstrap
- 构建各架构 `.deb`
- 构建 Termux-bai 定制包
- 进行依赖检查
- 进行软件包完整性检查
- 记录上游 commit/tag
- 记录 Termux-bai 修改版本
- 将构建产物交给 `termux-bai-repo`

### 4.5 软件包分类

#### A. 官方同步包

来自官方 `termux-packages`，经过 Termux-bai 自己的构建和发行版适配后重新生成。

#### B. Termux-bai 定制包

例如：

- Termux-bai 管理工具
- 开发环境初始化工具
- WebUI 相关组件
- 项目管理工具
- AI/MCP 辅助工具

#### C. 第三方集成包

必须明确来源、版本和构建方式，不把不可追踪的二进制文件直接塞进仓库。

---

## 5. 仓库三：termux-bai-repo

### 5.1 定位

**termux-bai-repo** 是 Termux-bai 的最终 APT Repository 仓库/发布层。

它不是源码构建仓库，也不是 Android App 仓库。

它负责保存和发布经过 Termux-bai 构建、验证和签名的软件包。

### 5.2 为什么必须独立

如果最终使用自定义 `TERMUX_APP_PACKAGE` / applicationId 形成独立发行版，Termux-bai 必须拥有自己的 APT Repository。

不能把官方 `com.termux` APT Repository 直接作为 Termux-bai 的最终发行仓库。

这里必须明确区分：

> **可以使用官方 `termux-packages` 作为上游源码和构建来源，但最终发布渠道属于 Termux-bai 自己。**

因此：

```text
Official termux-packages
        ↓
  上游源码 / 配方
        ↓
termux-bai-packages
        ↓
  Termux-bai 构建
        ↓
termux-bai-repo
        ↓
Termux-bai 用户
```

### 5.3 主要功能

`termux-bai-repo` 负责：

- `.deb` 软件包发布
- APT Packages 索引
- Release metadata
- Repository 签名
- 架构目录
- `main` / `root` / `x11` 等仓库组织
- stable / beta / edge 等发布通道
- 版本管理
- Repository 完整性验证
- 静态托管/CDN 配置
- 与 Bootstrap / Packages 版本对应关系

推荐最终结构：

```text
termux-bai-repo/
├── dists/
│   ├── stable/
│   ├── beta/
│   └── edge/
├── pool/
│   ├── main/
│   ├── root/
│   └── x11/
├── scripts/
└── README.md
```

实际 Repository 生成结构以 APT 发布工具和最终托管方式为准。

---

## 6. 三仓库之间的自动化关系

三个仓库通过 GitHub Actions 等 CI/CD 自动连接，而不是人工复制文件。

```text
             ┌─────────────────────┐
             │ Official termux-app │
             └──────────┬──────────┘
                        │
                 自动检测 / 同步
                        ▼
             ┌─────────────────────┐
             │     Termux-bai      │
             │ Android App 二改层   │
             └──────────┬──────────┘
                        │
                     APK
                        │
                        │
             ┌────────────────────────┐
             │ Official termux-       │
             │ packages               │
             └───────────┬────────────┘
                         │
                  自动同步 / Pin
                         ▼
             ┌────────────────────────┐
             │ termux-bai-packages    │
             │ 同步 + Patch + Build   │
             └───────────┬────────────┘
                         │
                   Bootstrap / .deb
                         │
                         ▼
             ┌────────────────────────┐
             │ termux-bai-repo        │
             │ APT 发布仓库            │
             └───────────┬────────────┘
                         │
                         ▼
                  Termux-bai Runtime
```

### 6.1 自动更新原则

官方更新后：

1. 检测新的 `termux-app` 版本。
2. 检测新的 `termux-packages` 上游变化。
3. 更新对应版本/commit 记录。
4. 应用 Termux-bai Patch。
5. 检测 Patch 冲突。
6. 构建 App。
7. 构建 Bootstrap。
8. 构建 Packages。
9. 运行测试。
10. 生成并签名 APT Repository。
11. 只有全部通过后才发布正式版本。

### 6.2 失败保护

任何关键环节失败：

```text
Sync
 ↓
Patch
 ↓
Build
 ↓
Test
 ↓
Publish
```

其中任一步失败，都不得覆盖当前稳定版本 Repository。

---

## 7. 官方基础层

### 7.1 termux-app

负责 Android 侧：

- Terminal Activity
- TerminalView
- Session 管理
- TermuxService
- Intent
- Android 权限
- 通知
- 生命周期
- Settings
- Android 兼容处理

Termux-bai 原则上保留官方 App 的职责边界，只在明确的二改模块中扩展功能。

### 7.2 termux-shared

公共 Android/Termux 代码保持独立的 shared 层。

主要包括：

- 路径常量
- Android 工具
- Intent 工具
- 权限处理
- 日志
- 公共 UI/兼容代码
- App 与插件共享逻辑

禁止在新代码中继续制造大量硬编码路径和重复的 Android 工具实现。

### 7.3 termux-packages

`termux-packages` 是 Termux 用户空间软件包及其构建系统的上游工程，不是另一个 APK。

它负责构建例如：

```text
bash
coreutils
apt
openssl
curl
wget
git
python
nodejs
clang
openssh
...
```

Termux-bai 使用它作为 Packages 层的上游来源。

---

## 8. Bootstrap 架构

Bootstrap 是新安装 Termux 后用于建立 `$PREFIX` 基础环境的核心组件。

```text
APK
 │
 ▼
Bootstrap
 │
 ├── bash
 ├── coreutils
 ├── apt
 ├── 基础运行库
 ├── 基础命令
 └── 初始化环境
 │
 ▼
$PREFIX
```

如果 Termux-bai 使用不同的 applicationId、前缀或其他身份相关配置，就必须构建与其匹配的 Bootstrap 和 Packages；不能把官方 `com.termux` 环境的二进制包直接作为 Termux-bai 的完整发行环境。

Bootstrap 由 `termux-bai-packages` 构建，并与 Termux-bai App 及 APT Repository 版本对应。

---

## 9. 二改层

所有用户体验和项目专属功能优先放到独立定制层，尽量通过独立模块或版本化 Patch 接入。

### UI

- 中文化
- 设置界面优化
- 菜单整理
- 页面结构优化
- 移动端交互优化

### Terminal

- Terminal UI 调整
- 渲染策略优化
- 滚动/复制体验
- 字体与中文显示
- 终端行为优化

### Keyboard

- Extra Keys
- 快捷键布局
- 中文开发快捷操作
- 页面化快捷键
- 自定义按键配置

### Theme

- 终端颜色主题
- 背景
- 字体
- 光暗模式
- 主题配置持久化

### Prompt

Prompt 系统必须独立于 UI 设置逻辑。

要求：

- 样式切换只修改当前 Prompt 状态。
- 颜色切换只修改当前 Prompt 状态。
- 不重复追加 Prompt。
- 不依赖用户再次回车才能刷新。
- 不污染 `.bashrc`。
- 对 Bash/Zsh 等 shell 做边界隔离。

---

## 10. Ubuntu / Proot / 开发环境集成原则

Termux-bai 本体不把 Ubuntu、Pi、PM2、MCP 等大型用户空间项目硬编码进 Android 核心。

采用：

```text
Termux-bai App
      │
      ▼
Termux 用户空间
      │
      ├── proot-distro
      │       └── Ubuntu
      ├── Node.js
      ├── Python
      ├── PM2
      ├── Pi
      ├── MCP
      └── WebUI
```

App 提供入口、管理和配置能力；具体开发工具仍由 `$PREFIX` / Ubuntu 用户空间负责。

---

## 11. 上游同步与升级策略

Termux-bai 不直接复制一个版本后永久脱离 upstream。

两条上游同步链独立维护：

```text
Official termux-app
        ↓
   Termux-bai
        ↓
   App patches
        ↓
      APK
```

以及：

```text
Official termux-packages
        ↓
termux-bai-packages
        ↓
 Packages patches
        ↓
Bootstrap + .deb
        ↓
termux-bai-repo
```

### 当前基线

```text
termux-app: v0.119.0-beta.3
```

`termux-packages` 不简单永久跟随 `master`；正式构建时必须锁定与当前 App/Bootstrap 兼容的 packages 构建基线，并记录对应 commit/tag。

### 版本管理

推荐所有上游基线和 Termux-bai Patch 都明确记录版本：

```text
app/
└── 0.119.0-beta.3/

packages/
└── <pinned-upstream-commit>/

patches/
├── app/
└── packages/
```

禁止跨版本直接套用未验证的 Patch。

### 升级流程

```text
新 upstream
   ↓
记录 App commit/tag
   ↓
记录 packages commit/tag
   ↓
建立新版本基线
   ↓
重新应用 Termux-bai patches
   ↓
自动检测冲突
   ↓
重新构建 Bootstrap
   ↓
重新构建 Packages
   ↓
发布到 Termux-bai APT Repository
   ↓
测试 APK / Runtime / APT
   ↓
生成 Release
```

因此 `v0.119.0-beta.3` 是起点，不是终点。

---

## 12. Git 分支与仓库边界

每个仓库独立维护 Git 历史：

```text
Termux-bai
├── main
├── stable
├── develop
├── upstream/*
└── feature/*

termux-bai-packages
├── main
├── upstream/*
├── packages/*
└── patch/*

termux-bai-repo
├── main
├── stable
├── beta
└── edge
```

原则：

- `main` 始终保持可构建/可发布状态。
- 大功能使用独立 feature branch。
- 上游同步单独处理。
- 发布版本打 Git tag。
- 每次重要修改必须留下明确 commit。
- 三个仓库不互相复制源码。
- 构建产物通过 CI/CD 从 Packages 仓库进入 Repository 仓库。

---

## 13. CI/CD

完整流水线：

```text
Upstream Update
      ↓
Sync
      ↓
Patch
      ↓
Lint
      ↓
Secret Scan
      ↓
Build App
      ↓
Build Bootstrap
      ↓
Build Packages
      ↓
Unit Test
      ↓
Integration Test
      ↓
Build APT Repository
      ↓
Sign
      ↓
Checksum
      ↓
APK Assemble
      ↓
Release
```

至少覆盖目标 Android CPU 架构，并在项目确认支持范围后固定架构矩阵。

发布前必须验证：

1. APK 安装
2. Bootstrap 初始化
3. `$PREFIX` 正常
4. `apt` 正常
5. APT Repository 可访问
6. 基础命令正常
7. 网络正常
8. shell 正常
9. Terminal 正常
10. Settings 正常
11. 插件/Intent 正常

---

## 14. 安全与发布

仓库中禁止出现：

- API Key
- Token
- 密码
- 私钥
- 真实账号凭据
- 本地机器敏感配置

所有 CI/CD secrets 使用 GitHub Secrets。

Repository signing key 必须独立于个人开发环境保存，并通过安全的 CI/CD secret 或专用发布环境使用。

Release 必须提供：

- APK
- SHA256
- 版本说明
- 架构说明
- 已知问题
- 上游版本
- Termux-bai 修改列表
- 对应 APT Repository channel / 版本信息

---

## 15. 推荐的三个仓库目录

### 15.1 Termux-bai

```text
Termux-bai/
├── README.md
├── ARCHITECTURE.md
├── LICENSE
├── docs/
├── app/
├── shared/
├── customization/
├── plugins/
├── patches/
├── build/
├── tests/
└── .github/
    └── workflows/
```

### 15.2 termux-bai-packages

```text
termux-bai-packages/
├── README.md
├── upstream/
├── packages/
├── patches/
├── bootstrap/
├── scripts/
├── build/
├── tests/
└── .github/
    └── workflows/
```

其中 `upstream/` 用于记录/同步官方 `termux-packages` 基线；不要求把所有官方历史版本永久复制进 Git 历史。

### 15.3 termux-bai-repo

```text
termux-bai-repo/
├── README.md
├── dists/
├── pool/
├── scripts/
└── .github/
    └── workflows/
```

Repository 仓库以**发行产物和仓库元数据**为核心，不承担软件包源码开发职责。

---

## 16. 实施阶段

### Phase 1 — 官方基线

- 固定 `termux-app v0.119.0-beta.3`
- 确认匹配的 `termux-packages` 构建基线
- 建立构建环境
- 完成原版构建验证

### Phase 2 — 三仓库建立

- 确定 applicationId
- 确定 `TERMUX_APP_PACKAGE` / 包名策略
- 建立 `Termux-bai`
- 建立 `termux-bai-packages`
- 建立 `termux-bai-repo`
- 建立三个仓库之间的 CI/CD 权限和触发关系

### Phase 3 — 身份、Bootstrap 与 Repository

- 构建对应 Bootstrap
- 构建对应 Packages
- 配置 Repository 签名和发布结构
- 验证 `pkg update / pkg upgrade / pkg install`

### Phase 4 — 二改基础层

- customization 层
- patch 管理
- 默认配置
- 中文化基础设施

### Phase 5 — Terminal

- 渲染
- 字体
- 键盘
- Prompt
- 主题

### Phase 6 — 开发环境

- proot-distro
- Ubuntu
- Node/Python
- PM2
- Pi
- MCP
- WebUI

### Phase 7 — CI/CD

- 自动同步上游
- 自动构建
- 自动测试
- 自动签名
- APT Repository 自动发布
- Release
- 失败保护与回滚

---

## 17. 最终目标

Termux-bai 最终不是“修改版 Termux APK”，而是一套完整的移动开发发行工程：

```text
                       Termux upstream
                       /            \
                      /              \
             termux-app        termux-packages
                 │                    │
                 ▼                    ▼
          ┌─────────────┐     ┌──────────────────┐
          │ Termux-bai  │     │ termux-bai-      │
          │ Android App │     │ packages         │
          └──────┬──────┘     └────────┬─────────┘
                 │                     │
                APK             Bootstrap / .deb
                 │                     │
                 │                     ▼
                 │              ┌──────────────┐
                 │              │ termux-bai-  │
                 │              │ repo         │
                 │              └──────┬───────┘
                 │                     │
                 └──────────┬──────────┘
                            ▼
                     Termux-bai Runtime
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
          Terminal       Ubuntu        Dev Tools
                              │
                       ┌──────┼───────┐
                       ▼      ▼       ▼
                      Pi     PM2    MCP/WebUI
```

核心原则：**官方结构不乱、二改功能集中、App 与 Packages 上游分开同步、Bootstrap/Packages/Repository 身份匹配、使用官方 `termux-packages` 作为上游来源、建立并维护 Termux-bai 自有 APT Repository、三个仓库职责清晰、上游可自动同步、版本可回退、构建可重复、发布可验证。**