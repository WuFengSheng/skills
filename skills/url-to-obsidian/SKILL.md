---
name: url-to-obsidian
description: >
  通过网址获取网页内容并创建为 Obsidian 笔记。支持技术文档、博客文章、GitHub README 等类型网页。
  支持可选的中文翻译（代码块、命令、包名、URL 保持不变），排版结构清晰简洁。
  优先使用 opencli web read 抓取，不可用时自动降级到 Firecrawl、fetchWebContent、fetch。
  使用前会主动收集信息：链接地址、笔记名称、存储位置、是否翻译 —— 信息不明确时提供选项让用户选择，
  用户也可一键确认默认建议。自动生成 frontmatter、标签和来源信息。
  Triggers on: 保存网址到obsidian、网页转obsidian笔记、url to obsidian、抓取网页到 obsidian、
  save url to obsidian、fetch and save to obsidian、把这个网页保存到 obsidian、
  收藏到 obsidian、保存到 obsidian 知识库、存档页面到 obsidian
allowed-tools: Bash(opencli:web read), Bash(wc:*), Bash(date:*), Bash(which:*), Bash(mkdir:*), Read, Write, Edit
---

# url-to-obsidian

将任意网址的内容抓取、清理并保存为 Obsidian 笔记。专为技术文档类内容优化（安装指南、使用说明、API 文档、README 等）。

---

## 核心工作流

```
预检查工具链 → 信息收集(URL+名称+位置+翻译) → 智能抓取 → 清理格式化 → 写入笔记 → 验证输出
```

每步完成后验证结果再进入下一步。**关键原则**：先检查再执行，避免无效等待；信息不明确时主动询问用户并提供选项。

---

## 步骤 0：工具链可用性预检查

在抓取前先检测各工具的可用性，直接选择最高优先级可用工具，**避免等待超时后再降级**。

### 检测方法

1. **opencli 检测**：
   ```bash
   opencli web read --help >/dev/null 2>&1 && echo "AVAIL" || echo "UNAVAIL"
   ```
   如果返回 "AVAIL"，直接使用 opencli（优先级 1）。

2. **如果 opencli 不可用**：跳过 opencli，直接尝试 MCP 工具。MCP 工具通过 ToolSearch 检测：
   - `mcp__firecrawl-mcp__firecrawl_scrape` → 优先级 2
   - `mcp__web-search__fetchWebContent` → 优先级 3
   - `mcp__fetch__fetch` → 优先级 4

3. **告知用户**当前使用的工具和原因：
   - 成功检测：`使用 opencli 抓取（已检测可用）`
   - 降级时：`opencli daemon 未运行，降级使用 Firecrawl`

### 如果所有工具都不可用

报告可用工具检测结果，请用户手动粘贴网页内容或检查工具安装状态。

---

## 步骤 1：信息收集（三阶段）

从用户消息中提取关键信息：**链接地址**、**笔记名称**、**存储位置**、**翻译偏好**。信息不完整时主动询问用户，提供合理选项供选择。

### 1.1 判断已有信息

从用户消息中解析已提供的信息：

| 信息项 | 提取规则 | 示例 |
|--------|---------|------|
| **链接地址 (URL)** | 匹配 `https?://` 开头的链接 | `https://example.com/doc` |
| **笔记名称** | "命名为XXX"、"标题用XXX"、"叫XXX"、引号包裹的名称 | `命名为 React 指南` |
| **存储位置** | "保存到XXX"、"存到XXX目录"、"放在XXX"、路径格式 | `保存到 ~/Obsidian/React/` |
| **标签** | "标签：a, b, c" 或 "#tag1 #tag2" | `标签：docker, tutorial` |

**快速路径判断**：如果用户消息中同时明确提供了 URL + 笔记名称 + 存储位置，直接跳到步骤 2 执行抓取，不进行询问。

### 1.2 阶段一：确保 URL

**URL 已提供** → 进入阶段二。

**URL 未提供** → 反问用户：

