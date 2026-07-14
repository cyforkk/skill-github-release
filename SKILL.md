---
name: skill-github-release
description: 配置 GitHub Actions 跨平台构建与 GitHub Release 自动发布。技术栈无关（Python/Go/Rust/Node/Tauri/Electron 等均可），含 gh CLI、矩阵构建、runner 选型与故障排查。
---

# GitHub 跨平台发布流水线（通用）

用 **gh CLI + GitHub Actions** 做「打标签 → 多平台构建 → 上传 Release」。

**适用：** 任何需要产出**按操作系统/架构区分的可运行包**的项目，不限定语言或打包工具。

**核心原则：**

1. **一个版本号（tag）对应一次发布**；一次发布可附带 **N 个平台产物**（常见 3～4 个，数量由项目决定，不是固定写死 4 个）。
2. **工作流只描述流程**；**构建命令、产物路径、系统依赖** 由具体项目填入占位符。
3. **Runner 标签尽量钉死版本**，不要依赖会漂移的 `*-latest`。
4. **矩阵 `fail-fast: false`**：单平台失败不拖死其它平台。

---

## 何时用这个 skill

用户说「配置跨平台发布」「Release 自动构建」「Windows/macOS/Linux 打包上传」时启用。

AI 落地时必须：

1. 先读项目现有构建方式（Makefile、`cargo`、`go build`、`uv`/`pyinstaller`、`npm`/`pnpm`、Tauri 等）。
2. **不要**把下文中的示例命令原样当成唯一真理。
3. 把占位符换成该仓库真实命令与路径。
4. 确认仓库 Actions 权限为 **Read and write**。

---

## 安装

```bash
# 全局（推荐）
# macOS / Linux
mkdir -p ~/.claude/skills/skill-github-release
cp SKILL.md ~/.claude/skills/skill-github-release/

# Windows PowerShell
New-Item -ItemType Directory -Force $HOME\.claude\skills\skill-github-release
Copy-Item SKILL.md $HOME\.claude\skills\skill-github-release\

# 或仅当前项目
mkdir -p .claude/skills/skill-github-release
cp SKILL.md .claude/skills/skill-github-release/
```

对 Agent 说：`使用 skill-github-release` / 配置跨平台 GitHub Release。

---

## 前置要求

- 已安装 `gh`（`brew install gh` / `winget install GitHub.cli`）
- `gh auth login` 且对目标仓库有写权限
- 仓库已存在；能推送 tag

---

## 阶段一：一次性仓库配置

### 1. 认证

```bash
gh auth login
gh auth status
```

### 2. 工作流权限（必做）

仓库 → **Settings → Actions → General → Workflow permissions**  
选择 **Read and write permissions**  

否则上传 Release 会报：`Resource not accessible by integration`。

### 3. 版本约定（由项目自定，保持一致）

任选一种并在全仓库统一，例如：

| 方式 | 示例 |
|------|------|
| 根目录版本文件 | `VERSION` 内容为 `v1.2.3`，tag 必须相同 |
| package.json / Cargo.toml / pyproject.toml | 与 tag 语义一致（是否带 `v` 前缀由项目约定） |

工作流里可加校验：tag 与版本文件不一致则失败。

---

## 阶段二：通用工作流骨架

路径：`.github/workflows/release.yml`

下面模板**故意不绑定**某一种语言。矩阵里用「语义字段」描述平台；`BUILD_CMD` / 产物路径由项目替换。

### 推荐触发方式（可并存）

```yaml
on:
  push:
    tags: ["v*"]           # 推送标签触发
  release:
    types: [published]     # 先建 Release 再构建也可
  workflow_dispatch:
    inputs:
      tag:
        description: "已有 tag，如 v1.0.0"
        required: true
        type: string
```

### Runner 选型（重要：不要写死 latest）

| 目标产物 | 推荐 `runs-on` | 说明 |
|----------|----------------|------|
| Windows x64 | `windows-2022` 或当前文档推荐的固定标签 | 避免 `windows-latest` 静默升级 |
| macOS arm64 | `macos-14` / `macos-15` 等**固定版本** | **不要**用 `macos-latest` 作为长期依赖（标签会迁移系统大版本） |
| macOS x64 (Intel) | 优先：在 **arm64 runner** 上交叉编译 / `universal`；或仍可用的 Intel 镜像 | **`macos-13` 常出现长时间 Waiting for a runner**，慎用 |
| Linux x64 | `ubuntu-22.04` 或 `ubuntu-24.04`（钉版本） | 按运行时依赖装系统包 |

> 原则：**钉版本、可复现**；每年检查一次 GitHub runner-images 公告再升级。

### 通用矩阵示例（占位符）

