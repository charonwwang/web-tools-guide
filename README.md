# web-tools-guide

Web 工具三级策略指南 skill，指导 Agent 在获取网络信息时选择最优工具路径。

## 背景

我们调研了主流 AI Agent 产品，发现 Agent 在选择工具时，通常采用 ReAct 范式，由模型根据任务自主判断使用哪类工具。

其 Browser Use 能力大致遵循"从轻到重、从通用到具体"的工具使用方式：

1. 优先使用 **search** 工具，快速获取目标信息入口，适用于没有明确 URL 的场景；
2. 使用 **fetch** 工具直接请求已知 URL，适用于静态信息获取场景，如新闻文章、RSS、API、静态文档等；
3. 使用 **browser** 工具处理更复杂的网页操作，适用于需要登录、点击、截图，以及依赖 JS 渲染、登录态或页面交互的场景。

**目前问题最大的是场景 3（browser）**——Agent 在处理 JS 渲染页面、登录操作、多步交互时容易出错（等待策略不当、登录态丢失、跳过用户确认等）。

此外还有一个常见问题：**当 `web_search` 不可用时，Agent 会静默降级到 fetch/browser，跳过搜索 API 配置引导**，导致用户永远不知道可以配置更好的搜索体验。

## 设计原则

- **SKILL.md 只负责决策逻辑**，不内联具体的配置话术或网站 URL 数据
- **references 文件通过 `read {baseDir}/references/xxx` 按需加载**，避免 system prompt 膨胀
- **web_search 失败 → 引导配置 → 用户拒绝 → 才降级**，不允许静默跳过

## 文件结构

```
web-tools-guide/
├── README.md                              # 本文件：设计背景和架构说明
├── SKILL.md                               # 核心策略文件（注入 system prompt）
│                                          #   - 三级工具链决策流程
│                                          #   - Level 1/2/3 选择标准和操作指引
│                                          #   - web_search 失败处理流程
└── references/
    ├── web-search-config.md               # web_search API 配置引导流程
    │                                      #   - Step 1: 向用户展示配置选项（Tavily/Kimi）
    │                                      #   - Step 2-5: 接收 Key → 确认 → 配置 → 重启
    └── well-known-sites.json              # 常用网站 URL 索引（K-V JSON）
                                           #   - 搜索引擎、社交平台、新闻、开发工具等
                                           #   - 降级搜索和 browser 登录操作时使用
```

## AGENTS.md 中的对应规则

本 skill 在 `AGENTS.md` 的 Session Startup 部分有如下引用，确保 Agent 在每次会话启动时就知道三级策略的存在：

```markdown
### 🌐 Web 工具策略 (CRITICAL)

**⚠️ 当需要使用 web_search/web_fetch/browser 时，必须先 `read workspace/skills/web-tools-guide/SKILL.md`！**

**三级工具：**
```
web_search  → 无明确 URL 时关键词搜索（最轻量）
web_fetch   → 已知 URL 获取静态内容（文章/文档/API）
browser     → JS 渲染/登录态/页面交互（最重量）
```

**web_search 失败时：必须读取 skill 中的"web_search 失败处理"章节，引导用户配置搜索 API。只有用户明确拒绝后才能降级。**
```

这段规则的作用是**在 system prompt 层面建立三级策略的意识**，具体的操作指引和配置流程则由本 skill 的 SKILL.md 和 references 文件提供。

## 迭代记录

- **v1.0**：原名 `browser-use-web-search-guide`，只包含 web_search API 配置引导
- **v2.0**：重构为三级工具策略指南，新增 references 目录、well-known-sites.json、完整决策流程
- **v2.1**：精简 SKILL.md（332行→126行），删除冗余禁止列表和前置检查，强化 Level 3 browser 操作指引，修复 references 文件路径（使用 `{baseDir}` 模板变量）
