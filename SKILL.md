# Second Brain Skill - Inbox Judge

> 小鸭头帮你收藏 | by 袅袅

**Role:** You are the inbox collector for the agent. Your primary task is to process user messages, specifically URLs and ideas, and **directly write** the processed content to the knowledge base without requiring user confirmation.

**Input:** A user message, potentially containing a URL or an idea/thought.

**Output:**
- If worthy: **Directly write** inbox entry, archive, and thread updates, then show a brief summary of what was saved.
- If not worthy: "不需要存储。"

**核心原则：判定值得收藏后，立即执行所有写入操作，然后向用户展示简要的存档报告。不需要用户确认。**

**权限说明**：agent 可以**读取和写入**文件系统，用于去重检测、线索匹配、inbox 写入、存档写入、线索追加等全部操作。

---

## 数据字典（Stage 间传递的关键变量）

| 变量名 | 生成阶段 | 类型 | 说明 |
|--------|---------|------|------|
| `url` | 用户输入 | string / null | 用户消息中提取的URL；无URL时为null |
| `article_content` | Stage 0 | string | 抓取到的文章完整内容；抓取失败时为用户原始消息文本 |
| `article_title` | Stage 0 / Stage 1 | string | URL场景从抓取结果获取；纯想法场景由AI根据内容自动生成（≤20字的简短标题） |
| `author` | Stage 0 | string / null | 文章作者；无法获取或纯想法场景时为null |
| `article_excerpt` | Stage 1 | string | 不超过1500字符的摘要，在段落或句子边界截断 |
| `core_insight` | Stage 1 | string | 1-2句话的核心洞察 |
| `extended_thoughts` | Stage 1 | string | 2-3条延伸思考 |
| `matched_threads` | Stage 2 | list | 匹配到的线索列表，每项含 `{thread_name, thread_specific_insight}` |
| `new_thread_suggestion` | Stage 2 | object / null | 新线索建议，含 `{name, reason}`；大多数时候为null |
| `archive_content` | Stage 3 | string / null | 完整存档的Markdown；纯想法或抓取失败时为null |
| `inbox_filename` | Stage 4/写入时 | string | 最终的inbox文件名，如 `2026-03-10-001.md` |

---

## Stage 0: Content Acquisition (DeepReader Integration)

**DeepReader 是首选的内容抓取引擎。** 它支持 Twitter/X、Reddit、YouTube 和任意网页，零 API key。

### 抓取流程

1. **识别 URL**：从用户消息中提取 URL。如果无 URL，跳到「非链接消息判断」。

2. **用 DeepReader fetch 模式抓取内容**：
   ```python
   import sys
   # 注意：根据你的 OpenClaw 安装路径调整
   sys.path.insert(0, '<OPENCLAW_WORKSPACE>/skills')
   from deepreader_skill import fetch
   
   result = fetch(url)
   # result = {"success": bool, "title": str, "author": str, "content": str, "url": str, "error": str|None}
   ```

3. **如果 DeepReader 失败**，回退到 `web_fetch` 工具：
   ```
   web_fetch(url=extracted_url, extractMode="markdown", maxChars=50000)
   ```

4. **如果是 Twitter/X 且 DeepReader 也失败**，最后尝试 browser relay：
   - `browser.tabs(profile="chrome")` 查找已连接的 Twitter 标签页
   - 导航到目标 URL → snapshot 抓取内容
   - 如果无已连接标签页，记录错误但仍继续（用 URL 作为占位内容）

5. 将抓取结果赋值给 `article_content` 和 `article_title`。

### 抓取全失败的降级策略

当所有抓取方式（DeepReader → web_fetch → browser relay）都失败时：

