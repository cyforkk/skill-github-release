---
name: github-release-pipeline
description: 用于配置 GitHub Actions 跨平台构建和自动发布。涵盖 gh CLI 操作、Windows/macOS/Linux 工作流配置、Release 发布和 CI/CD 故障排查。
---

# GitHub 跨平台发布流水线

使用 gh CLI 和 GitHub Actions 实现跨平台构建和 Release 自动发布。适用于任何需要构建平台特定产物的项目（Tauri、Electron、Go、Rust 原生等）。

**为什么选择 gh + GitHub Actions：** 无需外部服务，全部运行在 GitHub 基础设施上。一条 `gh release create` 命令即可触发所有平台构建，产物自动上传到 Release 页面。

**核心流程：** 打标签触发工作流，工作流构建并上传产物，用户从 Release 页面下载。初始设置完成后无需手动操作。

## 安装

将 skill 复制到项目的 `skills/` 或 `.claude/skills/` 目录：

```bash
git clone https://github.com/cyforkk/skill-github-release.git
# 中文用户
cp github-release-pipeline/SKILL.md your-project/skills/github-release-pipeline/
# 英文用户
cp github-release-pipeline/SKILL.en.md your-project/skills/github-release-pipeline/
```

安装后，对 Claude Code 说：`使用 github-release-pipeline skill`

## 前置要求

- 安装 gh CLI：`brew install gh` / `winget install GitHub.cli`
- 已认证：`gh auth login`
- 仓库已存在：`gh repo create <name> --public`

## 阶段一：初始化配置

### 步骤 1：认证 gh CLI

```bash
gh auth login
```

选择 "GitHub.com" -> "HTTPS" -> "Login with browser"。

验证：
```bash
gh auth status
```

### 步骤 2：创建仓库（如需要）

```bash
gh repo create <repo-name> --description "描述" --public
```

### 步骤 3：配置工作流权限

`GITHUB_TOKEN` 需要写入权限才能上传 Release 产物。

GitHub 网页 -> 仓库 -> Settings -> Actions -> General -> Workflow permissions -> **Read and write**

### 步骤 4：创建工作流文件

在 `.github/workflows/release.yml` 中创建，模板见阶段二。

## 阶段二：工作流模板

### 跨平台构建矩阵

