# Termux-bai — 二次构建完整架构

## 1. 项目定位

Termux-bai 是基于 Termux 官方工程架构进行长期维护的二次构建项目。

核心原则：

- 保留 Termux 官方的核心工程边界和运行机制。
- 二次开发集中在独立的定制层，尽量避免把定制逻辑散落到上游核心代码。
- App、Bootstrap、Packages、Repository、Plugins、CI/CD 分层维护。
- 所有与上游的修改保持可追踪、可同步、可回退。
- 如果修改应用包名或运行时身份，则同步构建匹配的 Bootstrap 和软件包，禁止混用不匹配的官方二进制环境。

目标不是制作一个一次性修改版 APK，而是建立一个可以长期跟随 Termux upstream 演进的完整发行工程。

---

## 2. 总体架构

```text
Termux-bai
│
├── upstream/
│   ├── termux-app/              # Android App 上游
│   ├── termux-packages/         # 软件包构建上游
│   └── plugins/                 # 官方插件上游（按需）
│
├── app/                         # App 二次开发层
│   ├── terminal/
│   ├── session/
│   ├── settings/
│   ├── service/
│   └── ui/
│
├── shared/                      # 公共代码/兼容层
│
├── bootstrap/                   # 各架构 Bootstrap 构建结果
│   ├── aarch64/
│   ├── arm/
│   ├── x86_64/
│   └── i686/
│
├── packages/                   # Termux packages 定制层
│   ├── bash/
│   ├── coreutils/
│   ├── apt/
│   ├── git/
│   ├── python/
│   ├── nodejs/
│   └── custom/
│
├── repository/                  # 自有 APT 仓库生成层
│   ├── packages/
│   ├── Packages/
│   └── release/
│
├── plugins/                     # 插件/扩展
│
├── customization/               # Termux-bai 二改功能
│   ├── ui/
│   ├── terminal/
│   ├── theme/
│   ├── keyboard/
│   ├── prompt/
│   ├── settings/
│   ├── shell/
│   └── defaults/
│
├── patches/                     # 严格版本化的上游补丁
│   └── <upstream-version>/
│
├── build/                       # 构建脚本、环境与检查
├── tests/                       # 测试
├── .github/workflows/           # CI/CD
└── docs/                        # 项目文档
```

---

## 3. 官方基础层

### 3.1 termux-app

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

### 3.2 termux-shared

公共 Android/Termux 代码放在 shared 层。

主要包括：

- 路径常量
- Android 工具
- Intent 工具
- 权限处理
- 日志
- 公共 UI/兼容代码
- App 与插件共享逻辑

禁止在新代码中继续制造大量硬编码路径和重复的 Android 工具实现。

### 3.3 termux-packages

负责 Termux 用户空间软件包：

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

软件包不是简单复制到 APK 中，而是通过 Termux packages 构建系统产生对应架构的包和 Bootstrap。

---

## 4. Bootstrap 架构

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
 ├── libc / linker 相关运行环境
 ├── 基础库
 └── 初始命令
 │
 ▼
$PREFIX
 │
 ├── bin/
 ├── lib/
 ├── include/
 ├── share/
 └── etc/
```

支持的 Android CPU 架构必须分别构建并验证。

重点原则：如果 Termux-bai 使用不同的应用包名、前缀或其他身份相关配置，就必须构建与其匹配的 Bootstrap 和 Packages；不能把官方 `com.termux` 环境的二进制包直接当作 Termux-bai 的完整发行环境。

---

## 5. Packages 层

Packages 分成三类：

### A. 官方同步包

尽量直接跟随 upstream，仅在必要时打补丁。

### B. Termux-bai 定制包

用于项目专属能力，例如：

- Termux-bai 管理工具
- 开发环境初始化工具
- WebUI 相关组件
- 项目管理工具
- AI/MCP 辅助工具

### C. 第三方集成包

必须明确来源、版本和构建方式，不把不可追踪的二进制文件直接塞进仓库。

---

## 6. Repository 层

Termux-bai 使用独立的软件包发布层：

```text
packages
   ↓
package build
   ↓
.deb
   ↓
APT metadata
   ↓
Repository
   ↓
Termux-bai apt
```

Repository 应支持：

- Packages
- Packages.xz / 压缩索引
- Release metadata
- 版本管理
- 架构区分
- 校验
- 发布记录

这样以后 Termux-bai 可以独立发布自己的软件包，而不需要重新发布整个 APK。

---

## 7. 二改层

所有用户体验和项目专属功能优先放到 `customization/`。

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

### Defaults

集中管理首次安装默认值：

- 默认主题
- 默认键盘
- 默认终端设置
- 默认 shell
- 默认 Prompt
- 默认语言

---

## 8. Ubuntu / Proot / 开发环境集成原则

Termux-bai 本体不应该把 Ubuntu、Pi、PM2、MCP 等大型用户空间项目硬编码进 Android 核心。

采用：

```text
Termux-bai App
      │
      ▼
Termux 用户空间
      │
      ├── proot-distro
      │       └── Ubuntu
      │
      ├── Node.js
      ├── Python
      ├── PM2
      ├── Pi
      ├── MCP
      └── WebUI