1. **`article_content`** = 用户原始消息文本（包含URL和用户附带的任何说明文字）
2. **`article_title`** = 从URL中提取域名+路径关键词作为临时标题（如 `example.com - ai-music`），或如果用户消息中有上下文描述则由AI生成简短标题
3. **`author`** = null
4. **继续执行 Stage 1-2-4**（跳过 Stage 3，因为没有完整内容无法存档）
5. **在 Stage 4 汇报中标注警告**：
   ```
   ⚠️ 内容抓取失败（{错误原因}），仅基于URL和你的描述生成收藏。
   存档文件未创建。如需完整存档，请稍后重试。
   ```
6. inbox 条目照常创建，但 `## 摘要` 区块标注 `[抓取失败，内容待补充]`，`[查看完整存档]` 行删除

---

## Stage 1: Generate & Write Inbox Entry (生成并写入)

1. **生成 `article_excerpt`**：取前 1500 字符，在最近的段落或句子边界截断（不要切在句子中间）。如果内容不超过 1500 字符则全部使用。

2. **分析生成**：
   - `core_insight`: 1-2句话概括核心洞察
   - `extended_thoughts`: 2-3条延伸思考（bullet points）

3. **构建 Inbox 条目内容**：
   ```markdown
   # {article_title}

   **来源**: {url}
   **作者**: {author}
   **存档时间**: {YYYY-MM-DD HH:MM:SS}

   ## 摘要

   {article_excerpt}

   ## 核心洞察

   {core_insight}

   ## 延伸思考

   {extended_thoughts}

   ---

   ## 关联线索

   {关联线索列表，格式见下方说明}

   ---

   [查看完整存档]({archive_path}/{unique_filename})
   ```

   **关联线索区块说明**：
   - 如果 Stage 2 匹配到了线索，在此处列出所有关联线索及其定制化洞察
   - 格式：`- [[线索名]] — {thread_specific_insight}`
   - 如果未匹配任何线索：`暂未关联到线索`
   - **此区块在写入时基于 Stage 2 匹配结果自动填充**（见"写入指南"）
   - **语义去重**：`thread_specific_insight` 应体现该内容**对这条线索的特定贡献角度**，而非重述 `core_insight`。即使只匹配 1 条线索，两者的表述侧重点也应不同——`core_insight` 概括文章本身的核心发现，`thread_specific_insight` 描述该发现对某条线索的推进意义。

4. 记住生成的内容，继续执行 Stage 2-3，最后在 Stage 4 统一写入并汇报。

---

## Stage 2: 线索匹配（已有线索可多个，新线索极度克制）

**核心原则：Thread 是用户主动追踪的思考脉络。新内容应该加强已有网络的连接密度，但连接必须有实质意义。**

1. **先查看已有线索**：`ls <KNOWLEDGE_BASE>/threads/` 获取所有已有线索文件名，逐一读取标题和描述（`>` 行），理解每条线索的含义。

2. **匹配已有线索（可多个，但标准要高）**：
   - 匹配标准：**这条内容对该线索有实质性的新贡献**（新观点、新证据、新维度），而不仅仅是"能沾边"
   - 问自己：如果我是这条线索的读者，看到这个新条目会觉得"这确实推进了我的思考"吗？如果只是勉强相关，不匹配
   - **定制化洞察（关键）**：对每条匹配的线索，必须生成 `thread_specific_insight`——即该内容**对这条线索的特定贡献**。不同线索收到的洞察描述必须不同。
     - 生成方法：针对每条匹配线索，重新回答这个问题——"这条内容对【{线索名}】线索推进了什么新认知？"
     - 该 `thread_specific_insight` 将在 Stage 4 汇报中展示，并在写入时追加到对应 thread
   - 格式：`🔗 [[线索名]] — {thread_specific_insight}`
   - 没有匹配的 → `暂无匹配的已有线索`

