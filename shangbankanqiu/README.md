# 上班看球 (Shàngbān Kànqiú) — 实时比赛播报 Skill

> 上班摸鱼看球神器！让 AI Agent 帮你实时播报比赛赛况到飞书/钉钉群。

## 这是什么？

一个跨平台的 AI Agent Skill（技能文件），安装后可以让你的 AI Coding Agent（如 OpenCode、Claude Code、Codex、Cursor 等）化身实时比赛播报员：

- 🔍 自动搜索正在进行的比赛
- 📊 每 3 分钟更新一次赛况
- 📲 通过飞书/钉钉群机器人推送到你的工作群
- 🏀 支持 NBA、⚽ 世界杯（更多赛事陆续支持中）

## 快速开始

### 1. 复制 Skill 到你的 Agent 技能目录

不同平台的技能目录位置不同，选择你使用的平台：

| 平台 | 技能目录 |
|------|----------|
| OpenCode | `~/.agents/skills/` |
| Claude Code | `~/.claude/skills/` |
| Codex | `~/.codex/skills/` |
| Cursor | `.cursor/skills/` (项目目录下) |
| OpenClaw | `~/.openclaw/skills/` |
| Trae | `.trae/skills/` (项目目录下) |

```bash
# 示例：安装到 OpenCode
cp -r shangbankanqiu ~/.agents/skills/
```

### 2. 配置群机器人

#### 飞书

1. 打开飞书 → 目标群聊 → 设置 → 群机器人 → 添加自定义机器人
2. 安全设置建议选「签名校验」，记下 Webhook 地址和签名密钥
3. 参考文档: https://open.feishu.cn/document/client-docs/bot-v3/add-custom-bot

#### 钉钉

1. 打开钉钉 → 目标群聊 → 设置 → 智能群助手 → 添加自定义机器人
2. 安全设置建议选「加签」，记下 Webhook 地址和加签密钥(SEC)
3. 参考文档: https://open.dingtalk.com/document/orgapp/custom-bot-send-message-type

### 3. 填写配置文件

```bash
cd shangbankanqiu
cp config.example.json config.json
# 编辑 config.json，填入你的 webhook 地址和密钥
```

### 4. 开始使用

在你的 AI Coding Agent 中输入：

```
使用 shangbankanqiu skill
```

然后按提示选择比赛类型即可。

或者直接说：

```
帮我看下现在有什么NBA比赛
世界杯有比赛吗
```

## 配置文件说明

```jsonc
{
  "feishu": {
    "webhook_url": "飞书机器人 Webhook 地址",
    "secret": "签名密钥（没有则留空）"
  },
  "dingtalk": {
    "webhook_url": "钉钉机器人 Webhook 地址",
    "secret": "加签密钥 SEC（没有则留空）"
  },
  "poll_interval_seconds": 180,  // 轮询间隔（秒），默认3分钟
  "language": "zh-CN"            // 播报语言
}
```

至少配置飞书或钉钉中的一个。

## 支持的赛事

| 赛事 | 状态 |
|------|------|
| 🏀 NBA | ✅ 已支持 |
| ⚽ 世界杯 (FIFA World Cup) | ✅ 已支持 |
| ⚽ 英超 | 🔜 计划中 |
| ⚽ 欧冠 | 🔜 计划中 |
| 🏈 NFL | 🔜 计划中 |

## 工作原理

```
用户触发 Skill
    ↓
询问比赛类型（NBA / 世界杯）
    ↓
搜索当前比赛状态
    ↓
┌─ 已完结 → 发送最终赛果 → 结束
├─ 未开始 → 发送赛事前瞻 → 结束
└─ 正在直播 → 发送实时播报 → 等待3分钟 → 重新搜索 → 循环直到比赛结束
```

## 常见问题

**Q: 需要 API Key 吗？**
A: 不需要。Skill 利用 AI Agent 自带的 web search 能力搜索比赛数据，不需要任何第三方 API。

**Q: 支持哪些 AI Agent 平台？**
A: OpenCode、Claude Code、Codex、Cursor、OpenClaw、Trae 等所有支持 SKILL.md 规范的平台。

**Q: 轮询间隔可以改吗？**
A: 可以，修改 `config.json` 中的 `poll_interval_seconds`，建议不低于 60 秒。

**Q: 可以同时推送到飞书和钉钉吗？**
A: 可以，两个都配置就会同时推送。

**Q: 如何停止播报？**
A: 在 Agent 对话中说「停止」「结束」或「stop」即可。

## 文件结构

```
shangbankanqiu/
├── SKILL.md           # 核心 Skill 定义文件
├── config.example.json # 配置文件模板
└── README.md          # 本文件
```

## License

MIT
