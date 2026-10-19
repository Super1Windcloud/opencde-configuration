# opencode-configuration

## 配置、skills 与插件的同步安装

本仓库可独立使用，安装不依赖其他配置仓库。需要 Python 3.11+、Git 和对应 CLI；OpenCode 依赖安装需要 npm。

```bash
# 仓库绝对路径执行；同步本机配置及 skills 到仓库
/path/to/opencode/sync.sh

# 查看安装计划，不修改本机、不联网安装
/path/to/opencode/install.sh --dry-run

# 安装配置、共享 skills 和插件
/path/to/opencode/install.sh

# 仅部署仓库内文件，报告尚未恢复的插件与外部依赖
/path/to/opencode/install.sh --offline
```

- `.harness.json` 声明本仓库的布局和默认同步范围；`scripts/harness.py` 随仓库分发，可独立运行。
- 原有全局指令、配置、状态栏和 hooks 保持各自布局；新增 `harness/shared/skills/` 保存共享 skills 及其配套资源。
- 安装将共享 skills 同时部署到 `~/.agents/skills/` 和 OpenCode 原生目录 `~/.config/opencode/skills/`，不依赖软链接或其他工具的插件缓存。
- `plugins.json` 记录可恢复的插件来源；有来源的插件通过真实 CLI 安装，插件附带的 skills 随插件恢复。
- 覆盖前备份到 `~/.harness-backups/<时间>/`；JSON 配置合并保留本机凭据，不清空用户技能或其他配置。
- 常见凭据字段不入库，拒绝沿目标软链接覆盖其他文件。skills 包保留原始内容，包括上游测试样例；提交前仍需检查自定义资源中的凭据。
- `--home /path/to/home` 可导出或安装到指定用户目录；正常目录软链接在导出时展开，运行时缓存、登录状态和会话不纳入同步。
- 同步采用增量覆盖，不自动删除仓库文件。第三方插件按市场当前版本恢复，不承诺锁定全部插件版本。缺少 CLI 或离线跳过的项目会逐项列出，安装失败返回非零状态。

OpenCode 旧的 `skills/testkit` 软链接会先备份，再迁移为直接可扫描的 `skills/<name>/SKILL.md`。Testkit 来源及 Git commit 记录在 `plugins.json`；npm 依赖有锁文件时执行 `npm ci`。配置中的 npm 插件由 OpenCode 启动时自动安装。

验证：

```bash
python3 -B -m unittest discover -s /path/to/opencode/tests -v
bash -n /path/to/opencode/install.sh /path/to/opencode/sync.sh
```


OpenCode 全局配置的版本化备份。用于在多台机器之间同步终端偏好、模型 Provider、MCP 服务等配置。

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