```
请提供要保存的网页链接地址。例如：
• 一篇技术文档的 URL
• 一个 GitHub 仓库地址
• 一篇博客文章链接
```

**多个 URL** → 逐个处理，每个 URL 独立走完整流程（独立询问名称和位置）。

### 1.3 阶段二：预抓取 + 目录探测

当用户未同时提供"笔记名称"和"存储位置"时，先做准备工作：

**预抓取标题**（不做完整翻译和格式化，仅提取标题）：
1. 按步骤 0 检测工具链，选择可用工具
2. 快速抓取网页内容（不执行步骤 3 的完整清理流程）
3. 从抓取内容中提取标题：
   - 匹配第一个 `# 标题`（Markdown H1）
   - 或从 `<title>` 标签提取
   - 或从 URL 路径最后一段推断
4. 对标题做基础清理：去除尾部站点名（如 ` | Example Site`）、去除多余空格

**存储位置选项生成**：
1. 默认提供常用路径作为选项（不扫描 vault）：
   - `~/Obsidian/` 根目录（默认）
   - `~/Obsidian/技术文档/`
   - `~/Obsidian/收藏/`
2. **仅在用户明确要求**（如"帮我看看有哪些目录"、"列出已有的文件夹"）时，才调用 `obsidian_list_files_in_vault` 探查目录结构

### 1.4 阶段三：结构化询问

使用 `AskUserQuestion` 工具一次性询问用户。根据阶段二的探测结果动态生成选项。

**问题 1：笔记名称**（如果用户未指定名称）

| 选项 | 内容 | 说明 |
|------|------|------|
| 选项 1（推荐） | 从网页标题生成的名称 | 阶段二预抓取的标题，干净简洁 |
| 选项 2 | 从 URL 路径推断的名称 | 如 URL 为 `react.dev/learn/installation`，建议 `React 安装指南` |
| 选项 3 | "自定义名称" | 用户选择后在 Other 中输入自定义名称 |

**问题 2：存储位置**（如果用户未指定位置）

| 选项 | 内容 | 说明 |
|------|------|------|
| 选项 1（默认） | `~/Obsidian/ 根目录` | 直接保存在 vault 根目录 |
| 选项 2-N | vault 中已有的子目录 | 从阶段二探测结果中选取，如 `~/Obsidian/技术文档/` |
| 最后选项 | "自定义目录" | 用户选择后在 Other 中输入自定义路径 |

**问题 3：是否翻译为中文**（始终询问，除非用户在消息中已明确指示）

| 选项 | 内容 | 说明 |
|------|------|------|
| 选项 1（默认） | "保留原文不翻译" | 保持网页原始语言，不做翻译处理 |
| 选项 2 | "翻译为简体中文" | 将正文翻译为简体中文（代码块、命令、URL保持原样） |

**AskUserQuestion 调用示例**：

```
问题1 header: "笔记名称"
  - "React 18 安装与使用指南"（根据网页标题生成，推荐）
  - "react-installation-guide"（根据 URL 路径推断）
  - "自定义名称..."
问题2 header: "存储位置"
  - "~/Obsidian/ 根目录"（默认）
  - "~/Obsidian/React 文档/"（已有目录）
  - "自定义目录..."
问题3 header: "翻译选项"
  - "保留原文不翻译"（默认）
  - "翻译为简体中文"
```

**注意**：
- 如果用户已指定了名称，问题 1 跳过，只问位置和翻译
- 如果用户已指定了位置，问题 2 跳过，只问名称和翻译
- 如果用户在消息中已明确指示翻译偏好（如"翻译成中文"、"不用翻译"），问题 3 跳过
- 三者都已指定 → 快速路径，不询问直接执行

### 1.5 信息确认

询问完成后，向用户展示确认摘要：

```
确认信息：
  🔗 链接：https://example.com/doc
  📝 笔记名称：React 18 安装指南
  📁 存储位置：~/Obsidian/React 文档/
  🌐 翻译：保留原文 / 翻译为简体中文
```

