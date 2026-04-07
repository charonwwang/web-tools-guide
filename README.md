# web-tools-guide

Web 工具策略 skill，指导 Agent 在获取网络信息时按场景选择最优工具。

## 背景

Agent 在选择 web 工具时，通常采用 ReAct 范式，由模型根据任务自主判断使用哪类工具。但实际使用中存在两个问题：

1. **工具选择不当**：Agent 倾向于直接调用 `web_search` 或 `browser`，忽略了中间的结构化工具（opencli），导致效率低或操作出错
2. **失败处理不当**：当 `web_search` API 未配置时，Agent 会静默降级到 fetch/browser，跳过配置引导，用户永远不知道可以配置更好的搜索体验

本 skill 解决这两个问题：定义清晰的分支决策模型，并规范每种工具的失败处理流程。

## 核心模型：分支决策（不是层级）

四个工具按场景分支选择，不是从上到下的优先级层级：

```
web_search  → 没有 URL，需要搜索信息        ─┐ 主选（按场景直选）
web_fetch   → 已知 URL，静态内容             ─┘
opencli     → 以上失败？CLI 结构化访问 70+ 站 ─  Fallback（优先于 browser）
browser     → 全都不行？浏览器自动化          ─  兜底（最后手段）
```

先按场景选 `web_search` 或 `web_fetch`；失败时先试 `opencli`，最后才上 `browser`。

## opencli 安装与配置

opencli 需要三个组件配合工作：CLI 本体、Browser Bridge 浏览器插件、Chrome 浏览器。

### 一键安装（推荐）

运行 skill 内置的安装脚本，自动完成所有步骤（幂等，可重复运行）：

```bash
bash skills/web-tools-guide/scripts/setup-opencli.sh
```

该脚本会自动：
1. 安装 opencli CLI（`npm install -g @jackwener/opencli`）
2. 从 GitHub Releases 下载 Browser Bridge 插件（`opencli-extension.zip`）
3. 更新 systemd 服务并重启浏览器加载插件

### 手动安装

如果需要手动安装，按以下步骤操作：

#### 1. 安装 CLI

```bash
npm install -g @jackwener/opencli
```

#### 2. 下载 Browser Bridge 插件

Browser Bridge 插件**不在 npm 包内**，需要从 GitHub Releases 单独下载：

```bash
# 从 GitHub Releases 下载预编译的 extension
curl -fsSL -o /tmp/opencli-extension.zip \
  https://github.com/jackwener/opencli/releases/latest/download/opencli-extension.zip

# 解压到指定目录
mkdir -p /root/.openclaw/opencli-extension
unzip -qo /tmp/opencli-extension.zip -d /root/.openclaw/opencli-extension
rm -f /tmp/opencli-extension.zip
```

解压后目录中应包含 `manifest.json`（Chrome 扩展必备文件）。

#### 3. 让 Chrome 加载插件

需要在 Chrome 启动参数中添加插件加载：

```bash
# extension 的绝对路径
EXTENSION_PATH=/root/.openclaw/opencli-extension

# Chrome 启动参数中添加：
--disable-extensions-except=$EXTENSION_PATH --load-extension=$EXTENSION_PATH
```

如果 Chrome 是通过 systemd 服务管理的（如 OpenClaw 的 `lighthouse-chromium.service`），修改服务文件的 `ExecStart` 行添加以上参数，然后：

```bash
systemctl daemon-reload
systemctl restart lighthouse-chromium  # 替换为你的服务名
```

#### 4. 验证

```bash
opencli doctor
```

三项全部显示 OK 即为成功：

```
CLI:        OK
Daemon:     OK
Extension:  OK
```

如果 Extension 显示 `not connected`，检查：
- `manifest.json` 是否存在于 extension 目录中（Step 2 下载是否成功）
- Chrome 是否正确加载了插件（Step 3 参数是否生效）
- Chrome 是否已重启

## 文件结构

```
web-tools-guide/
├── README.md                              # 本文件：背景和架构说明
├── SKILL.md                               # 核心策略文件（description 注入 system prompt）
│                                          #   - 分支决策流程图
│                                          #   - 四个工具的使用时机和操作指引
│                                          #   - web_search 失败处理流程
├── scripts/                               # 安装脚本目录
│   └── setup-opencli.sh                   # opencli 一键安装脚本（幂等）
│                                          #   - 安装 CLI + 下载插件 + 重启浏览器加载插件
└── references/                            # 按需加载的参考文件（避免 system prompt 膨胀）
    ├── opencli-guide.md                   # opencli 详细使用指引
    │                                      #   - 渐进式发现（--help 链）
    │                                      #   - 场景映射、输出格式、写操作注意
    ├── web-search-config.md               # web_search API 配置引导流程
    │                                      #   - Step 1: 向用户展示配置选项（Tavily/Kimi）
    │                                      #   - Step 2-5: 接收 Key → 确认 → 配置 → 重启
    └── well-known-sites.json              # 常用网站 URL 索引（K-V JSON）
                                           #   - 搜索引擎、社交平台、新闻、开发工具等
                                           #   - 降级搜索和 browser 登录操作时使用
```

**设计原则**：
- SKILL.md 只负责决策逻辑，不内联配置话术或网站 URL 数据
- references 文件通过 `read {baseDir}/references/xxx` 按需加载，避免 token 浪费
- opencli 通过 `--help` 渐进式发现命令，不需要记住所有命令
- web_search 失败 → 引导配置 → 用户拒绝 → 才降级，不允许静默跳过

## AGENTS.md 中的对应规则

本 skill 在 `AGENTS.md` 中有配套引导区块，位于 **Session Startup 最后、Memory 之前**（被 `<!-- WEB-TOOLS-STRATEGY-START/END -->` 注释包裹）：

```markdown
<!-- WEB-TOOLS-STRATEGY-START -->
### Web Tools Strategy (CRITICAL)

**Before using web_search/web_fetch/browser/opencli, you MUST `read workspace/skills/web-tools-guide/SKILL.md`!**

**Four tools, branch by scenario (NOT a hierarchy):**
web_search  -> No URL, need to search info         ─┐
web_fetch   -> Known URL, static content            ─┤ Primary (pick by scenario)
                                                     │
opencli     -> Either fails? CLI structured access  ─┤ Fallback (try before browser)
browser     -> All above fail? Full browser control ─┘ Last resort

**When web_search/web_fetch fail**: try `opencli` first (70+ sites, `opencli --help` to discover). Only escalate to `browser` when opencli also can't handle it.

**When web_search errors: You MUST read the skill's "web_search failure handling" section first, guide user to configure search API. Only fall back after user explicitly refuses.**
<!-- WEB-TOOLS-STRATEGY-END -->
```

这段规则的作用是**在 Agent 每次会话启动时建立分支决策的意识**。具体的操作指引、失败处理流程、opencli 用法等细节由 SKILL.md 和 references 文件提供。

### 为什么需要两层引导？

| 位置 | 触发时机 | 作用 |
|------|----------|------|
| **SKILL.md description** | 每条消息（注入 system prompt 的 `<available_skills>` XML） | 让 AI 在看到 web 相关任务时触发读取 SKILL.md |
| **AGENTS.md 区块** | 每次会话启动（Session Startup 流程读取） | 建立工具策略全局意识，强化"先读 skill 再调工具" |
| **SKILL.md 正文** | skill 被触发后按需读取 | 提供完整决策流程和操作指引 |

单靠 SKILL.md description 可能不够（AI 看到内置工具描述后可能直接调用），所以在 AGENTS.md 中额外加了显式引导，双重保障。