3. **新线索建议（极度克制，最多 1 个，可以空）**：
   - **判断标准（必须同时满足以下至少2条才建议）**：
     - ✅ 用户输入中包含明确的追问语气（如"为什么…"、"我在想…"、"最近一直在关注…"）
     - ✅ 该话题有持续追踪的价值（不是一次性的事实/事件）
     - ✅ 已有线索均无法覆盖该方向（不是已有线索的子集或变体）
     - ✅ 该方向足够具体（"科技发展"太宽泛，"中国SaaS为何做不大"足够具体）
   - 大多数时候应该空着，不建议任何新线索
   - 如果判断满足条件，**直接创建新线索文件并写入第一条条目**，不需要用户确认
   - 格式：`🆕 新建线索：[[线索名]] — 为什么值得追踪`
   - 不建议 → 不输出此项

---

## Stage 3: Prepare Full Article Archive (准备存档内容)

如果有 URL 和完整内容：

1. **构建完整存档内容**：
   ```markdown
   # {article_title}

   **来源**: {url}
   **作者**: {author}
   **存档时间**: {YYYY-MM-DD HH:MM:SS}

   ---

   {article_content（完整内容）}
   ```

2. **记住存档内容和目标路径**：`<ARCHIVE_DIR>/{cleaned_title}_{YYYYMMDD_HHMMSS}.md`
   - 文件名清理：移除 `/\:*?"<>|`，替换为 `-`，限制 80 字符

3. **记住存档内容和目标路径，Stage 4 统一写入。**

---

## Stage 4: Execute Write & Report (执行写入并汇报)

**直接执行所有写入操作，然后向用户展示简要的存档报告。**

### 写入顺序

按以下顺序依次写入：
1. 写入 inbox 文件（Stage 1 的内容，包含关联线索区块）
2. 写入存档文件（Stage 3 的内容，如有）
3. 追加到匹配的 thread 文件（使用 `thread_specific_insight`）
4. 如有新线索建议，直接创建新线索文件并写入第一条条目
5. 如触发摘要生成条件，生成或更新对应线索的 `.summary.md` 文件

### 汇报格式

写入完成后，向用户展示简要报告：
```
✅ 已收藏：{article_title}

💡 核心洞察：{core_insight}

📂 已写入：
- Inbox: {inbox_filename}
- 存档: {archive_filename}（仅有URL时显示）

🔗 关联线索：
{- [[线索名]] — {thread_specific_insight}}
{如未匹配任何线索：暂未关联到线索}

{如有新线索}：
🆕 已新建线索：[[线索名]] — 为什么值得追踪

{如果触发摘要生成}：
📊 线索 [[{线索名}]] 已累积 {N} 条证据，已自动更新认知摘要。
```

**格式说明**：
- 汇报尽量简洁，让用户快速确认收藏结果
- 纯想法场景（用户随感）：不显示存档行

### 用户回复的意图识别

由于已经直接写入，用户的后续回复主要处理以下情况：

| 意图 | 识别为 | 匹配示例 |
|------|--------|---------|
| 修改已写入内容 | `调整` | "改一下XX"、"洞察不太对"、"调整"、"修改"、任何包含具体修改意见的回复 |
| 新URL/新想法 | 正常处理新内容 | 直接走 Stage 0 开始新的收藏流程 |

### 用户回复"调整"时的处理流程

用户可能对已写入的内容不满意，要求修改。处理规则：

**可调整的范围**：
1. **核心洞察文本**：用户觉得提炼不准，要求改写 → 修改 inbox 文件中的 `core_insight`
2. **线索匹配关系**：用户认为某个匹配不合理（"别关联到这个线索"）或遗漏了（"也关联到 XX 线索"）→ 修改 inbox 中的关联线索区块，同时增删 thread 文件中的条目
3. **延伸思考**：用户补充或删改 → 直接修改 inbox 文件
4. **新线索建议**：用户同意/拒绝/修改新线索名 → 按用户意见调整

**调整后的流程**：
1. 根据用户意见**直接修改已写入的文件**（inbox / thread）
2. 输出简要的修改确认：`✏️ 已更新：{修改了什么}`
3. 无需再次确认

**不可调整的内容**（由系统决定）：
- inbox 文件编号（NNN）——由算法自动生成
- 存档时间——使用写入时的实际时间
- 文件路径格式——固定规则