```yaml
name: Release

on:
  release:
    types: [published]

jobs:
  build:
    strategy:
      fail-fast: false
      matrix:
        include:
          - platform: 'windows-latest'
            target: ''
            name: 'app-x64.exe'
            path: 'target/release/app.exe'
          - platform: 'macos-latest'
            target: 'aarch64-apple-darwin'
            name: 'app-aarch64.dmg'
            path: 'target/aarch64-apple-darwin/release/bundle/dmg/*.dmg'
          - platform: 'macos-latest'
            target: 'x86_64-apple-darwin'
            name: 'app-x64.dmg'
            path: 'target/x86_64-apple-darwin/release/bundle/dmg/*.dmg'
          - platform: 'ubuntu-22.04'
            target: ''
            name: 'app-x64.AppImage'
            path: 'target/release/bundle/appimage/*.AppImage'

    runs-on: ${{ matrix.platform }}

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}

      - name: Install Linux dependencies
        if: matrix.platform == 'ubuntu-22.04'
        run: |
          sudo apt-get update
          sudo apt-get install -y libwebkit2gtk-4.1-dev libappindicator3-dev librsvg2-dev patchelf

      - name: Install dependencies
        run: npm install

      - name: Build (Windows)
        if: matrix.platform == 'windows-latest'
        run: npm run build

      - name: Build (macOS)
        if: matrix.platform == 'macos-latest'
        run: npm run build -- --target ${{ matrix.target }}

      - name: Build (Linux)
        if: matrix.platform == 'ubuntu-22.04'
        run: npm run build -- --bundles appimage

      - name: Upload release asset
        uses: softprops/action-gh-release@v2
        with:
          files: ${{ matrix.path }}
          tag_name: ${{ github.ref_name }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 适配你的项目

替换以下占位符：

| 占位符 | 设置为 |
|---|---|
| `npm run build` | 你的构建命令（如 `npm run tauri build`、`make`、`go build`） |
| `npm run build -- --target ...` | 交叉编译的目标三元组 |
| `npm run build -- --bundles appimage` | Linux 下覆盖 bundle 类型 |
| `target/release/app.exe` | Windows 产物路径 |
| `target/*/release/bundle/dmg/*.dmg` | macOS 产物路径 |
| `target/release/bundle/appimage/*.AppImage` | Linux 产物路径 |
| `libwebkit2gtk-4.1-dev ...` | Linux 依赖（不需要则删除） |

## 阶段三：发布流程

```bash
# 1. 提交代码
git add . && git commit -m "feat: 描述" && git push

# 2. 创建并推送版本标签
git tag v1.0.0 && git push origin v1.0.0

# 3. 创建 Release（触发 CI）
gh release create v1.0.0 \
  --title "v1.0.0" \
  --notes "## 更新内容\n- 功能A\n- 功能B\n\n## 下载\n- Windows: app-x64.exe\n- macOS: app-x64.dmg / app-aarch64.dmg\n- Linux: app-x64.AppImage"

# 4. 查看构建进度
gh run list

# 5. 查看失败日志
gh run view <run-id> --log-failed
```

## 故障排查

### 问题：Resource not accessible by integration

**现象：** Release 上传报错 `Resource not accessible by integration`

**原因：** 默认 `GITHUB_TOKEN` 只有读权限

**解决：** 仓库 Settings -> Actions -> General -> Workflow permissions -> **Read and write**

### 问题：构建参数报错 unexpected argument

**现象：** `npm run build --target aarch64-apple-darwin` 失败

**原因：** `--target` 需要通过 `--` 传给底层构建工具

**解决：**
```yaml
run: npm run build -- --target aarch64-apple-darwin
```

### 问题：--bundle 参数无法识别

**现象：** `--bundle appimage` 报错 `unexpected argument`

**原因：** CLI 使用复数形式 `--bundles`

**解决：**
```yaml
run: npm run build -- --bundles appimage
```

### 问题：macOS dmg 未出现在 Release

**现象：** macOS 构建成功但 Release 页面没有 dmg

**原因：** 项目配置中未包含 dmg 格式

**解决：** 在项目 bundle targets 配置中添加 `dmg`（如 `tauri.conf.json` -> `bundle.targets`）

### 问题：Action 参数名变更

**现象：** `softprops/action-gh-release` 报错 `Unexpected input(s) 'tag'`

**原因：** v2 中 `tag` 参数已更名为 `tag_name`

**解决：**
```yaml
with:
  tag_name: ${{ github.ref_name }}
```

### 问题：工作流未触发

**现象：** 推送代码后工作流不执行

**原因：** 文件位置错误或 YAML 语法问题

**解决：**
- 确保文件在 `.github/workflows/release.yml`
- 验证 YAML 语法
- 检查仓库 Settings -> Actions 中工作流是否启用

## 最佳实践

1. **使用 `fail-fast: false`**：某个平台失败不影响其他平台继续构建
2. **锁定 Action 版本**（`@v4`、`@v2`）：避免破坏性变更
3. **使用 `--` 分隔符**：通过 npm 脚本传参给底层工具时
4. **本地先测试**：用 `act` CLI 在本地运行 workflow
5. **语义化版本标签**：`v1.0.0`、`v1.1.0`、`v2.0.0`
6. **Release 说明简洁**：按平台列出下载项
7. **显式设置 `GITHUB_TOKEN`**：在上传步骤中明确声明

## 关联工具

**配套 skill：**
- **superpowers:writing-plans** — 构建前规划功能时使用
- **superpowers:subagent-driven-development** — 多平台并行构建时使用

**相关 CLI 工具：**
- `gh` — GitHub CLI，仓库和 Release 管理
- `act` — 本地运行 GitHub Actions 工作流
