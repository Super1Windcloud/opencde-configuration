# opencode-configuration

## Testkit 完整依赖与 Figma 安装

本仓库可独立部署 OpenCode。默认一并安装 Testkit 完整 skills、Python/移动端/浏览器/性能/视频/逆向工具依赖，并同步 Figma 插件安装来源。需要 Python 3.11+、Git；macOS 系统包通过已有 Homebrew 补齐。full 需要 Node 24+、npm 10+，缺少时尝试由 Homebrew 准备。

```bash
~/.config/opencode/sync.sh
~/.config/opencode/install.sh --dry-run
~/.config/opencode/install.sh
~/.config/opencode/install.sh --dependencies core
~/.config/opencode/install.sh --offline
```

- 普通安装默认 `--dependencies full`；core 仅准备核心 Python 和 mobile-mcp；none 跳过运行环境/MCP 构建。
- 默认安装并登记 mobile-mcp，full 另含 Playwright MCP；使用 SDK 和独立运行目录的实际路径，不只复制 JSON。
- 插件安装允许列表为 Testkit、Figma；其他本机 skills 不同步到 GitHub，也不卸载。插件缓存、依赖缓存和登录凭据不入库。
- 运行环境位于 `~/.local/share/harness-testkit/`；全局指令会补充 `source .../env.sh` 的使用约定。
- 覆盖前备份；保留原有设置、凭据与其他 MCP。`--dry-run` 无写入和下载；`--offline` 不联网安装，只部署配置并报告待安装项。
- Xcode、设备信任、SDK 许可、WDA 签名、Midscene 模型凭据和 Figma OAuth 仍需本机准备，安装失败会返回非零状态。

详见 [完整依赖与 MCP 配置](runtime/README.md)、[实际验证范围](VALIDATION.md)。验证命令：`python3 -B -m unittest discover -s ~/.config/opencode/tests -v`。

## 目录结构

```
.
├── .gitignore      # 排除可重装依赖与敏感/本机文件
├── cli.json        # 终端 Settings（主题、键位、会话呈现、diff、通知等）
└── opencode.json   # 服务端 / 项目 Config（模型、Provider、MCP、Agent、权限等）
```

两个文件是**互相独立的两套配置面**，作用域不同，不可混放：

| 文件 | 安装位置 | 读取方 | 作用 |
| --- | --- | --- | --- |
| `cli.json` | `~/.config/opencode/cli.json` | 终端客户端 | 仅终端 UI 偏好，全局唯一，无项目级；未知字段会被拒绝 |
| `opencode.json` | `~/.config/opencode/opencode.json` | 后台 service | 模型、Provider、MCP、Agent、权限、格式化等 |

例如主题写进 `opencode.json` 不生效，`permissions` 写进 `cli.json` 也不生效。

## cli.json（Settings）

`$schema` 固定为 `https://opencode.ai/v2/cli.json`。当前启用的配置：

| 分组 | 配置 | 效果 |
| --- | --- | --- |
| `theme.name` | `nord` | 主题 |
| `session.sidebar` | `hide` | 隐藏会话侧栏 |
| `session.verbosity` | `high` | 会话记录最详细 |
| `session.permissions` | `autoaccept` | ⚠️ 自动接受所有权限请求，不再逐条询问 |
| `diffs.wrap` | `word` | diff 按词折行 |
| `prompt` | `image_preview` / `paste: full` | 图片预览、大段粘贴全量展示 |
| `terminal` | `title` / `copy: manual` | 更新窗口标题、手动复制 |
| `scroll.acceleration` | `true` | 连滚加速 |
| `attention.sound` | `true` | 开启提示音 |
| `animations` / `highlights` | `true` | 动画与更新亮点提示 |

可在 TUI 内按 `Ctrl+P` → **Open settings** 修改；文件改动会被热重载。
临时覆盖可用：`OPENCODE_CLI_CONFIG_CONTENT='{"tabs":{"mode":"off"}}' opencode`。

## opencode.json（Config）

`$schema` 固定为 `https://opencode.ai/config.json`。当前配置：

- **默认模型**：`opencode/deepseek-v4.1-flash-free`
- **Provider**
  - `xai`：`baseURL` = `https://aigo.hair/v1`，模型 `grok-4.6`
  - `google`：`baseURL` = `https://aigo.hair/v1`，模型 `gemini-3.8-flash`
- **MCP**：`mobile-mcp`（本地 stdio，`npx -y @mobilenext/mobile-mcp@latest`），并注入 `PATH` / `ANDROID_HOME` / `ANDROID_SDK_ROOT` 以访问 Android SDK
- **username**：`super`

> 该文件使用的是 V1 形态字段名（`provider`、`options`、`mcp.<name>`、`enabled`），
> OpenCode V2 会读取并在内存中归一化，功能正常。如需迁移到 V2 原生写法，对应关系为
> `provider`→`providers`、`options`→`settings`、`mcp.<name>`→`mcp.servers.<name>`、`enabled`→`disabled`（取反）。

## 必需的凭据（环境变量）

配置文件不保存密钥，所有 `apiKey` 均为 `{env:...}` 引用，需在 shell 中提供：

| 变量 | 用途 |
| --- | --- |
| `XAI_API_KEY` | `xai` provider 凭据 |
| `GOOGLE_GENERATIVE_AI_API_KEY` | `google` provider 凭据 |

写入 `~/.zshrc`（或等价 shell 配置）：

```sh
export XAI_API_KEY=...
export GOOGLE_GENERATIVE_AI_API_KEY=...
```

未设置时会被替换为空字符串，对应 provider 将因缺少凭据而不可用。
除 `{env:NAME}` 外，也可用 `{file:path/to/secret}` 从文件读取，路径相对配置文件目录或使用绝对/`~/` 路径。

## 在新机器上还原

```sh
git clone git@github.com:Super1Windcloud/opencde-configuration.git ~/.config/opencode
~/.config/opencode/install.sh
# 再按上一节配置两个环境变量，重开终端
```

若目标目录已存在，改为克隆到临时目录后拷贝 `cli.json`、`opencode.json` 到 `~/.config/opencode/`。

验证：

```sh
opencode --version
opencode service restart
opencode api get /api/info
```

## 安全约定

以下内容**刻意不纳入仓库**（见 `.gitignore`）：

| 排除项 | 原因 |
| --- | --- |
| `service.json` | 含本机后台服务密码，属运行时状态 |
| `*.bak*` | 历史备份文件 |
| `node_modules/`、`vendor/` | 插件及 skill 运行时依赖，可重装；依赖声明和锁文件纳入版本控制 |

新增配置前请确认没有把明文密钥、令牌或本机绝对路径写进受版本控制的文件。