```yaml
name: Release

on:
  push:
    tags: ["v*"]
  workflow_dispatch:
    inputs:
      tag:
        description: "现有 tag，如 v1.0.0"
        required: true
        type: string

permissions:
  contents: write

jobs:
  resolve:
    runs-on: ubuntu-24.04
    outputs:
      tag: ${{ steps.t.outputs.tag }}
      ref: ${{ steps.t.outputs.ref }}
    steps:
      - id: t
        shell: bash
        run: |
          if [ "${{ github.event_name }}" = "workflow_dispatch" ]; then
            echo "tag=${{ inputs.tag }}" >> "$GITHUB_OUTPUT"
            echo "ref=${{ inputs.tag }}" >> "$GITHUB_OUTPUT"
          else
            echo "tag=${{ github.ref_name }}" >> "$GITHUB_OUTPUT"
            echo "ref=${{ github.ref }}" >> "$GITHUB_OUTPUT"
          fi

  build:
    needs: resolve
    strategy:
      fail-fast: false
      matrix:
        include:
          # ——— 按项目增删行；字段名可改，保持语义清晰 ———
          - os: windows-2022
            asset: myapp-windows-x64.exe      # 最终上传到 Release 的文件名
            artifact_path: dist/myapp.exe    # 构建后实际路径
            # extra: 任意项目自定义字段，如 target_arch、goos、goarch

          - os: macos-15
            asset: myapp-macos-arm64
            artifact_path: dist/myapp
            # target_arch: arm64

          - os: macos-15
            asset: myapp-macos-x64
            artifact_path: dist/myapp
            # target_arch: x86_64   # 在 arm runner 上交叉时使用；需工具链支持

          - os: ubuntu-24.04
            asset: myapp-linux-x64
            artifact_path: dist/myapp

    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ needs.resolve.outputs.ref }}

      # ——— 安装运行时 / 工具链（按项目二选一或组合，删除无关步骤）———
      # 示例仅作「位置提示」，命令必须替换：
      # - uses: actions/setup-python@v5 / astral-sh/setup-uv@v6
      # - uses: actions/setup-go@v5
      # - uses: actions/setup-node@v4
      # - uses: dtolnay/rust-toolchain@stable
      # - 系统包：sudo apt-get install -y <你的依赖>

      - name: Install dependencies
        run: |
          # 替换为：uv sync / go mod download / npm ci / cargo fetch 等
          echo "TODO: install project deps"

      - name: Build
        env:
          # 可选：把矩阵字段传给构建脚本
          RELEASE_ASSET_NAME: ${{ matrix.asset }}
          # TARGET_ARCH: ${{ matrix.target_arch }}
        run: |
          # 替换为真实构建命令，并保证产出 matrix.artifact_path
          # 例：
          #   uv run pyinstaller app.spec
          #   go build -o dist/myapp ./cmd/myapp
          #   cargo build --release
          #   npm run build
          echo "TODO: build for ${{ matrix.os }}"
          test -f "${{ matrix.artifact_path }}"

      - name: Normalize asset filename
        shell: bash
        run: |
          mkdir -p release-out
          cp "${{ matrix.artifact_path }}" "release-out/${{ matrix.asset }}"
          chmod +x "release-out/${{ matrix.asset }}" 2>/dev/null || true

      - uses: actions/upload-artifact@v4
        with:
          name: ${{ matrix.asset }}
          path: release-out/${{ matrix.asset }}
          if-no-files-found: error

  publish:
    needs: [resolve, build]
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ needs.resolve.outputs.ref }}

      - uses: actions/download-artifact@v4
        with:
          path: artifacts

      - name: Collect files
        shell: bash
        run: |
          mkdir -p dist
          find artifacts -type f -exec cp -v {} dist/ \;
          ls -la dist

      - name: Create or update GitHub Release
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        shell: bash
        run: |
          set -euo pipefail
          tag="${{ needs.resolve.outputs.tag }}"
          mapfile -t files < <(find dist -type f | sort)
          notes="Release ${tag}

          本版本多平台构建产物见 Assets。
          "
          if gh release view "$tag" --json tagName -q .tagName >/dev/null 2>&1; then
            gh release upload "$tag" "${files[@]}" --clobber
          else
            gh release create "$tag" "${files[@]}" --title "$tag" --notes "$notes"
          fi
```

### 矩阵要出几个包？

**不固定。** 由产品需要决定：

- 只支持 Windows → 1 行  
- Win + Linux → 2 行  
- Win + macOS arm + Linux → 3 行  
- 再加 macOS Intel → 4 行（常见，但不是强制）

产物命名建议带平台与架构，例如：

`{app}-{os}-{arch}[.ext]`  
如 `myapp-windows-x64.exe`、`myapp-macos-arm64`、`myapp-linux-x64`。

