# n2n 多平台自动编译

自动从 [ntop/n2n](https://github.com/ntop/n2n) 上游拉取 Release，编译并发布多平台预编译二进制文件。

## 支持平台

| 平台    | 架构   | 说明                         |
|---------|--------|------------------------------|
| Linux   | x86    | 32-bit x86                   |
| Linux   | amd64  | 64-bit x86 (x86_64)          |
| Linux   | arm64  | AArch64 交叉编译             |
| Windows | x86    | 32-bit，MinGW 交叉编译       |
| Windows | amd64  | 64-bit，MinGW 交叉编译       |
| Windows | arm64  | ARM64，llvm-mingw 交叉编译   |
| macOS   | amd64  | Intel Mac (macos-13 runner)  |
| macOS   | arm64  | Apple Silicon M1+ (macos-14) |

## 工作流说明

### `check-release.yml` — 定时检查上游新版本

- **触发时间**：北京时间每天凌晨 2:00（UTC 18:00）
- **功能**：
  1. 查询 `ntop/n2n` 最新 stable release 和最新 pre-release
  2. 与 `.github/release-state.json` 中记录的已编译版本对比
  3. 发现新版本时，自动触发 `build.yml`
  4. 更新 `release-state.json` 防止重复编译

### `build.yml` — 多平台编译与发布

- **触发方式**：
  - 由 `check-release.yml` 自动触发（传入 tag、是否 prerelease 等参数）
  - 手动 `workflow_dispatch`，可指定任意 tag
  - 首次推送 `build.yml` 到主分支时，自动编译当前上游最新的 stable + pre-release 两个版本
- **编译流程**：
  - Linux：Ubuntu runner，CMake + GCC，arm64 使用 aarch64-linux-gnu 工具链交叉编译
  - Windows：Ubuntu runner，MinGW-w64 / llvm-mingw 交叉编译，输出 `.zip`
  - macOS：分别在 `macos-13`（Intel）和 `macos-14`（M1）原生 runner 上编译
- **产物**：自动创建 GitHub Release，上传所有平台压缩包

## 使用方法

### 1. 将本仓库三个文件放入你的仓库

```
.github/
├── workflows/
│   ├── check-release.yml
│   └── build.yml
└── release-state.json
```

### 2. 确认默认分支名

两个 workflow 文件中有两处 `ref: 'main'`，请改成你实际的默认分支名（如 `master`）：

- `check-release.yml` 第 95 行 `ref: 'main'`
- `check-release.yml` 第 112 行 `ref: 'main'`
- `build.yml` 第 53 行 `branches: - main`
- `build.yml` 第 54 行 `paths: - '.github/workflows/build.yml'`

### 3. 开启 Actions 写权限

进入仓库 → **Settings → Actions → General → Workflow permissions**，选择 **Read and write permissions**，并勾选 **Allow GitHub Actions to create and approve pull requests**。

### 4. 首次触发编译

推送 `build.yml` 到主分支后，`push` 触发器会自动查询上游最新 stable + pre-release 并各编译一次。

之后每天凌晨 2 点（北京时间）会自动检查是否有新版本。

### 5. 手动触发编译任意版本

进入 **Actions → Build n2n → Run workflow**，填写 tag（如 `3.1.1`）和是否为预发行版。

## 注意事项

- **Windows arm64**：使用 [llvm-mingw](https://github.com/mstorsjo/llvm-mingw) 交叉编译，构建时需要从 GitHub 下载约 200MB 工具链，首次较慢。
- **macOS 二进制**：使用 ad-hoc codesign 签名，用户运行前可能需要在"系统设置 → 隐私与安全性"中手动允许。
- **`release-state.json`**：由 Action 自动维护，请勿手动修改（会导致状态不一致）。
- **GITHUB_TOKEN 权限**：已在各 job 中通过 `permissions` 字段声明，无需额外配置 PAT。
