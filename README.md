# skill-github-release

Claude Code skill — 跨平台构建与 GitHub Release 自动发布流水线。

## 功能特性

- gh CLI 操作（仓库管理、Release 管理、工作流管理）
- Windows/macOS/Linux 矩阵构建工作流模板
- 完整的发布流程
- 6 个常见问题及解决方案
- 7 条最佳实践

## 安装方式

```bash
git clone https://github.com/cyforkk/skill-github-release.git

# macOS / Linux
cp skill-github-release/SKILL.md your-project/skills/skill-github-release/

# Windows (CMD / PowerShell)
copy skill-github-release\SKILL.md your-project\skills\skill-github-release\
```

## 使用方式

安装后对 Claude Code 说：

> 使用 skill-github-release skill

## 前置要求

- 安装 gh CLI：`brew install gh` / `winget install GitHub.cli`
- GitHub 账号认证：`gh auth login`

## 许可证

MIT