然后进入步骤 2 开始抓取。

---

## 步骤 2：抓取网页内容（智能降级链）

按预检查结果选择工具。以下为完整优先级表：

### 优先级 1：opencli web read（推荐）

```bash
opencli web read --url "<url>" --stdout
```

- **优势**：支持 JS 渲染、输出干净 Markdown、处理 SPA 页面
- **前提**：opencli daemon 运行 + Chrome 扩展已连接（`opencli doctor` 绿色）
- **额外参数**（按需使用）：
  - `--wait <秒>` — 页面加载后额外等待（默认 3 秒，内容多的页面可设为 5-8 秒）
  - `--wait-for "<CSS选择器>"` — 等待特定元素出现后再抓取
  - `--wait-until networkidle` — 等待网络空闲
- **降级条件**：daemon 未运行、扩展未连接、命令执行超时、返回错误
- **降级时记录原因**：如 `opencli daemon 未运行，自动降级到 Firecrawl`

### 优先级 2：Firecrawl scrape

使用 ToolSearch 加载 `mcp__firecrawl-mcp__firecrawl_scrape`，然后调用：
```
url: "<url>"
formats: ["markdown"]
onlyMainContent: true
```

- **优势**：功能全面、支持 JS 渲染（waitFor 参数）、自动提取主体内容
- **降级条件**：MCP 工具不可用、API 配额耗尽、返回错误
- **降级时记录原因**

### 优先级 3：fetchWebContent

使用 ToolSearch 加载 `mcp__web-search__fetchWebContent`，然后调用：
```
url: "<url>"
readability: true
maxChars: 50000
```

- **优势**：轻量级，适合静态文档页面
- **限制**：不支持 JS 渲染
- **降级时记录原因**

### 优先级 4：fetch（兜底）

使用 ToolSearch 加载 `mcp__fetch__fetch`，然后调用：
```
url: "<url>"
max_length: 10000
```

- **限制**：仅返回简化内容，上限 10000 字符
- **全部失败**：报告错误（列出失败原因），询问用户是否手动提供内容

### 特殊情况：GitHub URL

如果 URL 匹配 `github.com/*/*`（仓库 README 页面），**跳过降级链**，直接使用：

使用 ToolSearch 加载 `mcp__web-search__fetchGithubReadme`，然后调用。

如果 fetchGithubReadme 不可用，再走正常降级链。

---

## 步骤 3：内容清理与格式化

### 3.1 提取标题

按优先级尝试：
1. 从抓取内容中匹配第一个 `# 标题`（Markdown H1）
2. 从内容开头提取 `<title>` 标签文本
3. 从 URL 路径最后一段推断（去除扩展名、替换连字符为空格）
4. 如果以上都失败，使用域名作为标题

**标题清理规则**：
- 去除尾部站点名（如 ` | Example Site`、` - Blog Name`）
- 去除多余空格和特殊符号（如 `®`、`™`）
- 限制 120 字符

### 3.2 内容清理

- **去除冗余元数据**：移除抓取工具添加的元数据头（如 opencli 的 `# 站点名\n> 作者: ...\n> 原文链接: ...` 块）
- **去除页面噪音**：移除网页中残留的交互标签和导航残留，包括但不限于：
  - `Copy link`、`复制链接` 等复制按钮文字
  - `Terminal`、`bash`、`text`、`css`、`tsx` 等单独成行的代码块标签（保留代码块内的语言标记如 ` ```bash `）
  - 底部导航菜单链接（如页脚"Docs | Components | Blog"等）
  - 社交媒体链接列表
- **去除重复标题**：如果内容第一行 H1 与提取的标题相同，保留一个即可
- **保留代码块**：确保 ` ``` ` 代码块完整，语法高亮标记不丢失
- **保留表格**：Markdown 表格格式保持
- **链接处理**：相对链接转为绝对链接（基于原始 URL 的 base）
- **去除广告**：如果内容包含明显的广告文本，尽量清理