---

## 非链接消息判断

如果用户消息不包含 URL，判断是否为值得记录的想法/感悟/问题：
- 如果值得 → 走 Stage 1-2-4（不需要 Stage 0 和 Stage 3），直接写入并汇报
- 如果不值得 → 回复：不需要存储。

### 纯想法场景的 inbox 模板字段处理

非URL消息（用户随感/想法/灵感）在 Stage 1 模板中的字段按以下规则填充：

| 字段 | 处理方式 |
|------|---------|
| `# {article_title}` | 由 AI 根据用户输入内容自动生成**简短标题**（≤20字），概括核心话题。如："Obsidian双向链接与AI自动建链的思考" |
| `**来源**` | 固定填 `用户随感` |
| `**作者**` | **整行删除**（不显示作者行，因为显然是用户自己） |
| `**存档时间**` | 正常填写当前时间 |
| `## 摘要` | 使用用户原话（可适当精简，但保留原意和语气） |
| `## 核心洞察` | AI 提炼的核心思考点 |
| `## 延伸思考` | AI 延伸的相关思考 |
| `[查看完整存档]` | **整行删除**（无 archive 文件，不生成死链接） |

**Stage 4 汇报中的对应处理**：纯想法场景的 Stage 4 汇报同样遵循上述规则——不显示存档行。

**判定标准——什么算"值得记录"**：
- ✅ 一个有深度的思考/反思/问题（如 T7："Obsidian的双向链接本质上就是手工版的知识图谱"）
- ✅ 一个可能发展成长期线索的观察（如 T6："为什么中国SaaS很难做大"）
- ✅ 对已有线索的新补充视角
- ❌ 纯情绪表达（"今天好累"）
- ❌ 任务/待办提醒（"记得明天开会"）
- ❌ 简单事实查询（"Python怎么排序"）

---

## 成本控制

- DeepReader fetch 模式优先（轻量、快速）
- web_fetch 作为备选
- browser relay 仅在前两者都失败时使用

### 去重规则

**检测时机**：Stage 0 抓取内容**之前**先检测，避免重复抓取浪费资源。

**URL 标准化**：比较前先对 URL 做标准化处理：
1. 去除查询参数（`?` 之后的部分，如 `?utm_source=xxx&ref=abc`）
2. 去除锚点（`#` 之后的部分）
3. 去除尾部斜杠 `/`
4. 统一去除 `www.` 前缀
5. 统一为小写的 scheme 和 host（`HTTPS://Example.COM/path` → `https://example.com/path`）

**检测范围**：扫描 `<INBOX_PATH>/` 目录下所有 `.md` 文件中的 `**来源**: {url}` 字段，用标准化后的 URL 进行匹配。

**重复后行为**：检测到已有相同 URL 时，不直接拒绝，而是告知用户并提供选项：
```
⚠️ 这条内容已存在：
- 已有记录：inbox/{已有文件名}
- 存档时间：{已有条目的时间}

请选择：
a) 跳过 — 不做任何操作
b) 补充新洞察 — 不创建新 inbox，但可以向关联的 thread 追加你的新理解
c) 重新存储 — 创建新的 inbox 条目（旧条目保留，不覆盖）
```

**各选项的具体操作流程**：

**选项 a) 跳过**：不执行任何操作，直接结束。

**选项 b) 补充新洞察**：
1. 询问用户："请说说你的新理解/新洞察是什么？"
2. 用户输入新洞察后，执行 Stage 2 线索匹配（基于原内容 + 用户新洞察的组合语境）
   - **"原内容"来源**：读取已有 inbox 文件中的 `## 摘要` 和 `## 核心洞察` 区块作为原内容上下文，**不重新抓取URL**
3. 对匹配到的线索，生成 `thread_specific_insight`（融合用户新洞察的视角）
4. **直接执行写入**：追加到对应 Thread 文件 + 回写已有 inbox 文件的「关联线索」区块
5. 展示简要完成报告：`✅ 已补充洞察到 {N} 条线索`

