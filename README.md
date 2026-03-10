# 小鸭头 Second Brain Plus

小鸭头帮你收藏——一个用于 AI Agent 的第二大脑 Skill，自动归档你的灵感和好内容。

> by 袅袅 ([@hoitcao](https://github.com/hoitcao))

## 功能

- **智能判断**：自动判断用户消息是否值得存入知识库
- **内容抓取**：集成 DeepReader，支持 Twitter/X、Reddit、YouTube 和任意网页
- **生成预览**：不直接写入，先生成预览供用户确认
- **线索匹配**：自动匹配已有思考线索，建议关联
- **双轨存储**：Inbox 快速记录 + 完整文章存档

## 工作原理

```
用户消息 → 提取 URL → DeepReader 抓取 → 生成预览 → 等待确认 → 写入文件
                ↓
         无 URL 时 → 判断是否为值得记录的想法 → 生成预览
```

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
| `INBOX_PATH` | Inbox 文件夹路径 | AGENTS.md 或环境变量 |
| `ARCHIVE_DIR` | 完整文章存档目录 | AGENTS.md 或环境变量 |
| `KNOWLEDGE_BASE` | 知识库根目录 | AGENTS.md 或环境变量 |

### 示例目录结构

```
~/.openclaw/workspace/
├── knowledge/
│   ├── inbox/          # 每日 Inbox 条目
│   │   ├── 2026-02-22-001.md
│   │   └── 2026-02-22-002.md
│   └── threads/        # 思考线索
│       ├── ai-research.md
│       └── reading-notes.md
└── skills/
    └── second-brain-plus/
        └── SKILL.md    # 本文件
```

## 使用方法

1. **分享链接**：在对话中发送任意 URL
2. **分享想法**：直接发送你的想法、感悟
3. **AI 处理**：自动抓取内容、生成预览
4. **确认写入**：回复"写入"完成归档

## 依赖

- [OpenClaw](https://github.com/openclaw/openclaw) - AI Agent 运行时
- DeepReader Skill - 内容抓取（需另行安装）

## 致谢

- 遵循 [Agent Skills Standard](https://agentskills.io)
- 架构灵感来自 OpenClaw

## License

MIT
