# 小鸭头 Second Brain Plus

小鸭头帮你收藏——一个用于 AI Agent 的第二大脑 Skill，自动归档你的灵感和好内容。

> by 袅袅 ([@hoitcao](https://github.com/hoitcao))

## 功能

- **智能判断**：自动判断用户消息是否值得存入知识库
- **内容抓取**：集成 DeepReader，支持 Twitter/X、Reddit、YouTube 和任意网页；抓取失败时自动降级
- **直接写入**：判定值得收藏后立即写入，无需用户确认，减少交互摩擦
- **线索匹配**：自动匹配已有思考线索（Thread），生成定制化洞察并追加
- **双轨存储**：Inbox 快速记录 + 完整文章存档
- **去重检测**：URL 标准化后自动去重，已有内容可选择补充新洞察或重新存储
- **线索摘要**：线索累积满 5 条自动生成认知摘要，帮助快速回顾

## 工作原理

```
用户消息 → 提取 URL → DeepReader 抓取 → 生成洞察 → 匹配线索 → 直接写入全部文件 → 展示简报
                ↓
         无 URL 时 → 判断是否为值得记录的想法 → 直接写入 → 展示简报
```

**核心原则**：判定值得收藏后，立即执行所有写入操作（inbox、存档、线索追加），然后向用户展示简要的存档报告。不需要用户确认。

## 安装

### 方式一：使用 npx skills（推荐）

```bash
npx skills add hoitcao/second-brain-plus
```

### 方式二：手动安装

1. 下载本仓库的 `SKILL.md` 到你的 skills 目录：
   ```
   ~/.openclaw/skills/second-brain-plus/SKILL.md
   ```

2. 确保已安装 DeepReader Skill（用于内容抓取）

## 配置

在使用前，请确保配置以下路径：

| 配置项 | 说明 | 配置位置 |
|--------|------|----------|
| `KNOWLEDGE_BASE` | 知识库根目录 | AGENTS.md 或环境变量 |
| `INBOX_PATH` | Inbox 文件夹路径 | AGENTS.md 或环境变量 |
| `ARCHIVE_DIR` | 完整文章存档目录 | AGENTS.md 或环境变量 |

### 示例目录结构

```
<WORKSPACE>/
├── knowledge/          ← KNOWLEDGE_BASE
│   ├── inbox/          ← INBOX_PATH（每日 Inbox 条目）
│   │   ├── 2026-03-10-001.md
│   │   └── 2026-03-10-002.md
│   └── threads/        ← 思考线索
│       ├── AI与人类协作模式.md
│       └── 产品设计中的减法哲学.md
├── archive/            ← ARCHIVE_DIR（完整文章存档）
└── skills/
    └── second-brain-plus/
        └── SKILL.md
```

## 使用方法

1. **分享链接**：在对话中发送任意 URL，AI 自动抓取、提炼、写入
2. **分享想法**：直接发送你的灵感、反思、感悟，AI 自动判断并收藏
3. **自动完成**：无需确认，收藏完成后展示简报（标题、核心洞察、关联线索）
4. **按需调整**：如果对结果不满意，回复修改意见即可直接更新

### 收藏完成后的简报示例

```
✅ 已收藏：How to Do Great Work

💡 核心洞察：伟大的工作来源于好奇心驱动的深度探索，而非刻意规划……

📂 已写入：
- Inbox: 2026-03-10-001.md
- 存档: How-to-Do-Great-Work_20260310_143022.md

🔗 关联线索：
- [[注意力经济与深度工作]] — 该文从创作者视角论证了……
- [[写作作为思考工具]] — Paul Graham 的写作即思考实践……
```

## 依赖

- [OpenClaw](https://github.com/openclaw/openclaw) - AI Agent 运行时
- DeepReader Skill - 内容抓取（需另行安装）

## 致谢

- 遵循 [Agent Skills Standard](https://agentskills.io)
- 架构灵感来自 OpenClaw

## License

MIT