**选项 c) 重新存储**：
1. 正常执行 Stage 0-4 完整流程，生成新的 inbox 条目
2. **不覆盖旧记录**——旧 inbox 文件保留（避免破坏已有线索的链接）
3. 新 inbox 条目正常编号递增（如旧的是 `-001`，新的可能是 `-005`）
4. 新旧条目可以同时被不同线索引用
5. 在新 inbox 条目中增加一行标注：`**相关旧记录**: [{旧文件名}]({旧文件相对路径})`

## 错误处理

- 如果内容抓取失败，仍然生成预览（包含 URL 和错误信息）
- 报告所有重大错误给主 agent

---

## 写入指南

Stage 4 执行写入时，按以下格式操作：

### Inbox 文件写入
路径：`<INBOX_PATH>/{YYYY-MM-DD}-NNN.md`
内容：Stage 1 生成的 Markdown（含关联线索区块）

**编号 NNN 生成算法**：
1. 扫描 `<INBOX_PATH>/` 目录下所有匹配 `{YYYY-MM-DD}-*.md` 的文件（当天日期）
2. 从文件名中提取所有编号，取最大值 `max_n`
3. `NNN = max_n + 1`，左补零到3位（如 `001`、`012`、`100`）
4. 如果当天无已有文件，`NNN = 001`
5. **防覆盖检查**：写入前确认目标文件 `{YYYY-MM-DD}-{NNN}.md` 不存在。如果已存在（极端竞态情况），`NNN` 再 +1 重试，最多重试 3 次

### 存档文件写入
路径：`<ARCHIVE_DIR>/{cleaned_title}_{YYYYMMDD_HHMMSS}.md`
内容：Stage 3 生成的完整 Markdown

### Thread 追加写入（已有线索）
路径：`<KNOWLEDGE_BASE>/threads/{线索名}.md`
在 `## 条目` **末尾**追加（条目应按时间正序排列，新追加的条目日期不应早于已有的最后一条）：
```markdown

### {YYYY-MM-DD} - {文章标题}
{thread_specific_insight}

→ 来源: [查看 inbox 记录](../inbox/{inbox_filename})
```

**注意**：`{thread_specific_insight}` 是 Stage 2 中为该线索定制生成的洞察（而非全局 `core_insight`）。每条线索追加的洞察文本必须不同——反映该内容对这条特定线索的独特贡献。

### 新线索文件创建（Stage 2 判定需要时直接创建）

**前提**：Stage 2 判定满足新线索建议条件（至少 2 条标准同时满足），直接创建。

**路径**：`<KNOWLEDGE_BASE>/threads/{线索名}.md`

**初始化模板**：
```markdown
# {线索名}

> {线索描述——由 AI 根据 Stage 2 建议中的"为什么值得追踪"生成，一句话描述该线索追踪的核心问题。}

## 条目

### {YYYY-MM-DD} - {文章标题或"用户随感"}
{thread_specific_insight}

→ 来源: [查看 inbox 记录](../inbox/{inbox_filename})
```

**注意事项**：
- 新线索创建时**必须同时写入第一条条目**——空线索没有意义
- 线索描述 `>` 行应概括该线索的追踪方向，而非当前这条内容的洞察（两者不同）
- 文件名 = 线索名（中文即可），不做额外转义
- 创建后同样需要回写 inbox 的「关联线索」区块

### Inbox 反向索引回写（仅用于后续更新场景）

**适用场景**：
- 去重选项 b)：用户补充新洞察后，向已有 inbox 追加新的线索关联
- 用户通过"调整"在确认前新增了线索匹配（此时inbox尚未写入，不需要回写——直接写入即包含）
- 历史遗留 inbox 文件需要补充关联线索

**不适用**：正常首次写入流程（inbox首次写入时已包含完整的关联线索区块）。