### 3.3 边界感知截断

对大内容按边界截断，避免破坏 Markdown 结构：

| 内容大小 | 处理方式 |
|---------|---------|
| ≤ 80KB | 完整保存 |
| 80KB-200KB | 在最近的双换行（段落边界）处截断，优先保证完整段落 |
| > 200KB | 在最近的 H2 标题边界处截断，优先保证章节完整性 |

截断后添加标记：
```markdown
> [!warning] 内容过长已截断
> 原始约 X KB，保存约 Y KB。完整内容请访问原始 URL: <url>
```

**禁止在以下位置截断**：代码块内部、表格中间、列表中间。

### 3.4 技术文档优化

识别到技术文档特征时（包含代码块、命令行示例、API 端点等）：
- 确保 `bash`、`shell`、`json`、`yaml` 等代码块语言标记正确
- 保留安装命令的完整性（`npm install`、`pip install`、`brew install`、`git clone` 等）
- 如内容包含步骤编号（1. 2. 3.），保持其格式

### 3.5 中文化翻译（按需执行）

**仅在步骤 1 中用户选择了"翻译为简体中文"时才执行此步骤**。默认保留原文不翻译。

**翻译原则**：

| 需要翻译 | 保持原样 |
|---------|---------|
| 标题和章节名 | 代码块内容（含注释） |
| 描述性段落和说明文字 | 命令行和终端命令 |
| 表格中的文字说明 | 包名、库名（如 `@astryxdesign/core`） |
| 列表项的描述文字 | URL 链接地址 |
| Callout/提示框中的文字 | 文件名和路径（如 `app/page.tsx`） |
| 表格列头 | API 名称、组件名、函数名 |
| 技术术语的解释 | 语言/框架名称（React、Next.js、StyleX 等） |

**翻译质量标准**：
- 使用自然流畅的中文表达，避免生硬的直译
- 技术术语保持英文（如 CSS、API、CLI），但上下文用中文说明
- 代码块上方的描述性标签（如 `globals.css`、`app/page.tsx`）保留原样，下方的说明文字翻译
- 章节标题简洁有力，如 "Getting Started" → "快速开始"，"Install" → "安装"

---

## 步骤 4：确定存储位置与文件名

### 4.1 文件名

直接使用步骤 1 中用户确认的**笔记名称**作为文件名。

安全处理（对用户输入做基本清理）：
1. 移除非法字符：`/` `:` `\` `*` `?` `"` `<` `>` `|`
2. 去除首尾空格和点号
3. 如果处理后为空，使用 `未命名-<时间戳>`

**命名风格**：保留中文和英文混合，不强制翻译。尊重用户的选择。

### 4.2 存储位置

直接使用步骤 1 中用户选择的**存储位置**：

```
默认：~/Obsidian/<用户指定名称>.md
指定目录：~/Obsidian/<用户选择的目录>/<用户指定名称>.md
自定义路径：~/Obsidian/<用户自定义路径>/<用户指定名称>.md
```

- 目录不存在时自动创建（使用 `mkdir -p`）
- 用户选择"存档模式"时 → 保存到 `.raw/articles/<文件名>-<YYYY-MM-DD>.md`

### 4.3 冲突处理

如果目标文件已存在：
1. 比较内容是否相同（相同则跳过）
2. 内容不同时，询问用户选择：
   - **覆盖**：替换原文件
   - **追加**：在现有文件末尾添加新内容，用 `---` 分隔
   - **重命名**：文件名加 `-2`、`-3` 后缀
   - **归档**：保存到 `.raw/articles/<slug>-<日期>.md`，原文件不动
   - **跳过**：不保存

---

## 步骤 5：生成并保存笔记

### 5.1 笔记模板

**标准技术文档模板**（翻译状态取决于用户选择）：

