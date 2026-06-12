---
name: shangbankanqiu
description: 上班看球 — 实时比赛播报机器人，支持世界杯/NBA等赛事，通过飞书/钉钉群机器人推送赛况
argument-hint: "<比赛类型: 世界杯 | NBA>"
---

# 上班看球 (Shàngbān Kànqiú) — 实时比赛播报

## 概述

上班看球是一个跨平台的 AI Agent Skill，让用户在上班时也能通过飞书群机器人或钉钉群机器人实时接收比赛赛况播报。

**支持的平台**: OpenCode、Claude Code、Codex、Cursor、OpenClaw、Trae 等所有支持 SKILL.md 规范的 AI Coding Agent。

**当前支持的赛事**: 世界杯 (FIFA World Cup)、NBA

## 前置配置

在首次使用前，用户需要配置至少一个群机器人的 Webhook。请按以下步骤操作：

### 飞书群机器人配置

1. 打开飞书，进入目标群聊
2. 点击群设置 → 群机器人 → 添加机器人 → 自定义机器人
3. 配置机器人名称和头像（建议: "上班看球播报员"）
4. **安全设置**：建议选择「签名校验」或「IP 白名单」
5. 复制 Webhook 地址，格式为: `https://open.feishu.cn/open-apis/bot/v2/hook/xxxxxxxxx`
6. 如果启用了签名校验，请保存好「签名密钥」

参考文档: https://open.feishu.cn/document/client-docs/bot-v3/add-custom-bot

### 钉钉群机器人配置

1. 打开钉钉，进入目标群聊
2. 点击群设置 → 智能群助手 → 添加机器人 → 自定义机器人
3. 配置机器人名称（建议: "上班看球播报员"）
4. **安全设置**：建议选择「加签」
5. 复制 Webhook 地址，格式为: `https://oapi.dingtalk.com/robot/send?access_token=xxxxxxxxx`
6. 如果启用了加签，请保存好「加签密钥」(SEC)

参考文档: https://open.dingtalk.com/document/orgapp/custom-bot-send-message-type

### 配置文件

在 skill 所在目录创建 `config.json` 文件（可参考 `config.example.json`）：

```json
{
  "feishu": {
    "webhook_url": "https://open.feishu.cn/open-apis/bot/v2/hook/xxxxxxxxx",
    "secret": "你的签名密钥（可选，如未启用签名校验则留空字符串）"
  },
  "dingtalk": {
    "webhook_url": "https://oapi.dingtalk.com/robot/send?access_token=xxxxxxxxx",
    "secret": "你的加签密钥SEC（可选，如未启用加签则留空字符串）"
  },
  "poll_interval_seconds": 180,
  "language": "zh-CN"
}
```

- `poll_interval_seconds`: 轮询间隔，默认 180 秒（3 分钟）
- `language`: 播报语言，默认中文
- 至少配置一个平台（飞书或钉钉），两个都配置也可以

---

## 执行流程

### 第一步：询问用户想看的比赛

向用户询问以下信息：

1. **比赛类型**: 当前支持「世界杯」和「NBA」，请用户选择其一
2. **具体关注**: （可选）用户是否只关注特定球队/球员的比赛，默认播报所有正在进行的比赛

如果用户没有指定比赛类型，列出支持的赛事让用户选择。

### 第二步：搜索当前比赛状态

根据用户选择的比赛类型，使用搜索工具（web search / web fetch）查找当前正在进行的比赛。

**数据来源建议**（Agent 自行搜索获取）:
- **世界杯**: 搜索 "FIFA World Cup live scores today" 或 "世界杯 实时比分"
- **NBA**: 搜索 "NBA live scores today" 或 "NBA 实时比分"

**比赛状态判断**:

| 状态 | 处理方式 |
|------|----------|
| **正在直播 (LIVE)** | 进入第三步，开始轮询播报 |
| **已完结 (FT/Final)** | 直接汇总最终比分和关键数据，发送一次后结束 |
| **未开始 (Scheduled)** | 发送赛事前瞻（对阵双方、历史交锋、关键球员），告知用户开赛时间，不进入轮询 |

### 第三步：轮询播报（仅限正在进行的比赛）

对于正在进行的比赛，执行以下循环：