---

## 阶段三：按技术栈填写构建（附录，可选参考）

以下**不是**默认必选，只在项目匹配时参考。

### A. Python + PyInstaller / 类似单文件打包

- 安装：`uv` / `pip` + 打包工具  
- Linux GUI（Tk 等）常需系统包：`python3-tk`、`tcl`、`tk` 等；**托管 Python 可能没有 `_tkinter`，优先系统解释器或已带 Tk 的发行版**  
- macOS 多架构：可用环境变量/`target_arch` 在 **同一 arm runner** 上分别打 `arm64` 与 `x86_64`（x86_64 常需 Rosetta），**避免强依赖 macos-13**  
- 产物：`dist/App.exe` 或 `dist/App`，再改名为带平台后缀

### B. Go

```bash
GOOS=windows GOARCH=amd64 go build -o dist/myapp-windows-x64.exe .
GOOS=darwin  GOARCH=arm64 go build -o dist/myapp-macos-arm64 .
GOOS=linux   GOARCH=amd64 go build -o dist/myapp-linux-x64 .
```

可在单 OS runner 交叉编译，矩阵可简化。

### C. Rust

```bash
rustup target add x86_64-unknown-linux-gnu   # 按需
cargo build --release --target <triple>
```

### D. Node / Electron / Tauri 等

- 使用项目自己的 `package.json` 脚本  
- 通过 `npm run <script> -- --flags` 把参数传给底层 CLI（注意 `--`）  
- 系统依赖、签名、公证按该生态文档配置，**不要**假设所有项目都有 dmg/AppImage

---

## 阶段四：日常发布流程

```bash
# 1. 更新版本约定文件并提交
git add -A && git commit -m "chore: release v1.0.0" && git push

# 2. 打标签（与版本约定一致）
git tag v1.0.0
git push origin v1.0.0

# 3. 查看流水线
gh run list --workflow=Release
gh run watch

# 4. 失败时
gh run view <run-id> --log-failed

# 5. 用默认分支上最新 workflow 重跑某 tag（不迁 tag 时）
gh workflow run Release -f tag=v1.0.0
```

也可：`gh release create v1.0.0 --generate-notes` 再让 `release: published` 触发（若工作流支持）。

---

## 故障排查（通用）

### Resource not accessible by integration

- **原因：** token 无写权限  
- **处理：** Workflow permissions → Read and write  

### Waiting for a runner… 很久

- **原因：** 该 label 机器少或已缩容（常见于旧版 `macos-13`）  
- **处理：** 换仍有容量的固定 label；或交叉编译；取消卡住的 run  

### macos-latest / windows-latest 行为突变

- **原因：** latest 指向的系统镜像会迁移  
- **处理：** 钉死 `macos-15`、`windows-2022` 等，升级时显式改 PR  

### 构建成功但 Release 无文件

- 检查 `artifact_path` 是否真实存在  
- 上传步骤 `if-no-files-found: error`  
- `softprops/action-gh-release` 使用 `tag_name`（不是旧参数 `tag`）  
- 或用 `gh release upload` 并确认 `GH_TOKEN`  

### 工作流未触发

- 文件是否在 `.github/workflows/*.yml`  
- tag 是否匹配 `v*`  
- 仓库 Actions 是否启用  
- YAML 语法是否合法  

### 参数没传到打包 CLI

- npm/yarn 需：`npm run build -- --your-flag`  

---

## 最佳实践

1. **`fail-fast: false`**，平台彼此隔离  
2. **钉死 runner 与 Action 大版本**（`@v4`），升级单独做  
3. **产物名含 os + arch**，用户不用猜  
4. **版本源唯一**：tag ↔ VERSION/元数据 校验  
5. **publish 与 build 分 job**：先 artifact 再汇总上传，避免一边失败一边半上传  
6. **密钥与签名**：签名证书用 GitHub Secrets，不要写进仓库  
7. **文档写清下载表**：Release notes 里列出各平台文件名  
8. **技术栈相关步骤可删**：Linux 桌面依赖、移动端等按需保留  

---

## AI 执行检查清单

配置完成前确认：

- [ ] 已识别项目真实构建命令与产物路径  
- [ ] 矩阵行数符合产品需求（非硬性 4 平台）  
- [ ] runner 使用固定版本，避开无容量旧镜像  
- [ ] `permissions.contents: write` + 仓库 UI 写权限  
- [ ] tag 与版本约定一致  
- [ ] README/项目文档说明如何下载各平台包  
- [ ] 未把 Tauri/npm/Rust 示例原样塞进不相关项目  

---

## 关联工具

- `gh` — 仓库、Release、workflow  
- `act` — 可选，本地试跑 Actions（完整 macOS 矩阵通常仍需云端）
