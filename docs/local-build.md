# 本地编译 opencode

本文档记录从源码本地编译 opencode 并安装的完整流程。

## 前置要求

- **Bun 1.3+**：项目 `packageManager` 字段指定 `1.3.14`，建议使用相同版本

```bash
bun --version
```

## 1. 安装依赖

从 repo 根目录执行：

```bash
bun install
```

如果网络受限，使用代理：

```bash
HTTP_PROXY=http://localhost:8118 HTTPS_PROXY=http://localhost:8118 bun install
```

依赖较多（40+ workspace 包），首次安装可能需要 5-10 分钟。

## 2. 编译二进制

```bash
OPENCODE_CHANNEL=local OPENCODE_VERSION=1.18.1-local \
  ./packages/opencode/script/build.ts --single --skip-install
```

参数说明：

| 参数 | 说明 |
|------|------|
| `--single` | 仅编译当前平台（如 `darwin-arm64`），不交叉编译全部 12 个目标 |
| `--skip-install` | 跳过 `bun install --os="*" --cpu="*"` 跨平台依赖下载步骤（该步骤会下载 Windows/Linux 的 native 包，在国内网络环境容易超时） |
| `OPENCODE_CHANNEL=local` | 设置 channel 为 `local`，数据库将使用 `opencode-local.db`，与发布版隔离 |
| `OPENCODE_VERSION=1.18.1-local` | 版本号使用 release 版本加 `-local` 后缀，便于区分 |

编译产物位于 `packages/opencode/dist/opencode-<platform>/bin/opencode`。

验证：

```bash
./packages/opencode/dist/opencode-darwin-arm64/bin/opencode --version
# 输出: 1.18.1-local
```

### Channel 与数据库

版本号和数据库文件名由 channel 决定（见 `packages/core/src/database/database.ts` 的 `path()` 函数）：

| channel | 数据库文件 | 说明 |
|---------|-----------|------|
| `latest` | `opencode.db` | 发布版默认 |
| `beta` / `prod` | `opencode.db` | 与 latest 共用 |
| `local` | `opencode-local.db` | 本地编译（推荐） |
| `dev`（git 分支名） | `opencode-dev.db` | 不设 `OPENCODE_CHANNEL` 时的默认值 |
| 任意其他值 | `opencode-{channel}.db` | 按 channel 名隔离 |

设置 `OPENCODE_DISABLE_CHANNEL_DB=1` 可强制所有 channel 使用 `opencode.db`。

## 3. 安装到本地

推荐使用软链接，重新编译后自动生效无需重装：

```bash
mkdir -p ~/bin
ln -sf $(pwd)/packages/opencode/dist/opencode-darwin-arm64/bin/opencode ~/bin/opencode
```

确保 `~/bin` 在 PATH 中。在 `~/.zshrc` 添加：

```bash
export PATH="$HOME/bin:$PATH"
```

验证：

```bash
opencode --version
# 输出: 1.18.1-local
```

> 如果 `opencode --version` 显示旧版本，检查 `which opencode`，确认 `~/bin` 在 PATH 中优先级高于其他安装路径（如 `/opt/homebrew/bin`）。

## 4. 迁移 Session（可选）

切换 channel 后数据库会变。如需从旧库迁移 session：

```bash
# 查看旧库的 session
OPENCODE_DB=~/.local/share/opencode/opencode.db opencode session list

# 逐个导出
OPENCODE_DB=~/.local/share/opencode/opencode.db opencode export <sessionID> > /tmp/session.json

# 导入到新库（默认 channel=local 的库）
opencode import /tmp/session.json
```

> **不要直接 `cp` 数据库文件**，可能导致 `database disk image is malformed` 错误。使用 `export` / `import` 更安全。

## 常见问题

### bun install 超时

使用代理（见上方步骤 1）。也可设置 npm 镜像：

```bash
BUN_CONFIG_REGISTRY=https://registry.npmmirror.com bun install
```

### 编译时报 @esbuild 下载失败

加 `--skip-install` 跳过跨平台依赖下载。

### 数据库文件损坏

不要直接复制数据库文件。如果不慎损坏，删除后重建：

```bash
rm -f ~/.local/share/opencode/opencode-local.db
# 下次运行 opencode 会自动重建
```

然后通过 `export` / `import` 重新迁移 session。