1. **首次播报**: 立即搜索并发送当前赛况
2. **定时轮询**: 每隔 `poll_interval_seconds` 秒（默认 180 秒 = 3 分钟）重新搜索一次
3. **持续播报**: 每次轮询后发送更新到配置的群机器人
4. **终止条件**: 当比赛状态变为「已完结」时，发送最终赛果总结，结束轮询

**轮询最大次数**: 为防止无限循环，单次 skill 运行最多轮询 60 次（约 3 小时）。超时后提示用户如需继续可重新运行 skill。

### 第四步：通过群机器人发送消息

根据 `config.json` 中配置的平台，发送消息到对应的群机器人。

---

## 消息格式规范

### 飞书消息格式

飞书自定义机器人支持「富文本」和「交互式卡片」两种格式。推荐使用「富文本」格式：

**发送方式**: HTTP POST 到 webhook URL

**签名计算**（如果配置了 secret）:
```
timestamp = 当前秒级时间戳
sign = Base64(HMAC-SHA256(timestamp + "\n" + secret, secret))
```

**消息体示例 — 比赛开始播报**:
```json
{
  "timestamp": "1700000000",
  "sign": "xxxxxxxxxxxxx",
  "msg_type": "interactive",
  "card": {
    "header": {
      "title": { "tag": "plain_text", "content": "🏀 NBA 实时播报开始" },
      "template": "blue"
    },
    "elements": [
      {
        "tag": "div",
        "text": {
          "tag": "lark_md",
          "content": "**洛杉矶湖人 vs 金州勇士**\n🕐 第一节 8:32\n🏟️ Crypto.com Arena\n\n📊 **当前比分**: 湖人 18 - 15 勇士\n\n📈 **节间数据**:\n- 湖人: 命中率 45%, 篮板 8, 助攻 4\n- 勇士: 命中率 40%, 篮板 6, 助攻 3"
        }
      },
      {
        "tag": "hr"
      },
      {
        "tag": "note",
        "elements": [
          {
            "tag": "plain_text",
            "content": "🤖 上班看球播报员 | 下次更新: 3分钟后"
          }
        ]
      }
    ]
  }
}
```

**消息体示例 — 比赛结束总结**:
```json
{
  "timestamp": "1700000000",
  "sign": "xxxxxxxxxxxxx",
  "msg_type": "interactive",
  "card": {
    "header": {
      "title": { "tag": "plain_text", "content": "🏀 NBA 比赛结束" },
      "template": "green"
    },
    "elements": [
      {
        "tag": "div",
        "text": {
          "tag": "lark_md",
          "content": "**洛杉矶湖人 112 - 108 金州勇士**\n\n📊 **全场数据**:\n- 湖人: 詹姆斯 32分8篮板7助攻, 戴维斯 25分12篮板\n- 勇士: 库里 28分5助攻, 汤普森 22分\n\n🏆 **比赛回顾**: 湖人末节发力逆转取胜，詹姆斯关键三分锁定胜局。"
        }
      },
      {
        "tag": "hr"
      },
      {
        "tag": "note",
        "elements": [
          {
            "tag": "plain_text",
            "content": "🤖 上班看球播报员 | 播报结束"
          }
        ]
      }
    ]
  }
}
```

**无签名时**: 省略 `timestamp` 和 `sign` 字段，直接发送 `msg_type` 和 `card`。

### 钉钉消息格式

钉钉自定义机器人支持 Markdown 格式消息。

**发送方式**: HTTP POST 到 webhook URL

**加签计算**（如果配置了 secret）:
```
timestamp = 当前毫秒级时间戳
stringToSign = timestamp + "\n" + secret
sign = URLEncode(Base64(HMAC-SHA256(stringToSign, secret)))
最终 URL = webhook_url + "&timestamp=" + timestamp + "&sign=" + sign
```

**消息体示例 — 比赛开始播报**:
```json
{
  "msgtype": "markdown",
  "markdown": {
    "title": "🏀 NBA 实时播报",
    "text": "## 🏀 NBA 实时播报\n\n**洛杉矶湖人 vs 金州勇士**\n\n- 🕐 第一节 8:32\n- 🏟️ Crypto.com Arena\n\n---\n\n### 📊 当前比分\n\n**湖人 18 - 15 勇士**\n\n---\n\n### 📈 节间数据\n\n**湖人**: 命中率 45%, 篮板 8, 助攻 4\n\n**勇士**: 命中率 40%, 篮板 6, 助攻 3\n\n---\n\n> 🤖 上班看球播报员 | 下次更新: 3分钟后"
  }
}
```