```markdown
---
title: "<中文标题>"
source: "<原始URL>"
created: <YYYY-MM-DD>
fetch_tool: "<opencli | firecrawl | fetchWebContent | fetch | fetchGithubReadme>"
tags:
  - web-clip
  - <域名标签>
  - <内容标签1>
  - <内容标签2>
---

# <中文标题>

> [!note]- 来源信息
> - **URL**: <原始URL>
> - **抓取时间**: <YYYY-MM-DD>
> - **抓取工具**: <工具名>

<中文正文内容>
```

**GitHub README 模板**（fetch_tool 为 fetchGithubReadme 时）：

```markdown
---
title: "<仓库名> - <描述（翻译为中文）>"
source: "<GitHub URL>"
created: <YYYY-MM-DD>
fetch_tool: "fetchGithubReadme"
tags:
  - web-clip
  - github
  - readme
  - <语言标签>
---

# <仓库名>

> [!note]- 来源信息
> - **仓库**: <GitHub URL>
> - **抓取时间**: <YYYY-MM-DD>
> - **抓取工具**: fetchGithubReadme

<README 正文（翻译为中文）>
```

**排版简洁原则**：
- 正文直接从 H2 章节开始，无引言段落
- 章节标题简洁（2-6 字），如"安装"、"快速开始"、"示例应用"
- 不保留"内容大小"字段（已在来源信息中体现）
- 来源信息使用折叠 callout（`[!note]-`），默认收起减少视觉干扰

### 5.2 标签自动生成规则

- **必加标签**：`web-clip`
- **来源域名标签**（从 URL 域名推断）：

| 域名特征 | 标签 |
|---------|------|
| `github.com` | `github` |
| `npmjs.com` / `nodejs.org` | `nodejs` |
| `pypi.org` | `python` |
| `docs.rs` / `crates.io` | `rust` |
| `docker.com` / `hub.docker.com` | `docker` |
| `medium.com` / `dev.to` | `blog` |
| `*.readthedocs.io` / `docs.*` | `documentation` |

- **内容关键词标签**（从标题和内容提取，最多 5 个）：
  - 技术名词：如 `react`、`api`、`docker`、`cli`、`kubernetes`
  - 文档类型：`tutorial`、`installation`、`guide`、`reference`、`readme`、`changelog`
  - 编程语言：`javascript`、`python`、`rust`、`go`、`typescript`
- **用户自定义标签**：如果用户指定了标签（如"标签：docker, tutorial"），追加到列表

### 5.3 frontmatter 格式规范（强制执行）

严格遵循 Obsidian YAML frontmatter 格式，**这是不可协商的规则**：

- ✅ **标签必须用多行列表格式**：
  ```yaml
  tags:
    - web-clip
    - docker
  ```
- ❌ **禁止 inline 格式**：`tags: [web-clip, docker]` ← 绝对不会使用
- ✅ **日期格式**：`YYYY-MM-DD`（如 `2026-07-02`）
- ❌ **禁止 ISO datetime**：`2026-07-02T00:00:00Z` ← 错误格式
- ✅ **值包含特殊字符时加引号**：`title: "包含:冒号的标题"`
- ✅ **纯平 YAML**，不允许嵌套对象
- ✅ **fetch_tool 字段**：记录使用的抓取工具，便于追溯

### 5.4 写入操作与验证

**写入**：使用 `Write` 工具直接写入绝对路径：
```
Write: ~/Obsidian/<目录>/<文件名>.md
```

**写入后验证（必须执行）**：
1. 使用 `Read` 回读文件的前 20 行，检查：
   - `tags:` 字段是否使用了多行列表格式（绝对不能是 inline `[a, b]`）
   - `created:` 日期格式是否为 `YYYY-MM-DD`
   - `source:` 和 `fetch_tool:` 字段是否存在
   - 来源信息 callout 是否完整
2. 如果验证失败，使用 `Edit` 工具修正格式，再次验证
3. 检查文件大小是否合理（不为空、不被意外截断）

---

## 步骤 6：执行后输出

完成后向用户报告：

```
✅ 笔记已创建：~/Obsidian/<目录>/<文件名>.md
   - 标题：<标题>
   - 来源：<URL>
   - 抓取工具：<工具名><降级说明>
   - 内容大小：<约 X>KB
```

