# 上班看球 (Shàngbān Kànqiú) — 实时比赛播报 Skill

> 上班摸鱼看球神器！让 AI Agent 帮你实时播报比赛赛况到飞书/钉钉群。

## 这是什么？

一个跨平台的 AI Agent Skill（技能文件），安装后可以让你的 AI Coding Agent（如 OpenCode、Claude Code、Codex、Cursor 等）化身实时比赛播报员：

- 🔍 自动搜索正在进行的比赛
- 📊 每 3 分钟更新一次赛况
- 📋 不只是比分——还会推送实时事件（进球、换人、犯规、关键球、红黄牌等），让你不在场也能跟上比赛节奏
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
  "poll_interval_seconds": 180,    // 轮询间隔（秒），默认3分钟
  "max_polls": 60,                 // 单次最多轮询多少次（防无限循环），默认 60
  "max_consecutive_failures": 3,   // 连续失败几次后停止轮询
  "state_file": ".shangbankanqiu_state.json", // 去重用临时文件
  "language": "zh-CN",             // 播报语言

  // === 实时事件相关（commentary）===
  "include_events": true,          // 是否在比分基础上附带实时事件（进球/换人/犯规等），默认 true；设为 false 则退化为纯比分播报
  "max_events_per_msg": 6,         // 每次推送最多附带的事件条数（建议 4-8；钉钉 markdown 字段约 20KB 硬上限，过多易被截断）
  "commentary_dedup_window": 30    // 事件去重环形缓冲「记多少轮」（注意是轮数，不是事件条数）。实际容量 = window × max_events_per_msg，默认 30 × 6 = 180 条 id_hash（state 文件约 9KB）
}
```

至少配置飞书或钉钉中的一个。

## ⚠ 设计原则：本 Skill 不会后台化

很多人误以为「每 3 分钟推送一次」就需要后台 daemon / cron / 计划任务。本 Skill **故意不那样设计**：

- **轮询循环跑在宿主 Agent 的当前对话里**，sleep 是**前台同步阻塞**（`Start-Sleep` / `sleep`）
- **不会**让 Agent 创建 `.py` / `.sh` / `.ps1` 调度脚本
- **不会**用 `nohup` / `&` / `setInterval` / `crontab` 等任何后台化手段

这样设计的好处：
1. 跨 Agent 一致——codex / cursor / opencode / claude code 走出同样的对话流
2. 易于停止——你在该对话里说「停止」就能终止（最长等待约 60 秒，因为 sleep 按 60 秒切片执行）
3. 不污染系统——不留下 cron 条目、不留下 daemon 进程、不抢占端口
4. 你的 codex/cursor/opencode/claude code **应用本身**（在新对话/新窗口）不受影响，依然能正常用来写代码

如果你需要 Agent 在轮询的同时干别的活，请新开一个 Agent 实例/对话；**不要**改本 Skill 让它后台化。

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
搜索当前比赛状态  → 顺手嗅探 play-by-play / 文字实录 URL（Stage A）
    ↓
┌─ 已完结 → 发送最终赛果 → 结束
├─ 未开始 → 发送赛事前瞻 → 结束
└─ 正在直播
       ↓
   [若 include_events=true 且 Stage A 拿到 URL]
        webfetch 抓取该页 + LLM 提取最新事件（Stage B）
       ↓
   渲染消息（比分 + 事件列表，事件可为空）
       ↓
   推送到飞书 / 钉钉
       ↓
   等待 3 分钟 → 重新搜索 → 循环直到比赛结束
```

> Stage A 失败（嗅探不到 URL）或 Stage B 失败（webfetch 超时 / 解析失败）时，事件列表退化为空，**比分照常推送**——不会因为抓不到 commentary 就漏掉这一轮播报。

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
