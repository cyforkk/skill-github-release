# skill-github-release

Agent / Claude skill：用 **GitHub Actions + gh CLI** 做跨平台构建，并把产物挂到 **GitHub Release**。

> **技术栈无关。** 不绑定 Tauri、Electron、某一语言；平台数量不写死为 4 个。  
> 完整说明见仓库内 [`SKILL.md`](./SKILL.md)。

## 解决什么问题

把「打 tag → 多系统打包 → 上传下载页」固化成可复用流程，适合：

- 桌面客户端 / CLI / 单文件可执行包  
- 需要 Windows、macOS（arm/x64）、Linux 等**任意组合**产物  
- 希望用户在 Release 页一键下载，而不是自己编译  

## 核心原则

1. **一个 tag = 一次发布**；可附带 N 个平台文件（1～多个，按产品决定）  
2. **工作流只提供骨架**；构建命令、产物路径、系统依赖由**当前项目**填写  
3. **Runner 钉死版本**（如 `windows-2022`、`macos-15`、`ubuntu-24.04`），避免 `*-latest` 静默升级  
4. **矩阵 `fail-fast: false`**，单平台失败不影响其它平台  

## 功能一览

| 内容 | 说明 |
|------|------|
| gh 操作 | 登录、tag、Release、workflow 查看与重跑 |
| 通用 workflow 骨架 | `resolve` → `build`（矩阵）→ `publish` |
| Runner 选型 | 含 macos-13 排队、macos-latest 迁移等注意事项 |
| 技术栈附录 | Python/PyInstaller、Go、Rust、Node 等**可选**参考 |
| 故障排查 | 权限、无 runner、产物缺失、未触发等 |
| AI 检查清单 | 防止把 Tauri/npm 示例硬塞进无关项目 |

## 前置要求

- 安装 [GitHub CLI](https://cli.github.com/)：`brew install gh` / `winget install GitHub.cli`  
- `gh auth login`，对目标仓库有写权限  
- 目标仓库 **Settings → Actions → General → Workflow permissions** 设为 **Read and write**  

## 安装

支持两种位置：

| 位置 | 路径 | 作用域 |
|------|------|--------|
| 全局（推荐） | `~/.claude/skills/skill-github-release/` | 所有项目 |
| 项目内 | `<项目>/.claude/skills/skill-github-release/` | 仅当前仓库 |

**命令安装（从本仓库复制 `SKILL.md`）**

```bash
git clone https://github.com/cyforkk/skill-github-release.git
cd skill-github-release

# 全局 — macOS / Linux
mkdir -p ~/.claude/skills/skill-github-release
cp SKILL.md ~/.claude/skills/skill-github-release/

# 全局 — Windows PowerShell
New-Item -ItemType Directory -Force $HOME\.claude\skills\skill-github-release
Copy-Item SKILL.md $HOME\.claude\skills\skill-github-release\

# 项目内
mkdir -p /path/to/your-project/.claude/skills/skill-github-release
cp SKILL.md /path/to/your-project/.claude/skills/skill-github-release/
```

## 使用方式

对 Agent 说：

```text
使用 skill-github-release，给当前项目配置跨平台 GitHub Release
```

Agent **必须**：

1. 阅读 `SKILL.md`  
2. 识别本项目真实构建方式与产物路径  
3. 生成/改写 `.github/workflows/release.yml`（替换占位符，删除无关步骤）  
4. 不要照抄某一框架的 dmg/AppImage/npm 命令  

也可使用斜杠命令（若环境已注册）：`/skill-github-release`

## 典型发布命令（项目侧）

```bash
# 版本文件与 tag 保持一致后
git tag v1.0.0
git push origin v1.0.0
gh run list --workflow=Release
gh run watch
```

产物命名建议：`{应用名}-{系统}-{架构}[.ext]`  
例如 `myapp-windows-x64.exe`、`myapp-macos-arm64`、`myapp-linux-x64`。

## 与旧版差异

| 旧版 | 现版 |
|------|------|
| 主路径偏 Tauri / npm / Rust | 主路径为**通用占位符** |
| 示例像写死 4 平台 + dmg/AppImage | 平台数与格式**由项目决定** |
| `macos-latest` 等浮动标签 | **钉版本**；Intel 可用 arm 交叉；慎用 macos-13 |
| 故障偏框架 CLI 参数 | 增加 runner 排队、latest 漂移等**通用**问题 |

## 仓库结构

```text
skill-github-release/
├─ SKILL.md    # Agent 完整指南（主文档）
└─ README.md   # 人类可读简介与安装说明
```

## 许可证

MIT