多 URL 时汇总报告所有结果（成功/失败/跳过的数量和详情）。

---

## 错误处理一览

| 场景 | 处理方式 |
|------|---------|
| 预检查：所有工具不可用 | 报告检测结果，请用户手动粘贴内容 |
| obsidian MCP 工具不可用 | 使用固定常用路径列表（~/Obsidian/、~/Obsidian/技术文档/ 等），告知用户 |
| AskUserQuestion 超时/取消 | 使用默认名称（网页标题）和默认位置（~/Obsidian/ 根目录），告知用户可按需调整 |
| 用户选择的自定义路径不存在 | 自动创建目录（mkdir -p），然后继续保存 |
| opencli daemon 未运行 | 自动降级到 Firecrawl，告知原因 |
| Firecrawl API 不可用 | 降级到 fetchWebContent，告知原因 |
| 所有抓取工具都失败 | 报告错误，列出失败的 URL 和原因，不创建空笔记 |
| URL 返回 404/403 | 告知用户该 URL 无法访问，不重试 |
| 需要登录才能访问 | 告知用户该页面需要登录，建议手动复制内容后保存 |
| 网页内容为空/仅导航 | 告知用户内容质量差，询问是否仍保存 |
| 内容超过 80KB | 按边界感知截断策略处理，标注截断位置 |
| 磁盘空间不足 | 报告错误，不创建笔记 |
| frontmatter 格式验证失败 | 用 Edit 修正，重新验证直到通过 |

---

## 与其他技能的关系

- **claude-obsidian:save**：保存当前对话，本技能保存外部网页
- **claude-obsidian:wiki-ingest**：深度知识整合（提取实体/概念/交叉引用）。互补用法：
  - 快速存档 → `url-to-obsidian`
  - 深度整合 → 先用 `url-to-obsidian` 保存到 `.raw/articles/`，再 `wiki-ingest` 处理
- **claude-obsidian:defuddle**：如 `defuddle-cli` 已安装，可在步骤 3 可选使用
- **claude-obsidian:obsidian-markdown**：本技能遵循其 wikilinks、callouts、frontmatter 规范
- **opencli-usage**：了解 opencli 全局标志和适配器发现

---

## 注意事项

- **务必**在保存前使用 AskUserQuestion 收集确认笔记名称和存储位置（除非用户已在消息中明确提供）
- **务必**提供合理的默认选项（标题建议名称 + 常用目录），确保用户可一键确认不过度负担
- **务必**提供合理的默认存储路径（~/Obsidian/ 根目录 + 常用子目录），仅在用户明确要求时才扫描 vault 目录结构
- **如果** obsidian MCP 工具不可用，使用固定常用路径列表作为降级方案并告知用户
- **不要**在步骤 1 已经提供了名称/位置的情况下重复询问（快速路径）
- **绝对不要**在 frontmatter 中使用 inline 列表（`tags: [a, b]`），始终用多行格式
- **绝对不要**在 frontmatter 中使用嵌套对象，保持纯平 YAML
- **绝对不要**在代码块或表格中间截断内容
- **绝对不要**在用户未选择翻译的情况下自动翻译内容——翻译必须是用户明确选择后的行为
- **务必**在步骤 1 的 AskUserQuestion 中提供"保留原文"和"翻译为中文"两个选项，默认"保留原文"
- **务必**在写入后回读验证 frontmatter 格式
- **务必**在预检查阶段确定可用工具链，避免无效等待
- **务必**告知用户降级发生时使用的具体工具和原因
- **务必**清理页面噪音：去除 "Copy link"、代码块标签、导航页脚等残留
- **不要**修改 Obsidian vault 中已有的不相关文件
- **不要**对非 GitHub 的 URL 使用 fetchGithubReadme
- **不要**创建空的笔记文件
- **不要**保留冗余的引言段落，正文直接从 H2 章节开始