```

App 提供入口、管理和配置能力；具体开发工具仍由 `$PREFIX` / Ubuntu 用户空间负责。

这样可以保持 Termux 本体干净，同时让 Termux-bai 成为完整移动开发环境。

---

## 9. PM2 / MCP / WebUI 的位置

推荐结构：

```text
Android
  │
  ▼
Termux-bai
  │
  ▼
Termux $PREFIX
  │
  ├── PM2
  │    └── 管理长期运行服务
  │
  ├── MCP
  │    └── shell / file / python / git / pm2 / container
  │
  └── WebUI
       └── 浏览器访问本地服务
```

需要特别避免：

- 把 MCP 服务直接写进 Termux Android 核心。
- 把 PM2 生命周期强行绑定到 TerminalView。
- 把 WebUI 变成 Android 核心依赖。

这些都属于可选的用户空间能力。

---

## 10. 上游同步策略

Termux-bai 不直接复制一个版本后永久脱离 upstream。

采用：

```text
Upstream
   │
   ▼
版本同步
   │
   ▼
Termux-bai patch
   │
   ├── 官方修改
   ├── Termux-bai 修改
   └── 冲突检查
   │
   ▼
Build
   │
   ▼
Test
   │
   ▼
Release
```

每个重大上游版本使用独立补丁目录：

```text
patches/
└── 0.x.x/
    ├── app.patch
    ├── shared.patch
    ├── build.patch
    └── customization.patch
```

禁止跨版本直接套补丁。

---

## 11. Git 分支策略

建议：

```text
main
 │
 ├── stable
 │
 └── develop

upstream/*
 │
 └── 官方同步

feature/*
 │
 └── 单项二改功能
```

原则：

- `main` 始终保持可构建。
- 大功能单独 feature branch。
- 上游同步单独处理。
- 发布版本打 Git tag。
- 每次重要修改必须留下明确 commit。

---

## 12. CI/CD

完整流水线：

```text
Push
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
APK Assemble
 ↓
Sign
 ↓
Checksum
 ↓
Release
```

至少覆盖：

- aarch64
- arm
- x86_64
- i686（如果目标版本仍支持）

发布前必须验证：

1. APK 安装
2. Bootstrap 初始化
3. `$PREFIX` 正常
4. `apt` 正常
5. 基础命令正常
6. 网络正常
7. shell 正常
8. Terminal 正常
9. Settings 正常
10. 插件/Intent 正常

---

## 13. 安全与发布

仓库中禁止出现：

- API Key
- Token
- 密码
- 私钥
- 真实账号凭据
- 本地机器敏感配置

所有 CI/CD secrets 使用 GitHub Secrets。

Release 必须提供：

- APK
- SHA256
- 版本说明
- 架构说明
- 已知问题
- 上游版本
- Termux-bai 修改列表

---

## 14. 推荐的最终目录

```text
Termux-bai/
├── README.md
├── ARCHITECTURE.md
├── LICENSE
├── docs/
│   ├── BUILD.md
│   ├── DEVELOPMENT.md
│   ├── RELEASE.md
│   ├── UPSTREAM.md
│   └── CUSTOMIZATION.md
├── upstream/
├── app/
├── shared/
├── bootstrap/
├── packages/
├── repository/
├── customization/
├── plugins/
├── patches/
├── build/
├── tests/
└── .github/
    └── workflows/
```

---

## 15. 实施顺序

不要一开始就大规模修改 Terminal UI。

推荐顺序：

### Phase 1 — 官方基线

- 固定 upstream 版本
- 导入 termux-app
- 导入 termux-packages
- 建立构建环境
- 完成原版构建

### Phase 2 — 身份与 Bootstrap

- 确定 applicationId
- 确定包名策略
- 构建对应 Bootstrap
- 构建对应 Packages
- 建立 APT Repository

### Phase 3 — 二改基础层

- customization 层
- patch 管理
- 默认配置
- 中文化基础设施

### Phase 4 — Terminal

- 渲染
- 字体
- 键盘
- Prompt
- 主题

### Phase 5 — 开发环境

- proot-distro
- Ubuntu
- Node/Python
- PM2
- Pi
- MCP
- WebUI

### Phase 6 — CI/CD

- 自动构建
- 自动测试
- 自动签名
- Release
- 上游同步

---

## 16. 最终目标

Termux-bai 最终不是“修改版 Termux APK”，而是一套完整的移动开发发行工程：

```text
                 Termux upstream
                       │
                       ▼
              ┌─────────────────┐
              │   Termux-bai    │
              │  二次构建层      │
              └────────┬────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
        App         Bootstrap     Packages
          │            │            │
          └────────────┼────────────┘
                       ▼
                APT Repository
                       │
                       ▼
              Termux-bai Runtime
                       │
       ┌───────────────┼────────────────┐
       ▼               ▼                ▼
    Terminal         Ubuntu          Dev Tools
                         │
                 ┌───────┼────────┐
                 ▼       ▼        ▼
                Pi      PM2      MCP/WebUI
```

核心原则：**官方结构不乱、二改功能集中、Bootstrap/Packages 匹配、上游可同步、版本可回退、构建可重复、发布可验证。**