**消息体示例 — 比赛前瞻**:
```json
{
  "msgtype": "markdown",
  "markdown": {
    "title": "📋 比赛前瞻",
    "text": "## 📋 比赛前瞻\n\n**洛杉矶湖人 vs 金州勇士**\n\n- 🕐 开赛时间: 2026-06-13 10:30 (北京时间)\n- 🏟️ Crypto.com Arena\n\n---\n\n### 📈 历史交锋\n\n近5场: 湖人 3胜 - 2胜 勇士\n\n### ⭐ 关键球员\n\n**湖人**: 勒布朗·詹姆斯 (场均 25.3分)\n\n**勇士**: 斯蒂芬·库里 (场均 27.1分)\n\n---\n\n> 🤖 上班看球播报员 | 比赛尚未开始，敬请期待"
  }
}
```

---

## 轮询策略

```
┌─────────────────────────────────────────────────┐
│                   Skill 启动                      │
└─────────────────┬───────────────────────────────┘
                  ▼
         ┌───────────────┐
         │  询问比赛类型   │
         └───────┬───────┘
                  ▼
         ┌───────────────┐
         │  搜索比赛状态   │
         └───────┬───────┘
                  ▼
     ┌────────────┼────────────┐
     ▼            ▼            ▼
  已完结       正在直播      未开始
     │            │            │
     ▼            ▼            ▼
  发送最终      首次播报     发送前瞻
  赛果总结         │          结束
  结束            ▼
          ┌──────────────┐
          │ 等待 3 分钟   │◄──────┐
          └──────┬───────┘       │
                 ▼               │
          ┌──────────────┐       │
          │ 重新搜索赛况   │       │
          └──────┬───────┘       │
                 ▼               │
          ┌──────────────┐       │
          │ 发送更新播报   │       │
          └──────┬───────┘       │
                 ▼               │
          ┌──────────────┐       │
          │ 比赛结束?     │       │
          └──┬────────┬──┘       │
             │ 是     │ 否       │
             ▼        └──────────┘
      ┌──────────────┐
      │ 发送最终赛果   │
      │ 结束轮询      │
      └──────────────┘
```

---

## 重要规则

1. **必须先读 config.json**: 每次执行前读取配置文件，获取 webhook 地址和轮询间隔
2. **搜索优先**: 使用 Agent 自带的 web search / web fetch 工具获取实时比赛数据，不要依赖固定 API
3. **只播报进行中的比赛**: 已完结的比赛一次性总结，未开始的比赛只发前瞻
4. **消息去重**: 如果连续两次轮询的比分没有变化，在消息中注明「比分无变化」，但仍发送更新以确认播报仍在运行
5. **错误处理**: 如果搜索失败或发送消息失败，在下次轮询时重试，连续失败 3 次后通知用户并停止
6. **优雅终止**: 用户说「停止」「结束」「stop」时，发送最终赛果并结束轮询
7. **多场比赛**: 如果同一时间有多场进行中的比赛，在一条消息中汇总所有比赛，避免刷屏
8. **语言**: 所有播报内容使用中文（或 config.json 中配置的语言）

---

## 使用示例

```
用户: 帮我看下现在有什么NBA比赛
Agent: [读取 config.json] → [搜索 NBA live scores] → [发现 2 场进行中] → [发送播报到飞书/钉钉] → [开始轮询]

用户: 世界杯有比赛吗
Agent: [读取 config.json] → [搜索 World Cup live scores] → [发现 1 场未开始] → [发送前瞻] → [结束]

用户: 停止播报
Agent: [发送最终赛果] → [结束轮询]
```

---

## 平台兼容性说明

本 Skill 是一个纯 Markdown 指令文件，不依赖任何特定平台的 API 或 SDK。Agent 在执行时：

- 使用平台自带的 **web search** 工具获取比赛数据
- 使用平台自带的 **web fetch** 或 **bash/curl** 工具发送 webhook 消息
- 使用平台自带的 **文件读写** 工具读取 config.json

**已测试兼容的平台**:
- OpenCode (opencode)
- Claude Code (claude)
- Codex (openai/codex)
- Cursor
- OpenClaw
- Trae

如果你使用的平台不支持上述工具中的某一个，请参考对应平台的文档寻找替代方案。