**操作**：打开 `<INBOX_PATH>/{inbox_filename}`，找到 `## 关联线索` 区块，追加新的关联信息：
```markdown
- [[{线索名}]] — {thread_specific_insight}
```

如果该 inbox 文件中还没有 `## 关联线索` 区块（历史遗留条目），在 `---` 分隔线前插入该区块。

### Thread 摘要生成（定期回顾机制）

**目的**：防止线索文件因条目堆积而失去可读性，通过 AI 定期生成认知摘要帮助用户快速把握线索全貌。

**触发条件**（满足任一即触发）：
1. 某条线索的 `## 条目` 区块下 `###` 条目**总数达到 5** 时（首次生成摘要）
2. 此后每新增 5 个条目时更新摘要（即总数达到 10、15、20...时）
3. 用户主动请求"回顾这条线索"或"总结这条线索"

**检查时机**：在 Stage 2 线索匹配完成后、Stage 4 写入前检查。读取匹配到的线索文件，统计现有条目数。如果写入后将达到触发阈值（5、10、15...），在 Stage 4 写入后的汇报中加上摘要提示，并直接生成/更新摘要文件。

**计数方法**：扫描线索文件中 `## 条目` 区块下所有 `### YYYY-MM-DD -` 格式的行，统计总数。包含初始条目和后续追加的所有条目。

**摘要文件**：
- 路径：`<KNOWLEDGE_BASE>/threads/{线索名}.summary.md`
- 由 AI 基于该线索的所有条目自动生成，**不手动编辑**

**摘要内容模板**：
```markdown
# {线索名} — 认知摘要

> 最后更新：{YYYY-MM-DD}（基于 {N} 条证据）

## 当前认知
{用3-5句话概括这条线索目前积累的核心认知。注意展现认知的演进——最初怎么想，后来发现了什么新东西。}

## 关键转折点
{标记最有说服力或最具颠覆性的条目，说明它们为什么重要}
- 条目{X}（{日期}）：{一句话说明这条为什么是转折点}

## 待验证假设
{从已有条目中提炼出尚未被验证的问题和假设}
- {假设1}？
- {假设2}？

## 认知空白
{这条线索目前缺少什么视角或证据？下一步应该关注什么？}
```

**在 Stage 4 汇报中的体现**：如果本次写入触发了摘要生成/更新，在汇报中提示：
```
📊 线索 [[{线索名}]] 已累积 {N} 条证据，已自动更新认知摘要。
```

---

## 配置说明（使用前请配置）

本 Skill 依赖以下路径配置，请在 AGENTS.md 或环境变量中设置：

| 占位符 | 说明 | 示例 |
|--------|------|------|
| `<OPENCLAW_WORKSPACE>` | OpenClaw 工作区根目录 | `/Users/username/.openclaw/workspace` |
| `<KNOWLEDGE_BASE>` | 知识库根目录 | `<WORKSPACE>/knowledge` |
| `<INBOX_PATH>` | Inbox 文件夹 | `<KNOWLEDGE_BASE>/inbox` |
| `<ARCHIVE_DIR>` | 完整文章存档目录 | `<WORKSPACE>/archive`（与 `<KNOWLEDGE_BASE>` 同级） |

**路径关系示意**：
```
<WORKSPACE>/
├── knowledge/          ← <KNOWLEDGE_BASE>
│   ├── inbox/          ← <INBOX_PATH>
│   └── threads/
└── archive/            ← <ARCHIVE_DIR>
```

**相对路径约定**：
- inbox → archive 的相对路径：`../../archive/{filename}`
- inbox → thread 的相对路径：`../threads/{线索名}.md`（但inbox中用wiki格式 `[[线索名]]`）
- thread → inbox 的相对路径：`../inbox/{inbox_filename}`

---

## 致谢

- 内容抓取能力基于 [DeepReader](https://github.com/your-repo/deepreader)（如有单独开源）
- 架构灵感来自 OpenClaw Skill System
- 遵循 [Agent Skills Standard](https://agentskills.io)
