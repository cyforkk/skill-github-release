# skill-github-release

Claude Code skill — 跨平台构建与 GitHub Release 自动发布流水线。

## 功能特性

- gh CLI 操作（仓库管理、Release 管理、工作流管理）
- Windows/macOS/Linux 矩阵构建工作流模板
- 完整的发布流程
- 6 个常见问题及解决方案
- 7 条最佳实践

## 前置要求

- 安装 gh CLI：`brew install gh` / `winget install GitHub.cli`
- GitHub 账号认证：`gh auth login`

## 安装方式

支持两种安装位置：
- **全局 skill**（推荐）：`~/.claude/skills/skill-github-release/`，所有项目共享
- **项目 skill**：`<项目>/.claude/skills/skill-github-release/`，仅当前项目可用

**方式一：命令安装**

```bash
git clone https://github.com/cyforkk/skill-github-release.git

# 全局安装（推荐）—— macOS / Linux
mkdir -p ~/.claude/skills/skill-github-release
cp skill-github-release/SKILL.md ~/.claude/skills/skill-github-release/

# 全局安装 —— Windows (PowerShell)
New-Item -ItemType Directory -Force $HOME\.claude\skills\skill-github-release
Copy-Item skill-github-release\SKILL.md $HOME\.claude\skills\skill-github-release\
```

> 项目安装：将目标路径中的 `~/.claude/skills/`（macOS/Linux）或 `$HOME\.claude\skills\`（Windows）改为 `<项目>/.claude/skills/` 即可。

**方式二：手动复制**

将 `SKILL.md` 复制到目标目录（不存在则先创建）：
- 全局（推荐）：`~/.claude/skills/skill-github-release/SKILL.md`
- 项目：`<项目>/.claude/skills/skill-github-release/SKILL.md`

## 使用方式

安装后，对 Claude Code 说：

> 使用 skill-github-release skill

若 skill 已注册为可调用，也可使用斜杠命令：

```
/skill-github-release
```

## 许可证

MIT
