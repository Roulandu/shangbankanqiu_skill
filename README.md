# 上班看球 (Shàngbān Kànqiú)

> 上班摸鱼看球神器 — 让 AI Agent 帮你实时播报比赛赛况到飞书/钉钉群。

## 这是什么？

一个跨平台的 AI Agent Skill，安装后可以让你的 AI Coding Agent（OpenCode、Claude Code、Codex、Cursor、OpenClaw、Trae 等）化身实时比赛播报员，自动搜索赛况并通过群机器人推送到你的工作群。

- 🔍 自动搜索正在进行的比赛（Stage A+ 精准搜索中文实时源，减少进球延迟检测）
- 📊 每 2 分钟更新一次赛况（文字直播模式默认 90 秒）
- 📋 比分 + 实时事件（进球/换人/犯规/关键球/红黄牌）+ 🎤 文字直播模式
- ⚽ 世界杯足球优先：可选 API provider 优先获取实时比分、半场状态、关键事件和更细比赛动作
- 🧭 有扩展事件源时支持“谁持球推进、长传找谁、谁解围”等更像文字直播的播报
- 🛟 没有 API key 时自动回退到原有 web_search / webfetch 模式
- ⏸️ 足球中场休息会继续轮询，不会被误当终场
- 📲 通过飞书/钉钉群机器人推送
- 🏀 NBA · ⚽ 世界杯

## 快速开始

### 1. 复制 Skill

将 `shangbankanqiu/` 目录复制到你的 Agent 技能目录：

| 平台 | 技能目录 |
|------|----------|
| OpenCode | `~/.agents/skills/` |
| Claude Code | `~/.claude/skills/` |
| Codex | `~/.codex/skills/` |
| Cursor | `.cursor/skills/` |
| OpenClaw | `~/.openclaw/skills/` |
| Trae | `.trae/skills/` |

### 2. 配置群机器人

在飞书或钉钉群中添加自定义机器人，获取 Webhook 地址。

### 3. 填写配置

```bash
cd shangbankanqiu
cp config.example.json config.json
# 编辑 config.json，填入 webhook 地址和密钥
```

### 4. 使用

在 Agent 中说：

```
使用 shangbankanqiu skill
帮我看下现在有什么NBA比赛
世界杯有比赛吗
```

## 目录结构

```
├── README.md                    # 本文件
└── shangbankanqiu/
    ├── SKILL.md                 # Skill 定义文件
    ├── config.example.json      # 配置文件模板
    └── README.md                # 详细使用文档
```

## License

MIT
