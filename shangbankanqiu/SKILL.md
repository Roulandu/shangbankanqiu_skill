---
name: shangbankanqiu
description: 上班看球 — 实时比赛播报，支持世界杯/NBA等赛事，通过飞书/钉钉群机器人推送赛况。本 Skill 不启动任何后台脚本/守护进程，所有工作完全在宿主 Agent 的对话循环内同步进行，因此 codex / cursor / opencode / claude code 都能一致地按相同协议执行。
argument-hint: "<比赛类型: 世界杯 | NBA>"
---

# 上班看球 (Shàngbān Kànqiú) — 实时比赛播报

## ⚠ CRITICAL RULES（任何 Agent 执行本 Skill 前必读）

> 这一节决定本 Skill 是否能跨 Agent 一致地工作。**违反任何一条都视为执行失败。**

| # | 规则 | 解释 |
|---|------|------|
| 1 | **禁止后台化** | 严禁启动后台进程/守护进程/计划任务/cron。所有轮询必须在宿主 Agent 的**当前对话循环**内同步进行（详见 §禁止清单）。 |
| 2 | **禁止脚本编写** | 严禁创建 `.py` / `.js` / `.sh` / `.ps1` 调度脚本来抓数据或循环发推。所有动作只能用 Agent 自带的 **web_search / shell / file_read** 工具完成。 |
| 3 | **默认靠 LLM web_search** | 默认不调用固定体育 API；只有足球场景中用户在 `providers.football` 显式启用且配置了 API key 的可选 provider，才允许按优先级尝试。未配置、关闭或失败时，必须回退到 Agent 自带的 `web_search` / `webfetch` 旧路径。 |
| 4 | **必须在前台 sleep** | 等待必须在前台执行 `Start-Sleep -Seconds 180` (Windows) 或 `sleep 180` (Unix)，**禁止 nohup / setTimeout / 异步回调**。Agent 的对话循环在 sleep 期间必须真实阻塞。 |
| 5 | **跨平台一致** | 4 家宿主 Agent (codex / cursor / opencode / claude code) 必须按 §标准协议 走出**步骤一致**的对话流（搜→渲染→发→sleep→搜）。 |
| 6 | **绝不静默失败** | webhook 调用必须打印 HTTP 响应；连续 N 轮失败必须停止轮询并向用户报告，不得继续 sleep。 |
| 7 | **BREAK-risk: events 不得进入 snapshot/dedup** | `events` 字段是按事件去重的、自然增长的 list。若把它塞进 `extract_score_snapshot` 的快照、或让 `scores_equal` 对它做 string-diff，会导致每轮快照都「看起来变化」→ dedup 失效 → 推送风暴刷爆群。事件去重的唯一正路是主循环里基于 `seen_event_ids` ring buffer 过滤，**禁止**塞进 snapshot（见 §extract_score_snapshot 排除列表与 §scores_equal 语义）。 |
| 8 | **BREAK-risk: 钉钉 markdown 20KB 上限** | 钉钉 markdown `text` 字段约 ~20KB 硬上限，超长会被服务端拒收并返回 `errcode != 0`。冗长的 commentary 直接拼接极易超限——`render_dingtalk_md` **必须**走 18000 字节 size guard（见 T10 / §钉钉 18000 字节 size guard），按 FIFO 丢队首事件直到落入阈值，并附「已截断 N 条」标记。play_by_play 模式事件更多（推荐 `max_events_per_msg=8-12`），size guard 更容易触发，必须确保截断逻辑在两种模式下都正确运行。 |
| 9 | **BREAK-risk: 真实配置键禁止 `_` 前缀** | `read_config` 通过 `strip_comment_keys` **递归剥离**所有以 `_` 开头的键（见 §read_config）。把 `include_events` / `max_events_per_msg` / `commentary_dedup_window` 等真实配置键写成 `_include_events` 之类会被**静默剥离**——配置看似已填、运行时全是默认值，极难排查。**只有** `_comment_*` 形态的注释兄弟键才能用 `_` 前缀，真实配置键必须**字面量**拼写。 |
| 10 | **BREAK-risk: play_by_play 模式每轮必推** | `commentary_style == "play_by_play"` 时主循环 push gate **永不跳过**——即使比分未变且无新事件。这是设计意图（用户选择文字直播就是想要持续更新）。若错误地在 play_by_play 模式下沿用 events_only 的 skip 逻辑，会导致文字直播模式下长时间无推送，用户体验退化为"装了文字直播但跟纯比分一样"。 |
| 11 | **BREAK-risk: 足球半场不是终场** | `HT` / `PAUSED` / `Half Time` / `Halftime` / `中场` 必须归一为 `HALFTIME`，继续轮询，不得触发 `render_final` 或 `cleanup_state`。只有显式 `FINISHED` / `FT` / `Full-Time` / `AET` / `AP`，或强多源终场确认，才允许结束足球播报。 |

### 🚫 禁止清单（明文黑名单）

下面这些操作 Agent **绝对不许做**，无论是世界杯还是 NBA，无论平台是 Windows 还是 macOS / Linux：

```text
# ❌ 禁止：把轮询丢到后台
nohup python poller.py &
python poller.py &
disown
screen -dmS poller ...
tmux new-session -d ...
start /B ...                          # Windows cmd
Start-Process -WindowStyle Hidden     # Windows PowerShell
schtasks /Create ...                  # Windows 计划任务
crontab -e                            # Unix cron
launchctl load ...                    # macOS launchd
systemd-run --user ...

# ❌ 禁止：写一个 daemon 脚本来代替 Agent 自身的循环
echo '...' > /tmp/poller.sh && bash /tmp/poller.sh &

# ❌ 禁止：用编程语言的异步定时器
setInterval(fn, 180000)
asyncio.create_task(...)
threading.Timer(180, ...).start()
subprocess.Popen(..., start_new_session=True)
os.fork()
child_process.spawn(cmd, { detached: true })

# ❌ 禁止：PowerShell 后台 Job / 计划任务 / 隐藏进程
Start-Job -ScriptBlock { ... }
Register-ScheduledJob ...
Register-ObjectEvent ...
wscript.shell.Run "...", 0, $false        # 隐藏窗口异步运行

# ❌ 禁止：Unix at / batch 一次性调度
echo "cmd" | at now + 1 minute
batch < script.sh

# ❌ 禁止：fork 出独立的 Agent 子进程做轮询
opencode run "..." &
claude -p "..." > /tmp/log &
codex exec "..." &

# ✅ 允许：宿主 Agent 在自己的对话循环里，用 shell 工具同步执行
sleep 180                  # Unix / macOS / WSL / Git Bash
Start-Sleep -Seconds 180   # Windows PowerShell
```

**判定标准**：如果 sleep 命令一返回，控制权立刻回到 Agent 自己（同一个对话），且 Agent 在执行下一次 web_search 之前**没有创建任何长期存活的子进程**——就是合规的。

---

## 概述

上班看球是一个跨平台的 AI Agent Skill，让用户在上班时也能通过飞书群机器人或钉钉群机器人实时接收比赛赛况播报。

**支持的平台**: opencode、Claude Code、Codex、Cursor、OpenClaw、Trae 等所有支持 SKILL.md 规范的 AI Coding Agent。

**当前支持的赛事**: 世界杯 (FIFA World Cup)、NBA

**核心定位**: 本 Skill **不是后台服务**，而是一段让宿主 Agent 在**当前对话**里反复执行的「播报循环指令」。Agent 干完一轮就 sleep 2 分钟，醒来再干一轮，直到比赛结束或用户喊停。在该对话内 Agent 是真的阻塞的，但用户开新的对话/新窗口/新 Agent 实例依然可以正常使用 codex / cursor / opencode / claude code 做别的事——这就是「应用可以一致运行」的含义。

---

## 各宿主 Agent 工具映射表

为保证 4 家 Agent 走出一致结果，本表列出**每家 Agent 必须使用的工具名**。Agent 在执行本 Skill 时**只能**用本表中的工具——不要自己引入额外工具。

| Agent | 网络搜索 | Shell 执行 | 文件读取 | 文件写入 |
|-------|---------|-----------|---------|---------|
| **opencode** | `websearch_web_search_exa`（首选）/ `webfetch`（兜底直拉网页）/ `task(subagent_type="librarian")`（复杂查证） — 具体工具名以宿主提供的为准，**不要**自己 import 任何 SDK | `bash`（Windows = PowerShell 5.1） | `read` | `write`（覆盖式）/ `edit`（增量改） |
| **Claude Code (claude)** | `WebSearch` + `WebFetch` | `Bash` | `Read` | `Write` / `Edit` |
| **Codex (openai/codex)** | `web_search`（内置） | `shell` | `shell cat ...` | **`shell` + heredoc/echo**（state_file 是简单 JSON，用 `cat > file <<'EOF' ... EOF` 或 `printf '%s' '...' > file`；**不要**用 `apply_patch`，那是 diff 工具） |
| **Cursor (cursor-agent)** | 内置 web search | 内置 terminal | 内置 read_file | 内置 edit_file |

> **如果你的 Agent 没有 web_search 工具**：使用 `webfetch` 直接 GET 一个赛况聚合页（例如 `https://www.google.com/search?q=NBA+live+scores+today` 或 `https://www.flashscore.com/`），再让 LLM 解析返回的 HTML/Markdown。除足球场景中已显式启用且配置 key 的可选 provider 外，**绝不**降级为调用固定 API。

> **事件处理与宿主无关 (host-agnostic)**：Event handling guidance is host-agnostic — codex、cursor、opencode、claude code MUST all follow the same Stage A/B fetch + `id_hash` dedup + N-element render rules above. 即四家 Agent 的事件抓取入口（Stage A URL 嗅探）、事件抽取流水线（Stage B + 同一份 sha1 `id_hash` 算法）、推送门控与渲染（N-element 卡片、18KB size guard）必须**完全一致**——平台差异**只**体现在「web_search / webfetch / shell 这一个工具调用换成本表里对应的工具名」。详见 §Stage B 与 §主循环 push gate。

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
  "poll_interval_seconds": 120,
  "max_polls": 60,
  "max_consecutive_failures": 3,
  "state_file": ".shangbankanqiu_state.json",
  "language": "zh-CN",
  "include_events": true,
  "max_events_per_msg": 6,
  "commentary_dedup_window": 30,
  "commentary_style": "events_only",
  "play_by_play_default_poll_interval_seconds": 90,
  "search_freshness_boost": true,
  "_comment_providers": "可选足球 API 数据源。所有 provider 默认关闭；未配置 API key 时自动使用 web_search/webfetch 旧路径。",
  "providers": {
    "football": {
      "priority": [
        "sportradar_extended",
        "sportmonks",
        "api_football",
        "football_data_org",
        "web"
      ],
      "sportradar_extended": {
        "enabled": false,
        "api_key": ""
      },
      "sportmonks": {
        "enabled": false,
        "api_key": ""
      },
      "api_football": {
        "enabled": false,
        "api_key": ""
      },
      "football_data_org": {
        "enabled": false,
        "api_key": "",
        "status_only": true
      }
    }
  }
}
```

| 字段 | 类型 | 默认 | 范围 | 含义 |
|------|------|------|------|------|
| `poll_interval_seconds` | integer | 120 | ≥ 60 | 两轮之间 sleep 的秒数，建议不低于 60 |
| `max_polls` | integer | 60 | ≥ 1 | 单次 Skill 运行最多轮询多少次（防止无限循环） |
| `max_consecutive_failures` | integer | 3 | ≥ 1 | 连续失败几次就终止 |
| `state_file` | string | `.shangbankanqiu_state.json` | — | 跨轮去重用的临时状态文件（存上次比分），相对 skill 目录 |
| `language` | string | `zh-CN` | — | 播报语言 |
| `include_events` | boolean | `true` | `true` / `false` | 是否在比分推送的同时抓取并广播比赛内事件（goal / foul / sub / key_play 等）。`false` 时整段 Stage B 跳过、所有 match `events=[]`，本 Skill 退化为纯比分播报，详见 §Stage B Graceful Degradation。 |
| `max_events_per_msg` | integer | 6 | 1–20 | 每次推送最多附带的事件条数。钉钉 markdown `text` 字段约 **20KB** 硬上限——`events_only` 模式推荐 4–8（再多容易被钉钉服务端截断；详见 §钉钉 18000 字节 size guard）；`play_by_play` 模式推荐 8–12（文字直播事件更多，但仍需注意钉钉 size guard）。 |
| `commentary_dedup_window` | integer | 30 | 1–200 | `seen_event_ids` ring buffer 记多少**轮**的事件 id_hash；buffer 容量 `cap = commentary_dedup_window × max_events_per_msg`。默认 30 × 6 = 180 条 id_hash（state 文件约 9KB）。详见 §state file schema。 |
| `commentary_style` | string | `"events_only"` | `"events_only"` / `"play_by_play"` | 播报风格。`events_only` = 仅在比分变化或出现进球/犯规等重大事件时推送（当前默认行为）。`play_by_play` = 文字直播模式——像文字解说员一样每轮推送比赛进程，即使比分未变也有叙述更新；LLM 提取更细粒度的事件类型（射门/扑救/角球等），且每轮至少产一条 `commentary` 类型事件描述当前局势。详见 §Stage B LLM prompt 变体 与 §render_live 3×2 状态表。 |
| `play_by_play_default_poll_interval_seconds` | integer | 90 | ≥ 60 | 文字直播模式推荐的轮询间隔（秒）。当 `commentary_style == "play_by_play"` 且用户未自定义 `poll_interval_seconds`（仍为默认值 180）时，自动使用此值代替，使文字直播更「实时」。用户显式设置了非 180 的 `poll_interval_seconds` 时，以用户设置为准。 |
| `search_freshness_boost` | boolean | `true` | `true` / `false` | 是否启用 Stage A+ Fallback（实时源精准搜索）。`true`（默认）：当 Stage A 主搜索没找到文字直播 URL 时，自动用 `site:` 限定搜索精准定位中文实时源（虎扑/新浪），大幅减少「进球后长时间检测不到」的问题。`false`：关闭此 fallback，回到旧行为（Stage A 嗅探不到 URL 时直接 `events=[]`）。详见 §build_pbp_search_query 与 FAQ Q1。 |
| `providers.football.priority` | array&lt;string&gt; | 见示例 | provider 名称列表 | 足球数据源优先级。建议按 `sportradar_extended` → `sportmonks` / `api_football` → `football_data_org` → `web` 排序；所有 API provider 都不可用时必须回退到 `web_search` / `webfetch` 旧路径。 |
| `providers.football.*.enabled` | boolean | `false` | `true` / `false` | 是否启用对应足球 API provider。默认全部关闭；只有显式设为 `true` 且配置了可用 `api_key` 时才允许尝试该 provider。 |
| `providers.football.*.api_key` | string | `""` | — | 对应 provider 的 API key。留空时视为未配置，Agent 必须跳过该 provider 并继续尝试下一优先级或最终回退到 `web`。 |
| `providers.football.football_data_org.status_only` | boolean | `true` | `true` / `false` | 是否仅使用 football-data.org 做比赛状态校准。建议保持 `true`，用于区分 `PAUSED` / `FINISHED` 等状态，避免把中场休息误判为完场。 |

> **⚠ 配置键命名警告（重申已有规则）**：上表所有「真实键」**绝不**以下划线 `_` 开头；只有 `_comment_*` 形态的注释兄弟键才会被 `strip_comment_keys` 剥离（见 §read_config）。把真实键写成 `_include_events` / `_max_events_per_msg` 之类**会被静默剥离**——配置看起来填了但运行时全是默认值，极难排查。`include_events` / `max_events_per_msg` / `commentary_dedup_window` / `commentary_style` / `play_by_play_default_poll_interval_seconds` / `search_freshness_boost` 所有真实配置键**必须**按本表的字面量拼写出现在 `config.json` 顶层。

至少配置飞书或钉钉中的一个，两个都配置也可以（会同时推送）。

---

## 标准协议（4 家 Agent 必须严格按此执行）

> **目标**: 无论你是 codex、cursor、opencode 还是 claude code，按这套伪代码执行后给用户的对话流应当是一致的（同样的步骤序、同样的消息格式、同样的退出条件）。

### 主流程伪代码

```text
# ========== 启动阶段 (只跑 1 次) ==========
config         = read_config()                          # 见 §read_config
match_type     = ask_user_if_missing(in {世界杯, NBA})    # 缺则问，不要默认
follow         = ask_user_optional(关注球队/球员)          # 可选，可为空字符串
state_path     = resolve_state_path(config.state_file)  # 见 §resolve_state_path
notify_stop_hint_once()                                 # 告知用户怎么终止，见 §优雅终止

# ========== 首次诊断 (只跑 1 次) ==========
if is_football_match_type(match_type):
    matches = fetch_football_snapshot(match_type, follow, config)  # API provider chain → web fallback
else:
    raw     = web_search(build_query(match_type, follow))
    matches = parse_search_result(raw)
state   = classify_state(matches)  # see classify_state / football_status_machine

if state == "未开始":
    sign_and_post(render_preview(matches), config)      # 见 §sign_and_post
    EXIT("已发送赛事前瞻，本次 skill 运行结束")

if state == "已完结":
    sign_and_post(render_final(matches), config)
    EXIT("已发送最终赛果总结，本次 skill 运行结束")

# state == "正在直播" → 进入轮询
prev              = read_state_file(state_path)         # 见 §read_state_file (返回 None / dict)
persisted         = prev if isinstance(prev, dict) else {}   # 永远是 dict，方便 mutate；空 dict 表示「首轮无历史」
last_pushed_score = persisted.get("score")              # None 表示首轮
fail_count        = 0
poll_idx          = 0
# play_by_play mode recommends shorter intervals for a more "live" feel.
# If user left poll_interval_seconds at the default 120, auto-reduce to
# play_by_play_default_poll_interval_seconds (default 90). If user explicitly
# set a non-default value, respect it.
# LIMITATION: 代码无法区分「用户主动设了 120」和「默认值 120」——两者在 config 中看起来一样。
# 如果用户确实想在 play_by_play 模式下用 120 秒间隔，需将 poll_interval_seconds 设为非 120 的值
# （如 121 或 119），否则会被自动降为 play_by_play_default_poll_interval_seconds。
base_interval = config.poll_interval_seconds
if config.get("commentary_style", "events_only") == "play_by_play" and base_interval >= 120:
    # Only auto-reduce if user hasn't customized (120 = default from config.example.json)
    base_interval = min(base_interval, config.get("play_by_play_default_poll_interval_seconds", 90))
poll_interval     = max(base_interval, 60)   # 强制最小 60 秒

# ========== 轮询循环（在【当前对话】内同步执行；禁止后台化） ==========
while poll_idx < config.max_polls:
    poll_idx += 1

    # ---- 第 1 步: SEARCH ----
    try:
        if is_football_match_type(match_type):
            fresh = fetch_football_snapshot(match_type, follow, config)  # API provider chain → web fallback
        else:
            raw   = web_search(build_query(match_type, follow))
            fresh = parse_search_result(raw)
    except Exception:
        fail_count += 1
        if fail_count >= config.max_consecutive_failures:
            cleanup_state(state_path)
            notify_user("连续 N 次搜索失败，停止轮询"); EXIT
        sleep_in_foreground(poll_interval)
        continue

    # ---- 第 1.5 步: STAGE B（事件抓取）—— 见 §Stage B Event Extraction ----
    # P2-1 (Oracle Round 10)：Stage B 必须由主循环显式调用；不能把它当成「另一处文档」就跳过。
    # 任何 Stage B 子步异常一律 graceful=[]——**绝不**让 Stage B 失败影响第 2 步的 push gate。
    if config.include_events:
        for m in fresh:
            provider_events = m.get("events", []) or []  # provider 预置事件；Stage B 只能补充/合并，不得覆盖
            # 1.5a. 仅对正在直播的场次抓事件；未开始/已完结跳过。
            # P2-2：match dict 没有 .state 字段；用 phase_keywords（§parse_search_result）判断。
            phase_kws = m.get("phase_keywords", []) or []
            live_kws  = {"Live", "直播中", "Q1", "Q2", "Q3", "Q4", "OT", "1H", "2H",
                         "上半场", "下半场", "加时", "进行中", "中场"}
            if not any(kw in phase_kws for kw in live_kws):
                m["events"] = provider_events
                continue
            # 1.5b. 从 Stage A snippets 选 play-by-play URL（按 §Stage A 5 行优先级表）
            #       优先级：URL 缓存 > Stage A 嗅探 > Stage A+ 补搜
            try:
                match_key_for_url = m.get("home", "") + "|" + m.get("away", "")

                # ── 1. 优先使用跨轮 URL 缓存（见 §state file schema pbp_urls） ──
                # 第二轮起直接用已发现的 URL，跳过整个 URL 发现阶段，大幅减少延迟。
                # 但缓存有 staleness 保护：如果连续 N 轮从同一 URL 抓到 0 条新事件，
                # 说明该 URL 可能已过期或更新太慢，自动清除缓存重新发现。
                pbp_url = persisted.get("pbp_urls", {}).get(match_key_for_url)
                if pbp_url is not None:
                    empty_streak = persisted.get("pbp_url_empty_streaks", {}).get(match_key_for_url, 0)
                    if empty_streak >= 3:                        # 连续 3 轮无新事件 → 缓存可能过期
                        del persisted["pbp_urls"][match_key_for_url]
                        if "pbp_url_empty_streaks" in persisted:
                            persisted["pbp_url_empty_streaks"].pop(match_key_for_url, None)
                        pbp_url = None                          # 强制重新发现

                # ── 2. 缓存 miss → 从 Stage A snippets 嗅探 ──
                if pbp_url is None:
                    pbp_url = pick_playbyplay_url_from_stage_a(m)

                # ── 3. Stage A+ Fallback ──
                # 当 Stage A 主搜索没返回 playbyplay URL 时，用 site: 限定搜索
                # 精准定位中文实时源。这是解决"进球后长时间检测不到"的核心机制：
                # Stage A 的通用搜索返回的往往是 ESPN/Google 聚合页（延迟 3-10min），
                # 而虎扑/新浪文字直播页更新延迟仅 ~30s。但通用搜索很少返回这些页面。
                # Stage A+ 用 site: 限定直接搜索中文源的 playbyplay 页面。
                if pbp_url is None and config.get("search_freshness_boost", True):
                    pbp_q   = build_pbp_search_query(match_type, m.get("home", ""), m.get("away", ""))
                    try:
                        pbp_raw = web_search(pbp_q)
                        # 从 Stage A+ 搜索结果中提取 URL，复用同一套 pick_playbyplay_url 逻辑
                        pbp_match = {"source_url": "", "raw": pbp_raw}  # 临时 match dict，仅用于 URL 嗅探
                        # 尝试从搜索结果中找 source_url
                        pbp_urls_from_raw = extract_urls_from_text(pbp_raw or "")
                        if pbp_urls_from_raw:
                            pbp_match["source_url"] = pbp_urls_from_raw[0]      # 取第一个 URL 作为 source_url
                        pbp_match["raw"] = pbp_raw or ""
                        pbp_url = pick_playbyplay_url_from_stage_a(pbp_match)
                    except Exception:
                        pbp_url = None                                  # Stage A+ 也失败 → 优雅回退

                # ── 4. 写入 URL 缓存（发现即缓存，后续轮次直接复用） ──
                if pbp_url is not None:
                    if "pbp_urls" not in persisted:
                        persisted["pbp_urls"] = {}
                    persisted["pbp_urls"][match_key_for_url] = pbp_url
                    # 重置空轮计数器（新 URL 或 URL 仍有效）
                    if "pbp_url_empty_streaks" not in persisted:
                        persisted["pbp_url_empty_streaks"] = {}
                    persisted["pbp_url_empty_streaks"][match_key_for_url] = 0

                if pbp_url is None:
                    m["events"] = provider_events
                    continue
                page_md    = webfetch(pbp_url, format="markdown", timeout_seconds=15)
                # P0-2 (Oracle Round 12): 函数签名是 (page_md, match_key, schema)；不接受 cap。
                # cap 截断由调用方在下一行做（raw_events[-cap_b:]）。
                raw_events = llm_extract_events(
                    page_md,
                    match_key = m.get("home", "") + "|" + m.get("away", ""),
                    schema    = "see §parse_search_result event schema (id_hash + ts + team + type + text)",
                )
                cap_b      = (3 if config.get("commentary_style", "events_only") == "play_by_play" else 2) * config["max_events_per_msg"]    # play_by_play: 3× 防丢(默认24); events_only: 2×(默认12)
                # merge provider events + Stage B events by id_hash；Task 5 会扩展多源合并细则。
                merged_events = []
                seen_event_hashes = set()
                for e in provider_events + raw_events[-cap_b:]:
                    eid = e.get("id_hash")
                    if eid and eid not in seen_event_hashes:
                        merged_events.append(e)
                        seen_event_hashes.add(eid)
                m["events"] = merged_events
                # 更新 URL 缓存空轮计数器：有事件 → 重置；无事件 → 递增
                if "pbp_url_empty_streaks" not in persisted:
                    persisted["pbp_url_empty_streaks"] = {}
                if len(m["events"]) > 0:
                    persisted["pbp_url_empty_streaks"][match_key_for_url] = 0
                else:
                    persisted["pbp_url_empty_streaks"][match_key_for_url] = \
                        persisted["pbp_url_empty_streaks"].get(match_key_for_url, 0) + 1
            except Exception:
                m["events"] = provider_events             # graceful=provider_events；绝不 raise
    else:
        for m in fresh:
            m["events"] = []                              # include_events=false → 退化成纯比分模式

    # ---- 第 1.7 步: SCORE-CHANGE EVENT SYNTHESIS（比分变化事件合成） ----
    # 当比分发生变化且 events 中没有现成 goal 事件时，补充合成进球事件。
    # 这是最后一道防线——即使搜索延迟、URL 缺失、webfetch 超时，只要比分变了，
    # 用户一定能看到一条「某队进球」的推送，而不是干巴巴的「比分 0-0 → 1-0」。
    #
    # CRITICAL RULE #7 补充：events 不进 snapshot/dedup 是对的。但合成事件是独立的——
    # 它们有专门的 id_hash 前缀 "syn:" ，不会与 Stage B 的真实事件冲突。
    # 合成事件只补缺 goal；必须保留 provider / Stage B 已有的非 goal 事件。
    # merge_events_by_id 保留同一 id_hash 的首次出现，丢弃后续重复 id。
    if last_pushed_score is not None and config.include_events:
        # 将 last_pushed_score（list of {home, away, score, phase}）转为 match_key → score dict
        # 便于按 match_key 查找旧比分。last_pushed_score 来自 extract_score_snapshot，
        # 其结构是 [{"home": "Lakers", "away": "Celtics", "score": "98-92", "phase": "Q3"}, ...]
        old_score_map = {}
        for s in last_pushed_score:
            key = s.get("home", "") + "|" + s.get("away", "")
            old_score_map[key] = s.get("score", "")          # e.g. "0-0" or "98-92"

        for m in fresh:
            # 仅对正在直播的场次做合成
            phase_kws = m.get("phase_keywords", []) or []
            live_kws  = {"Live", "直播中", "Q1", "Q2", "Q3", "Q4", "OT", "1H", "2H",
                         "上半场", "下半场", "加时", "进行中", "中场"}
            if not any(kw in phase_kws for kw in live_kws):
                continue
            # 仅当 Stage B 未抓到进球事件时才合成
            # 注意：不能用 len(m["events"]) > 0 判断——Stage B 可能返回了旧事件（缓存页面），
            # 但其中不包含本轮新进球。必须检查 events 中是否已有 type="goal" 的事件。
            existing_goals = [e for e in m.get("events", []) if e.get("type") == "goal"]
            if existing_goals:
                continue
            # 检查该场比赛比分是否变化
            match_key = m.get("home", "") + "|" + m.get("away", "")
            old_score_raw = old_score_map.get(match_key)       # e.g. "0-0"
            new_score_raw = m.get("score", "")                 # e.g. "1-0" or "98-92"
            if not old_score_raw or not new_score_raw:
                continue
            # 解析新旧比分（处理 "98-92" / "98-92 OT" / "2-1 (PK)" 等格式）
            try:
                # 用正则提取数字部分，避免 split("-") 被 "OT" / "(PK)" 等后缀干扰
                old_nums = re.findall(r'\d+', old_score_raw)
                new_nums = re.findall(r'\d+', new_score_raw)
                if len(old_nums) < 2 or len(new_nums) < 2:
                    continue
                old_home, old_away = int(old_nums[0]), int(old_nums[1])
                new_home, new_away = int(new_nums[0]), int(new_nums[1])
            except (ValueError, IndexError):
                continue
            home_delta = new_home - old_home
            away_delta = new_away - old_away
            # 合成进球事件
            syn_events = []
            if home_delta > 0:
                for i in range(home_delta):
                    syn_events.append({
                        "id_hash": "syn:" + sha1((match_key + "|" + new_score_raw + "|home_goal|" + str(i)).encode("utf-8"))[:10],
                        "ts":      "",
                        "team":    "home",
                        "type":    "goal",
                        "text":    m.get("home", "") + " 进球！比分 " + new_score_raw,
                    })
            if away_delta > 0:
                for i in range(away_delta):
                    syn_events.append({
                        "id_hash": "syn:" + sha1((match_key + "|" + new_score_raw + "|away_goal|" + str(i)).encode("utf-8"))[:10],
                        "ts":      "",
                        "team":    "away",
                        "type":    "goal",
                        "text":    m.get("away", "") + " 进球！比分 " + new_score_raw,
                    })
            if syn_events:
                m["events"] = merge_events_by_id(m.get("events", []) + syn_events)

    # ---- 第 2 步: RENDER + PUSH GATE ----
    state          = classify_state(fresh)                # 每轮都要重判，状态会变化（直播→完结）
    score_snapshot = extract_score_snapshot(fresh)        # 见 §extract_score_snapshot
    is_finished    = (state == "已完结")

    # 2a. 计算本轮所有候选「新事件」（post-dedup against persisted["seen_event_ids"]）
    #     —— 见 §state file schema (seen_event_ids) 与 §parse_search_result event schema
    #     注意：persisted 是 state file dict（首轮 = {}），切勿与上一行的 state（match 分类字符串）混淆
    seen_ids               = persisted.get("seen_event_ids", [])
    all_events_this_cycle  = []                           # list of (match, event)
    for m in fresh:
        # NOTE（P2-7, Oracle Round 10）：m 是 §parse_search_result 输出的 dict，必须用 m.get("events", [])。
        # 不能写 hasattr(m, "events")——Python 的 hasattr 检查的是 attribute 不是 dict key，对 dict
        # 永远返 False，会导致 events 永远读不到 → 整个 events 特性静默失效。
        for e in m.get("events", []):
            if e["id_hash"] not in seen_ids:
                all_events_this_cycle.append((m, e))

    score_changed  = (last_pushed_score is None) or (not scores_equal(score_snapshot, last_pushed_score))
    has_new_events = len(all_events_this_cycle) > 0

    # 2b. push gate —— commentary_style determines skip behavior:
    #
    #   commentary_style == "events_only" (original behavior):
    #     score 未变 + 无新事件 → skip 整轮
    #     其它三种组合 → 推送
    #
    #   commentary_style == "play_by_play" (文字直播):
    #     NEVER skip — always push. Even when score unchanged and no new events,
    #     the current score/phase itself is the update (the user chose play_by_play
    #     precisely because they want continuous updates).
    #     Exception: if Stage B completely failed AND score unchanged → still push
    #     (score+phase is still useful info in play_by_play mode).
    should_skip = (not score_changed and not has_new_events
                   and config.get("commentary_style", "events_only") == "events_only")

    if should_skip:
        log("no score change and no new events (events_only mode) — skip push")
        # 不进入 PERSIST，不调用 sign_and_post，直接跳到 SLEEP
        # （is_finished 检查仍在第 5 步执行，比赛刚好在「无变化无事件」轮结束的极小概率场景下，
        #  下一轮 fresh 重新分类即可触达 is_finished 分支。）
        if is_finished:
            sign_and_post(render_final(fresh), config); cleanup_state(state_path); EXIT("比赛已结束，已发送最终赛果")
        if user_said_stop():
            sign_and_post(render_final(fresh, note="用户主动停止播报"), config); cleanup_state(state_path); EXIT
        sleep_in_foreground(poll_interval); continue

    # 2c. 给每个 match 挂上 new_events（post-dedup + trim 到 max_events_per_msg，保留最新 N 条）
    #      —— render_live 会从 m.new_events 平铺到 msg.events（见 §render_live 伪代码）
    cap_msg = config["max_events_per_msg"]   # 等价 config.max_events_per_msg；本 SKILL 中 config.X 与 config["X"] 互通（dict 访问）
    for m in fresh:
        # NOTE（P2-7）：m.get("events", [])，而非 m.events / hasattr——见上方 2a 同名注释
        m["new_events"] = [e for e in m.get("events", [])
                           if e["id_hash"] not in seen_ids]
        m["new_events"] = m["new_events"][-cap_msg:]      # 只保留最新 cap_msg 条

    # 2d. 渲染 —— render_live 内部会按 2x2 状态表选 body 前缀（见 §render_live）
    msg = render_live(fresh, last_pushed_score, score_snapshot, config)

    # 2e. 防御式 SKIP 检查 —— 理论上 2b 已经过滤过；这里防止 render_live 内部判定哨兵漏网
    if msg == "SKIP":
        log("render_live returned SKIP sentinel — defensive skip, no push")
        if is_finished:
            sign_and_post(render_final(fresh), config); cleanup_state(state_path); EXIT
        if user_said_stop():
            sign_and_post(render_final(fresh, note="用户主动停止播报"), config); cleanup_state(state_path); EXIT
        sleep_in_foreground(poll_interval); continue

    # ---- 第 3 步: POST + SIGN ----
    push_result = sign_and_post(msg, config)              # {feishu: bool|null, dingtalk: bool|null}
    push_ok     = all_configured_succeeded(push_result, config)
    if not push_ok:
        fail_count += 1
        if fail_count >= config.max_consecutive_failures:
            cleanup_state(state_path)
            notify_user("Webhook 推送连续 N 次失败"); EXIT
        # 推送失败时跳过 PERSIST，避免 last_pushed_score / seen_event_ids 被未送达的内容污染
        # CRITICAL: ring buffer mutation happens ONLY ON push_ok=true.
        # 仅在推送成功（push_ok=true）时才更新 seen_event_ids —— 失败的 Feishu/DingTalk
        # 发送绝对不能 poison dedup，否则下一轮这批事件就被错误标记成「已推送」永远丢失。
        # 推送失败 → 这批事件保留在 m.events 里，下一轮重新走 dedup → 重新尝试。
    else:
        fail_count = 0
        # ---- 第 4 步: PERSIST (仅在推送成功时执行) ----
        # 4a. 比分快照
        last_pushed_score = score_snapshot
        # 4b. seen_event_ids 环形缓冲区追加 + 裁剪 —— 见 §state file schema
        #     persisted 是 state file dict（首轮 = {}），mutate 后整体写盘
        # CRITICAL（P1, Oracle Round 10）：new_ids 必须从 m.new_events（trimmed, 实际推送的）取，
        # 而不是 all_events_this_cycle（pre-trim, 含被 cap_msg 裁掉的旧事件）。否则被裁掉的旧事件
        # 会被错误标 seen，下一轮 dedup 把它过滤掉 → **永久丢失**。
        # 例：Stage B 返 12 条新事件，cap_msg=6，只推后 6 条；前 6 条若进 seen_event_ids 就再也不会推。
        new_ids = []
        for m in fresh:
            for e in m.get("new_events", []):
                new_ids.append(e["id_hash"])
        persisted["seen_event_ids"] = persisted.get("seen_event_ids", []) + new_ids
        cap_ring = config.commentary_dedup_window * config.max_events_per_msg
        if len(persisted["seen_event_ids"]) > cap_ring:
            persisted["seen_event_ids"] = persisted["seen_event_ids"][-cap_ring:]
        # 4c. 整体写盘（score + ts + seen_event_ids + pbp_urls 一次性原子持久化）
        persisted["score"] = score_snapshot
        persisted["ts"]    = now()
        # pbp_urls 已在 §第 1.5 步中发现时即时写入 persisted dict，
        # 这里随整体写盘一起持久化到磁盘。
        write_state_file_safely(state_path, persisted)

    # ---- 第 5 步: TERMINATE? ----
    if is_finished:
        sign_and_post(render_final(fresh), config)
        cleanup_state(state_path)
        EXIT("比赛已结束，已发送最终赛果")

    if user_said_stop():                                  # 见 §优雅终止
        sign_and_post(render_final(fresh, note="用户主动停止播报"), config)
        cleanup_state(state_path)
        EXIT("用户请求停止，已发送最终赛果")

    # ---- 第 6 步: SLEEP (前台同步阻塞，禁止后台化) ----
    sleep_in_foreground(poll_interval)

# 跑满 max_polls 仍未结束
cleanup_state(state_path)
notify_user("已轮询 {max_polls} 轮（约 {max_polls*poll_interval/60} 分钟），如需继续请重新运行")
EXIT
```

### 配套子函数语义（4 家 Agent 必须严格按此实现）

#### 伪代码约定（pseudocode placeholders）

下面这些函数名遍布本节伪代码，但**不是**需要 Agent 单独定义/调用的具名工具——它们是「这一步动作」的**伪代码占位符**，4 家 Agent 各自用最贴近的本地能力实现就行：

| 占位符 | 语义 | 4 家 Agent 各自怎么落地 |
|--------|------|------------------------|
| `log("...")` / `log_warning("...")` / `log_error("...")` | 把一行字符串写到 Agent 自己的对话流（最简单就是 `print` 或 Agent 的标准输出） | codex/cursor/opencode/claude code 都直接以一行文字回显给用户即可——不要 fopen 写日志文件、不要起 logging 模块 |
| `now()` | 当前时间（用于日志时间戳、state file 中的 `updated_at` 等） | Python `datetime.utcnow().isoformat()` / shell `date -u +%FT%TZ` / Agent 自己的 datetime 工具——只要返回一个 ISO 8601 字符串即可 |
| `notify_user("...")` | 把一段提示展示给当前对话里的人类用户 | 直接在对话里输出文本即可——和 `log` 几乎等价，区别在于 `notify_user` 偏向「重要的一次性提醒」（如 stop hint） |
| `ask_user_if_missing(field, prompt)` / `ask_user_optional(...)` | 当 config 字段为空/未填 → 在对话里向用户追问 | 各 Agent 用自己的 `ask` / `question` / 直接用文本提问皆可——本 skill **不依赖** 任何特定的 question 工具 |
| `host_agent_self_reason(prompt)` | 「让宿主 Agent 自己 LLM backbone 处理这段 prompt」（详见 §`llm_extract_events`） | 不是函数调用——Agent 把 prompt 当作自己 reasoning step 的输入，再读出输出 |
| 标准库函数：`sha1`、`json.loads`、`JSONDecodeError`、`hmac`、`base64`、`time` | 标准库等价物 | Python 直接用 `hashlib.sha1` 等；其他语言用各自标准库的等价 API |
| `config.X` ↔ `config["X"]`（以及 `match.home` ↔ `match["home"]` 等其他**纯读** dict 访问） | 伪代码两种风格**等价**——`config` / `m`（来自 `parse_search_result`）都是 dict | 各 Agent 用自己语言惯用的 dict 访问即可（Python 用 `["X"]` / `.get("X")`；JSON 风格 mock 用 `.X`）。**例外**：写赋值（`match["events"] = [...]` / `m["new_events"] = [...]`）必须用 bracket，不要写 `match.events = [...]`——见 P0-3 / P1-1 教训。 |

**关键非约束**：本节列出的占位符都**不**承担「跨 Agent 一致性」契约——它们只影响日志可读性 / UX 提示文案，输出不进 `id_hash` 计算，不写 state file，不通过 webhook 推送。所以 4 家 Agent 在 `log()` 文案上有微小差异是 OK 的；**真正需要 byte-equal 的是 `id_hash` 算法、prompt 字面、state file schema、webhook body schema**——这些都已经在各自函数里**字面**规定了。

#### `read_config()`

读取 skill 同目录的 `config.json`。如果文件不存在或解析失败 → 立刻终止并提示用户先 `cp config.example.json config.json`。**不要用任何「合理默认值」往下走**。

**注释键过滤（必须）**：`config.example.json` 里使用 `_comment_*` 形式的键作为「JSON 友好的注释」（标准 JSON 不支持 `//`）。读取后必须**递归剥离所有以 `_` 开头的键**：

```text
def strip_comment_keys(obj):
    if isinstance(obj, dict):
        return {k: strip_comment_keys(v) for k, v in obj.items() if not k.startswith("_")}
    if isinstance(obj, list):
        return [strip_comment_keys(x) for x in obj]
    return obj

config = strip_comment_keys(json.parse(read("config.json")))
```

这样用户从 `config.example.json` 直接复制保留 `_comment_*` 也能正常运行，且不会把注释当成未知字段误用。

#### `build_query(match_type, follow)` —— Stage A: 比分搜索 + 文字直播 URL 嗅探

```text
def build_query(match_type, follow):
    """返回 Stage A 主搜索查询（单条）。
    策略：中英混合查询，确保搜索引擎同时返回中文（虎扑/新浪）和英文（ESPN）结果。
    中文关键词在前——大多数搜索引擎对查询前段的词赋予更高权重，
    这有助于将中文实时源（更新延迟 ~30s）排在英文聚合页（延迟 3–10min）之前。"""
    base = {
      "世界杯": "世界杯 比分 直播 FIFA World Cup live scores today",
      "NBA":   "NBA 比分 直播 NBA live scores today",
    }[match_type]

    if follow is non-empty:
        return base + " " + follow            # 例: "NBA 比分 直播 NBA live scores today 湖人"
    else:
        return base                           # 不要拼接空的 "+"，避免查询畸形
```

> **为什么中英混合**：纯英文 `"NBA live scores today"` 几乎只返回 ESPN/Google 体育聚合页——这些页面对中文比赛源有 3–10 分钟的天然延迟，且搜索结果中极少包含 `nba.hupu.com` 或 `match.sports.sina.com.cn` 的文字直播 URL，导致 Stage B 的 URL 嗅探几乎必然 miss（见 §Stage A+ Fallback 的设计动机）。在查询前段加入 `比分 直播` 等中文关键词后，搜索引擎更可能返回虎扑/新浪等中文实时页面，大幅提高 Stage A → Stage B 的 URL 传递成功率。

**Stage A 双重职责（当 `config.include_events=true` 时）**：本函数的签名和 base 字符串保持不变；但当 `include_events=true` 且本轮走 `web_search(build_query(...))` 路径时，Agent **必须额外做一件事**：扫描搜索结果的 snippet / 摘要 / 链接列表，**留住**任何看起来像「文字直播 / 文字实录 / play-by-play / commentary」页面的 URL，作为后续 Stage B（见下方 §Stage B: Event Extraction）的事件抓取入口。被 Stage A 截获的 URL 应该在 match dict 里以 `source_url` 字段透传到 Stage B（或由 Agent 内部保留在临时变量里，二选一，按宿主 Agent 习惯实现）。足球可选 provider 若缺失、关闭、无 key 或失败，provider fallback 回到本段 web 路径。

> **如果 Stage A 没嗅探到 URL**：不要在 `build_query` 这一步补搜——Stage A+ Fallback（§`build_pbp_search_query`）会在主循环 §第 1.5 步中按需触发，与主搜索解耦。

**URL 优先级链**（Agent 嗅探时按此顺序匹配，命中第一条就停）：

1. `nba.hupu.com/games/playbyplay/{id}` — 中文 NBA 文字实录（首选）
2. `espn.com/nba/playbyplay/_/gameId/{id}` — 英文 NBA play-by-play（NBA 兜底）
3. `espn.com/soccer/commentary/_/gameId/{id}` — 英文世界杯 / 足球 commentary
4. `match.sports.sina.com.cn/livecast/g/live.php?id={id}` — 新浪世界杯直播间（中文，中等可靠度）
5. `nba.sina.com.cn/...` / `zhibo8.cc/...` — discovery-only（JS 重，纯 GET 多半只拿到壳，仅作发现入口，**不**作为主要事件来源）

完整的 shape / pitfalls / fallback 规则全部写在下方 §Stage B 的 source priority chain 表里，**这里不重复**，避免双源真理。

如果 `include_events=false`（默认值或用户显式关闭），Stage A **跳过** URL 嗅探，搜索结果只用于比分/状态判断，下游 Stage B 整段不执行。这是 §Graceful Degradation 的第一档。

#### `build_pbp_search_query(match_type, home, away)` —— Stage A+ Fallback: 文字直播 URL 精准搜索

**设计动机**：Stage A 的 `build_query` 是通用搜索，返回的搜索结果以比分聚合页为主，很少包含 `nba.hupu.com` 或 `match.sports.sina.com.cn` 的文字直播 URL。当 `pick_playbyplay_url_from_stage_a` 返回 None 时，Stage B 整段跳过 → `events=[]` → 用户看到"进球后几轮播报都无事件"。

**Stage A+ 是一个有节制的 fallback**：
- **仅**在 Stage A 主搜索没找到 playbyplay URL 时触发（由主循环 §第 1.5 步 控制）
- **每场每轮最多 1 次额外搜索**——不是无限制地反复搜索
- 受 `search_freshness_boost` 配置控制（默认 `true`；设为 `false` 可关闭此 fallback，回到旧行为）
- 搜索失败时优雅回退 `events=[]`，不阻塞主循环

```text
def build_pbp_search_query(match_type, home, away):
    """返回 Stage A+ 的搜索查询。
    策略：优先用 site: 限定（Google/Bing 支持），同时附加无 site: 的纯中文关键词
    作为兜底（Exa/语义搜索引擎兼容）。Agent 将整条查询字符串传给 web_search，
    由搜索引擎自行解释——不支持 site: 的引擎会忽略该词但仍匹配中文关键词。"""
    if match_type == "NBA":
        # 虎扑是中文 NBA 文字直播首选源（更新延迟 ~30s）
        # 例: "site:nba.hupu.com/games 湖人 勇士 文字直播 nba.hupu.com"
        team_part = ""
        if home and away:
            team_part = " " + home + " " + away
        return "site:nba.hupu.com/games" + team_part + " 文字直播 nba.hupu.com"
    elif match_type == "世界杯":
        # 新浪是中文足球直播首选源（ESPN 英文源延迟更大）
        # 例: "site:match.sports.sina.com.cn 巴西 德国 直播 match.sports.sina.com.cn"
        team_part = ""
        if home and away:
            team_part = " " + home + " " + away
        return "site:match.sports.sina.com.cn" + team_part + " 直播 match.sports.sina.com.cn"
    else:
        # 未知赛事类型：通用中文搜索
        team_part = ""
        if home and away:
            team_part = " " + home + " " + away
        return match_type + team_part + " 文字直播 play-by-play"
```

> **site: 兼容性说明**：`site:` 是 Google/Bing 支持的语法，Exa 等语义搜索引擎可能忽略它。因此每条查询同时附加了裸域名（如 `nba.hupu.com`）作为兜底——不支持 `site:` 的引擎仍能通过域名关键词匹配到目标页面。

> **为什么不直接在 `build_query` 里加 site: 限定**：`build_query` 同时承担比分搜索和 URL 嗅探双重职责。加 site: 限定会大幅缩小搜索范围，导致比分/状态信息不全（例如 `site:nba.hupu.com` 只返回虎扑页面，拿不到 ESPN scoreboard 的比分）。Stage A+ 将这两个职责解耦——主搜索负责全面覆盖，fallback 负责精准定位实时源。

#### Football Provider Adapter Layer（足球 API 数据源编排）

足球赛事（`match_type == "世界杯"` 或其他 football/soccer 赛事）在进入 Stage A 通用 web search 之前，Agent **必须**先按配置尝试足球 provider chain。Provider 只负责更快、更结构化地给出比分/状态/事件候选；它们不是必需依赖，也不得改变默认可运行性。

所有 provider 都是可选能力：缺少 key、`enabled=false`、被限流、无该赛事覆盖、响应失败、超时、schema 无法归一化时，Agent 必须跳过该 provider 并继续下一个；全部 API provider 不可用时，必须回退到既有 `web_search` / `webfetch` 路径（Stage A / Stage A+ / Stage B），主循环不得因为 provider 失败而中止。

Provider capability flags:

| Provider | score_status | events | commentary | extended_timeline | 说明 |
|---|---:|---:|---:|---:|---|
| `sportradar_extended` | yes | yes | yes | yes | 当 key、套餐与赛事覆盖支持时，优先用于 extended timeline / play-by-play 级别数据。 |
| `sportmonks` | yes | yes | yes | no | 用于比分、事件、文字/解说类 commentary；覆盖和字段以实际响应为准。 |
| `api_football` | yes | yes | yes | no | 用于比分、事件、文字/解说类 commentary；覆盖和字段以实际响应为准。 |
| `football_data_org` | yes | no | no | no | 默认 `status_only=true`，主要校准 `PAUSED` / `FINISHED` 等状态，避免把中场误判为完场。 |
| `web` | yes | yes | yes | no | 既有 Stage A / Stage A+ / Stage B fallback；无 API key 也必须可用。 |

默认 provider 优先级固定为：

```text
sportradar_extended → sportmonks → api_football → football_data_org → web
```

Normalized match snapshot（所有 provider / web fallback 输出都必须归一化为 match dict list；metadata 字段允许 sparse/optional，单场 shape 如下）:

```json
{
  "match_key": "home|away",
  "home": "Brazil",
  "away": "Germany",
  "score": "1-0",
  "phase": "2H",
  "phase_keywords": ["Live", "2H", "进行中"],
  "football_status": "IN_PLAY",
  "minute": "64",
  "possession": {"home": 53, "away": 47},
  "events": [],
  "source": "sportradar_extended",
  "source_freshness_seconds": 8,
  "capabilities": ["score_status", "events", "commentary", "extended_timeline"]
}
```

字段约定：
- `football_status` 是 provider 原始/归一化足球状态的承载位，供后续 `classify_state / football_status_machine` 参考；本节只定义数据流，不展开 Task 3 的状态机细节。
- `source` / `capabilities` 应由 provider 或 `decorate_web_snapshot` 设置；`source_freshness_seconds` 表示 provider 数据相对当前轮询时刻的估计新鲜度，未知时可省略或为 `null`。
- `events` 使用 §parse_search_result event schema；provider 没有事件能力时填 `[]`，不要阻塞 Stage B。
- `capabilities` 只声明本次 snapshot 实际可用能力，不要把 provider 理论能力硬塞进去。
- provider 预置的 `events` / `commentary` 不应被 Stage B 覆盖；Stage B 只追加补充事件，并按 `id_hash` 与 `provider_events` 合并。若 provider 和 Stage B 都没有事件，最终才允许 `m["events"] = []`。

```text
def build_football_provider_chain(config):
    football_cfg = config.get("providers", {}).get("football", {})
    priority = football_cfg.get("priority") or [
        "sportradar_extended",
        "sportmonks",
        "api_football",
        "football_data_org",
        "web",
    ]

    chain = []
    for name in priority:
        if name == "web":
            chain.append({"name": "web", "capabilities": ["score_status", "events", "commentary"]})
            continue

        provider_cfg = football_cfg.get(name, {})
        if not provider_cfg.get("enabled", False):
            continue
        if not provider_cfg.get("api_key"):
            continue

        chain.append({
            "name": name,
            "config": provider_cfg,
            "capabilities": provider_capabilities(name, provider_cfg),
            "timeout_seconds": min(provider_cfg.get("timeout_seconds", 8), 10),
        })

    if not any(p["name"] == "web" for p in chain):
        chain.append({"name": "web", "capabilities": ["score_status", "events", "commentary"]})
    return chain


def fetch_football_snapshot(match_type, follow, config):
    for provider in build_football_provider_chain(config):
        if provider["name"] == "web":
            raw = web_search(build_query(match_type, follow))
            return decorate_web_snapshot(parse_search_result(raw))

        try:
            raw = fetch_provider_once(provider, match_type, follow)
            matches = normalize_provider_response(provider, raw)
            if matches:
                return matches
        except (RateLimit, NoCoverage, TimeoutError, RequestError, NormalizeError):
            log("football provider skipped: " + provider["name"])
            continue

    # Defensive fallback; build_football_provider_chain should already include web.
    raw = web_search(build_query(match_type, follow))
    return decorate_web_snapshot(parse_search_result(raw))


def decorate_web_snapshot(matches):
    for m in matches:
        m["source"] = "web"
        m["capabilities"] = ["score_status", "events", "commentary"]
        # source_freshness_seconds is optional/sparse for web; only set it if known.
    return matches
```

`fetch_provider_once` / `normalize_provider_response` 是 host-agent protocol steps，而不是 SDK 要求：Agent 可以用自身可用的 HTTP/webfetch/shell 网络能力短超时调用 provider，也可以在宿主不支持直接 API 调用时直接跳过 provider。每次 provider 尝试都必须短 timeout、无副作用、失败不抛出到主循环；只有最终 `web` fallback 仍失败时，才进入主流程已有的连续失败计数。

#### Score-Change Event Synthesis（比分变化事件合成）

**设计动机**：即使启用了 Stage A+ Fallback，仍然可能出现 Stage B 没抓到进球事件的场景（webfetch 超时、反爬挡掉、页面 JS-only 拿不到事件，或 provider 只给了非进球 commentary 等）。此时比分已经从 0-0 变成 1-0——用户明明知道有进球，却只看到一条干巴巴的「比分 0-0 → 1-0」推送，没有任何进球事件。

**解决方案**：在主循环 §第 1.7 步，当比分发生变化且当前 events 中没有 existing goal 时，自动合成缺失的进球事件，并保留 provider / Stage B 已有的 non-goal events。这是最后一道防线——确保即使所有搜索和抓取都没给出 goal，用户仍能看到一条「某队进球！」的推送。

**合成规则**：

1. **触发条件**：该场比赛比分发生变化（通过对比 `last_pushed_score` 与当前 `score`）**且** `m.get("events", [])` 中 no existing goal（没有 `type="goal"` 的事件）
2. **合成内容**：根据比分差值生成进球事件。例如比分从 0-0 变成 1-0 → 合成 1 条 `type="goal"` 事件；0-0 变成 2-1 → 合成 3 条事件（2 条主队进球 + 1 条客队进球）
3. **`id_hash` 前缀**：合成事件使用 `"syn:"` 前缀（10 字符），与 Stage B 的真实事件（sha1[:12]）无冲突风险。格式：`"syn:" + sha1((match_key + "|" + score + "|" + team_side + "_goal"))[:10]`
4. **`text` 模板**：`"{队名} 进球！比分 {新比分}"`——简洁但信息充分
5. **`ts` 为空**：合成事件没有精确时间戳（我们只知道"这一轮比分变了"，不知道进球发生的精确时刻）
6. **合并而不覆盖**：`m["events"] = merge_events_by_id(m.get("events", []) + syn_events)`；`merge_events_by_id` 保留同一 `id_hash` 的首次出现，丢弃后续重复 id，确保 provider / Stage B non-goal events 不被合成进球覆盖。

**不触发的场景**：
- 已有 `type="goal"` 的真实事件 → 不需要合成
- `include_events=false` → 整段合成跳过
- 比分未变化 → 不合成
- 首轮（`last_pushed_score is None`）→ 无旧比分可对比，不合成

#### `parse_search_result(raw)` —— 把非结构化搜索结果归一化

`web_search` / `webfetch` 返回的是**非结构化的网页摘要 / HTML / Markdown**，不是 JSON。`classify_state` 和 `build_msg_from_fresh` 不能直接吃原始返回，必须先经过这个解析器归一化为标准结构。

**输入**：搜索工具返回的原始字符串或结构（Markdown 摘要 / HTML 片段 / 搜索结果 list）。

**输出**：标准化的 `matches` 列表，每个元素是字典：

```text
{
  "home":          "Los Angeles Lakers",   # 主队名
  "away":          "Boston Celtics",       # 客队名
  "score":         "98-92",                # 比分字符串；未开始留 ""
  "phase":         "Q3 4:21",              # 节次/阶段；可能是 "Final"/"未开始"/"HT"/"Q3 4:21"
  "phase_keywords":["Live","Q3"],          # 命中的关键词集合（喂给 classify_state）
  "football_status": "",                   # 足球专用归一化状态；非足球可空/省略；足球值见 §football_status_machine
  "minute":        "",                     # 足球比赛分钟，如 "45+3'"；非足球可空/省略
  "source":        "web",                  # web / sportmonks / api_football / football_data_org / sportradar_extended
  "source_freshness_seconds": null,         # 可选：来源新鲜度秒数；未知可空/省略
  "capabilities":  ["score_status"],        # 来源能力标签，如 score_status / events / commentary
  "summary":       "湖人主场 98-92 凯尔特人，第三节还剩 4:21",  # 一句话中文摘要
  "source_url":    "https://...",          # 来源（可选，记录用）
  "raw":           "<原始片段>",            # 透传原文，便于 LLM 兜底
  "events": [                              # 关键事件列表；本场比赛在搜索结果里能识别到的事件
    {
      "id_hash": "<sha1((match_key + '|' + ts + '|' + text).encode('utf-8'))[:12]>",  # 去重键，REQUIRED；算法见下方（必须 byte-equal Stage B）
      "ts":      "21'" | "Q3 4:32" | "HT" | "",         # 比赛时钟字符串，REQUIRED（可为 ""）
      "team":    "home" | "away" | "",                  # 归属队；中性事件留 ""
      "type":    "goal" | "foul" | "sub" | "card" | "key_play" | "other" | "shot" | "save" | "corner" | "free_kick" | "timeout" | "turnover" | "rebound" | "commentary",   # 14 个枚举；后 8 个仅 commentary_style="play_by_play" 时使用（与 §llm_extract_events VALID_TYPES byte-equal）
      "text":    "≤80 个中文字符的单行描述"
    }
  ]
}
```

字段规则：`football_status` 的合法值由 §football_status_machine 定义；非足球比赛可以省略或留空。足球的 `HALFTIME` / `SUSPENDED` / `UNKNOWN_LIVE` 都属于 live polling 状态，**永不触发 final**。

**`events` 字段说明** (schema field name = `events:` array of event dicts)：

- `events` **可以是空列表 `[]`**，含义：本场目前无可识别事件，或抓取/解析事件失败。空列表是合法值，不视为解析错误。
- `events` 仅承载「本轮搜索结果中能直接读出来的事件」，`parse_search_result` 不主动发起额外请求去补全事件（拉取逻辑属于其他子函数的职责）。
- 字段语义：
  - `id_hash`：长度恰好 12 的十六进制字符串，跨轮去重的唯一键，必填。
  - `ts`：游戏内时钟字符串，**保留原始风格**——足球用 `"21'"` / `"45+2'"` / `"HT"`；篮球用 `"Q3 4:32"` / `"OT 1:05"`；无法获取时填 `""`。必填字段，但允许 `""`。`commentary` 类型事件**允许 `ts` 为 `""`**（叙述性事件不总是有精确时钟，此时 `id_hash` 仍由 `match_key + "|" + "" + "|" + text` 计算）。
  - `team`：`"home"` / `"away"` 之一；裁判/中立事件填 `""`。
  - `type`：枚举值之一；不在枚举内的杂项一律归到 `"other"`，不要新造字符串。当 `commentary_style == "events_only"` 时仅使用前 6 个类型（goal/foul/sub/card/key_play/other）；当 `commentary_style == "play_by_play"` 时可使用全部 14 个类型。各扩展类型含义：
    - `shot`：射门 / 投篮未中（球权未转化为得分）
    - `save`：扑救 / 封盖（守门员扑出 / 篮球盖帽）
    - `corner`：角球
    - `free_kick`：任意球 / 罚球（未得分的那种）
    - `timeout`：暂停（篮球暂停 / 足球医疗暂停等）
    - `turnover`：失误 / 丢球权（传球失误、被抢断等）
    - `rebound`：篮板球
    - `commentary`：文字解说叙述行——当无具体技术事件但需描述比赛节奏/局势时使用（如「双方中场争夺激烈，均无威胁进攻」）。这是 `play_by_play` 模式的**兜底类型**，保证每轮至少有一条输出。
  - `text`：单行中文描述，长度 ≤ 80 个字符（按字符数算，不是字节数）；不要含换行/Markdown。

**`id_hash` 计算规则（必须严格按此实现，4 家 Agent 一致）**：

```text
match_key = home + "|" + away                         # 例: "Los Angeles Lakers|Boston Celtics"
id_hash   = sha1((match_key + "|" + ts + "|" + text).encode("utf-8")).hexdigest()[:12]
```

- `match_key` 用 `home + "|" + away` 拼接，目的是**防止跨场比赛碰撞**：两场不同比赛若各自出现「第 21 分钟射门」这种相似文本，单纯哈希 `ts + text` 会撞到同一个 `id_hash`，引入 `match_key` 即可隔离。
- 拼接时**必须**用 `"|"` 作为 `match_key` / `ts` / `text` 三段之间的硬分隔符，并显式 `.encode("utf-8")` 后再 sha1——这样 `("AB","C","D")` 与 `("A","BC","D")` 不会撞到同一个 hash。**所有调用 `id_hash` 计算的位置（§parse_search_result、§Stage B 的 `llm_extract_events`）必须 byte-equal 这一行表达式**，否则 Stage A 与 Stage B 算出的 hash 不互通 → `seen_event_ids` ring buffer 跨 stage 去重失效 → 同一事件被推送两次。
- 拼接时不做大小写归一化，`home` / `away` / `ts` / `text` 直接按 `parse_search_result` 输出的字符串原样进帐——只要解析器对同一事件输出稳定，哈希就稳定。
- 截取前 12 个十六进制字符（48 bit）已足够区分同一场比赛量级的事件数量，不要改成 8 / 16 等其他长度，避免与 §state file schema 的 `seen_event_ids` 不匹配。

**Worked example**（一场假想的湖人 vs 凯尔特人比赛，解析器应给出类似下面的 match dict）：

```text
{
  "home": "Los Angeles Lakers",
  "away": "Boston Celtics",
  "score": "98-92",
  "phase": "Q3 4:21",
  "phase_keywords": ["Live", "Q3"],
  "football_status": "",
  "minute": "",
  "source": "web",
  "source_freshness_seconds": null,
  "capabilities": ["score_status"],
  "summary": "湖人主场 98-92 凯尔特人，第三节还剩 4:21",
  "source_url": "https://example.com/box/lal-bos",
  "raw": "<原始 HTML 片段>",
  "events": [
    {
      "id_hash": "a1b2c3d4e5f6",
      "ts":      "Q2 3:12",
      "team":    "home",
      "text":    "詹姆斯快攻暴扣，湖人反超 2 分",
      "type":    "key_play"
    },
    {
      "id_hash": "0f9e8d7c6b5a",
      "ts":      "Q3 6:48",
      "team":    "away",
      "text":    "塔图姆三分命中，凯尔特人追到只差 1 分",
      "type":    "goal"
    },
    {
      "id_hash": "112233445566",
      "ts":      "Q3 4:21",
      "team":    "home",
      "text":    "戴维斯第 4 次犯规，下场休息",
      "type":    "foul"
    }
  ]
}
```

足球场景同理：`ts` 写 `"21'"`、`type` 用 `"goal"` / `"card"` / `"sub"` / `"other"`，`text` 例如 `"梅西禁区内推射破门，阿根廷 1-0"`。

**LLM-side 解析指引**（Agent 必须按此顺序尝试）：

1. **关键词扫描** —— 在原始文本里 grep 中英文关键词（见 `classify_state` 的关键词列表 + 比分正则 `\d{1,3}\s*[-:–]\s*\d{1,3}`），命中即填进对应字段
2. **多比赛分割** —— 一次搜索常返回多场比赛；用空行 / `<li>` / Markdown 列表项 / 队名对边界切分，每场各自走 1 步
3. **LLM 兜底** —— 如果正则/关键词覆盖不到（例如返回的是带表格的 HTML），让 LLM 自己读 `raw` 然后填充上述字典；这是允许且推荐的，因为 LLM 本来就在循环里
4. **歧义保护** —— 足球按 §football_status_machine 先归一 `football_status`；非足球若 `phase_keywords` 同时命中「Final」+「Live/HT/中场」，先按 live-like 处理，避免半场误判终场。只有比分没有阶段词时填 `phase="进行中"` 并把 `Live` 加进 keywords，让 `classify_state` 判直播
5. **空结果** —— 如果原始返回里**完全没有比赛**（搜索没命中、或当天没赛事），返回 `[]`；调用方应按「未开始」分支处理或直接告诉用户「今天没比赛」

**禁止**：
- 不要让 Agent 自己写 `.py` 解析脚本——用 LLM 直接读 `raw` 比写脚本更稳
- 不要硬编码 ESPN/官方 API 的 JSON schema——本 Skill 不打那些 API
- 不要在 `parse_search_result` 里就发推送——它只做解析，副作用为零

#### Stage B: Event Extraction （事件抓取）

**目的（PURPOSE）**：从一个**纯 HTTP GET 即可拉到**（不需要 API key、不需要 JS 渲染）的「文字直播 / 文字实录 / play-by-play / commentary」页面，抽取每场比赛的逐事件流，输出严格符合 §parse_search_result event schema 的事件列表，喂给每个 match dict 的 `events: [...]` 字段。

Stage B 是 Stage A 的下游、`classify_state` 的旁路；它**只负责把 events 字段填上**，不做去重也不做推送（去重 + 推送门控由后续 commit 在主循环里处理，不在本节范围内）。

##### Source Priority Chain（来源优先级链）

Agent 必须按下表从 1 → 5 依次尝试，命中第一条且能拉到事件就停止；命中失败则降级到下一条；全部失败则该场比赛 `events = []`。

| 排名 | 赛事 | 语言 | URL 模式 | 期望 HTML/文本形态 | 已知 pitfalls |
|------|------|------|---------|-------------------|--------------|
| 1 | NBA | 中文 | `https://nba.hupu.com/games/playbyplay/{hupu_game_id}` | 静态 HTML，表格行 `(时间, 球队, 事件, 比分)` | `hupu_game_id` 需通过 `web_search` 嗅探（例：`site:nba.hupu.com/games/playbyplay 湖人 勇士`），不能瞎猜 |
| 2 | NBA | 英文 | `https://www.espn.com/nba/playbyplay/_/gameId/{espn_game_id}` | 「PLAYS」section，按节（quarter）+ 时间 + 描述分块 | `gameId` 通过 `https://www.espn.com/nba/scoreboard/_/date/{YYYYMMDD}` 嗅探；响应式 markup **会重复输出同一事件**，Agent 必须按 `(quarter, time, event_text)` 三元组去重 |
| 3 | 世界杯 / 足球 | 英文 | `https://www.espn.com/soccer/commentary/_/gameId/{espn_game_id}` | 「Play By Play」流，含 `21'` / `45+2'` / `HT` / `FT` 等时钟，goal/card/sub 注释 + 偶尔编辑评述 | `gameId` 通过 `https://www.espn.com/soccer/scoreboard/_/league/fifa.world` 嗅探；仅英文；部分行是散文式评述，LLM 抽取时**保持简洁**（≤80 字符），避免把整段评论塞进 `text` |
| 4 | 世界杯 / 足球 | 中文 | `http://match.sports.sina.com.cn/livecast/g/live.php?id={sina_match_id}` | 直播间外壳，事件流**有时**是 JS 后注入 | GBK / GB2312 解码（`webfetch` 默认按 UTF-8 解可能乱码，需要在 LLM 端识别乱码并提示降级）；纯 GET **不一定**能拿到事件行——把它当 **discovery / 兜底**，不要当主要 commentary 来源 |
| 5 | NBA / 世界杯 | 中文 | `nba.sina.com.cn/...` / `zhibo8.cc/...` | 多为 SPA 壳，纯 GET 通常只拿到 `<div id="app"></div>` | **discovery-only**，标记为低优先级；Agent 拉到只看到壳就立即放弃，不要尝试解析 |

##### 流水线步骤（PIPELINE STEPS）

```text
# Step 1: 对 Stage A 已经识别为「正在直播」的每一场 match in matches:
# P2-2 (Oracle Round 10)：match dict (§parse_search_result) 没有 .state 字段。
# 必须通过 phase_keywords 判定 live；不能写 match.state（AttributeError / KeyError）。
# CRITICAL（P1-1, Oracle Round 12）：match 是 dict（§parse_search_result 输出），
# 整段伪代码必须用 match["events"] / match["home"] / match["away"] 形式访问。
# 绝不写 match.events / match.home / match.away——同 P0-3 那个 getattr-on-dict 静默失败陷阱。
for match in matches:
    phase_kws = match.get("phase_keywords", []) or []
    live_kws  = {"Live", "直播中", "Q1", "Q2", "Q3", "Q4", "OT", "1H", "2H",
                 "上半场", "下半场", "加时", "进行中", "中场"}
    if not any(kw in phase_kws for kw in live_kws):
        continue                                      # 未开始/已完结不抓事件

    # 1a. URL 发现优先级：缓存 > Stage A 嗅探 > Stage A+ 补搜
    #     （与 §主循环 第 1.5 步完全一致，这里列出等价逻辑供参考）
    match_key_for_url = match.get("home", "") + "|" + match.get("away", "")

    # 1a-1. 优先使用跨轮 URL 缓存
    pbp_url = persisted.get("pbp_urls", {}).get(match_key_for_url)

    # 1a-2. 缓存 miss → 从 Stage A 截获的搜索结果 snippets / 链接里嗅探
    if pbp_url is None:
        pbp_url = pick_playbyplay_url_from_stage_a(match)

    # 1a-3. 找不到 URL → 尝试 Stage A+ Fallback（见 §build_pbp_search_query）
    #     Stage A+ 用 site: 限定搜索精准定位中文实时源，受 search_freshness_boost 配置控制
    if pbp_url is None:
        if config.get("search_freshness_boost", True):
            pbp_q = build_pbp_search_query(match_type, match.get("home", ""), match.get("away", ""))
            try:
                pbp_raw = web_search(pbp_q)
                pbp_match = {"source_url": "", "raw": pbp_raw or ""}
                pbp_urls = extract_urls_from_text(pbp_raw or "")
                if pbp_urls:
                    pbp_match["source_url"] = pbp_urls[0]
                pbp_url = pick_playbyplay_url_from_stage_a(pbp_match)
            except Exception:
                pbp_url = None

    # 1a-4. 写入 URL 缓存
    if pbp_url is not None:
        if "pbp_urls" not in persisted:
            persisted["pbp_urls"] = {}
        persisted["pbp_urls"][match_key_for_url] = pbp_url

    if pbp_url is None:
        match["events"] = []
        log("no play-by-play source for " + match.get("home", "") + " vs " + match.get("away", "") + ", skip Stage B")
        continue

# Step 2: 对每个有 URL 的 match：
    # 2a. 用宿主 Agent 自带的 webfetch 拉一次 markdown
    #     （opencode=webfetch / Claude Code=WebFetch / Codex=shell+curl / Cursor=内置）
    #     —— 工具映射严格遵从本文件顶部的「§各宿主 Agent 工具映射表」，本节不重复列表
    try:
        page_md = webfetch(pbp_url, format="markdown", timeout_seconds=15)
    except (timeout / http_error / empty_body):
        match["events"] = []
        log_warning("Stage B fetch failed for " + pbp_url + ", events=[]")
        continue

    if page_md is empty or page_md is shell-only:
        match["events"] = []
        continue

    # 2b. LLM-side 抽取：让 LLM 直接读 page_md，按 §parse_search_result event schema 输出
    #     —— 不要写 .py 脚本、不要正则裸抽，LLM 已经在循环里
    #     —— P0-2 (Oracle Round 12): 此处不传 cap，因为函数签名是 (page_md, match_key, schema)；
    #        cap 的修剪已经在下方 2c 里通过 raw_events[-cap_b:] 完成。
    raw_events = llm_extract_events(
        page_md,
        match_key = match.get("home", "") + "|" + match.get("away", ""),
        schema    = "see §parse_search_result event schema (id_hash + ts + team + type + text)"
    )

    # 2c. 只保留最新的 2 × max_events_per_msg 条（默认 max_events_per_msg=6 → 取 12 条）
    #     超额抓取是为了给后续 commit 的去重逻辑（seen_event_ids 过滤）留缓冲，
    #     避免过滤完直接空掉。
    cap_b = (3 if config.get("commentary_style", "events_only") == "play_by_play" else 2) * config["max_events_per_msg"]   # play_by_play: 3×=24; events_only: 2×=12
    match["events"] = raw_events[-cap_b:]      # 取末尾 cap_b 条（最新）

    # 2d. id_hash 计算严格复用 §parse_search_result 已规定的算法（byte-equal）：
    #     id_hash = sha1((match_key + "|" + ts + "|" + text).encode("utf-8")).hexdigest()[:12]
    #     四家 Agent 必须用同一份算法，跨 Agent / 跨 Stage 同事件 → 同 hash，下游去重才一致。

# Step 3: 修剪 + 去重 = 后续 commit 的工作（在主循环里做）
#         Stage B 在这里只负责把 match["events"] 填到 ≤ 2× cap，不读 seen_event_ids，
#         不写 state_file，不发推送。零副作用。
```

##### `pick_playbyplay_url_from_stage_a(match)` —— Stage A → Stage B URL 选择器

**P0-1 fix (Oracle Round 11)**：本函数必须在所有 4 家 Agent 中以**完全相同**的算法实现。绝不让 LLM 自由发挥——按下方决策树字面量执行，否则跨 Agent 不一致。

**输入**：单场 match dict（来自 §parse_search_result，含 `home`/`away`/`source_url`/`raw` 字段）

**输出**：play-by-play URL 字符串，或 `None`（找不到任何匹配候选）

**算法（决策树，严格按顺序匹配，找到第一个就返回）**：

```text
def pick_playbyplay_url_from_stage_a(match):
    """
    扫描 match 自身携带的 URL 候选池，按 §Stage B 5 行优先级表逐行匹配；
    第一行命中即返回该 URL，零命中返回 None。
    禁止：发起新的 web_search、调用任何 LLM、向其他场比赛借 URL。
    """
    # 1. 收集候选 URL 池（仅来自当前 match 自身，不跨场借用）
    candidates = []
    if match.get("source_url"):
        candidates.append(match["source_url"])
    raw_text = match.get("raw", "") or ""
    candidates.extend(extract_urls_from_text(raw_text))   # 见下方辅助函数
    # 去重，保留首次出现顺序
    seen = set(); pool = []
    for u in candidates:
        if u and u not in seen:
            seen.add(u); pool.append(u)
    if not pool:
        return None

    # 2. 按 §Stage B 源优先级表（行 1→5）依次匹配，第一行命中即返回
    #    NOTE：严格按表内字面量 host 段匹配，不做模糊化、不调 LLM 判断
    PRIORITY_PATTERNS = [
        # (行号, 联赛域, host 子串列表, path 子串列表)
        (1, "NBA",    ["nba.hupu.com"],
                      ["/games/", "/livecast"]),                 # 行 1: 虎扑 NBA 文字直播（首选）
        (2, "NBA",    ["espn.com"],
                      ["/nba/playbyplay"]),                      # 行 2: ESPN NBA play-by-play（兜底，英文）
        (3, "soccer", ["espn.com"],
                      ["/soccer/commentary", "/soccer/match"]),  # 行 3: ESPN soccer commentary（足球首选）
        (4, "soccer", ["match.sports.sina.com.cn",
                       "sports.sina.com.cn"],
                      ["/livecast", "/live.php"]),               # 行 4: 新浪足球直播间（兜底, 可能 SPA）
        (5, "any",    ["nba.sina.com.cn", "zhibo8.cc"],
                      []),                                       # 行 5: discovery-only / SPA 壳（最低优先级）
    ]

    for _row, _league, host_subs, path_subs in PRIORITY_PATTERNS:
        for url in pool:
            host = url_host(url)                    # e.g. "nba.hupu.com"
            path = url_path(url)                    # e.g. "/games/12345"
            if not any(h in host for h in host_subs):
                continue
            if path_subs and not any(p in path for p in path_subs):
                continue
            return url                              # 命中第一行，返回

    # 3. 5 行全部 miss → None（Stage B 进入 graceful=[] 路径）
    return None


# === 辅助函数（4 家 Agent 必须用等价实现） =================================
def extract_urls_from_text(text):
    """从任意文本里抽取 http(s):// URL；纯字符串扫描，无 LLM。"""
    # Python:  re.findall(r"https?://[^\s)\]\"'<>]+", text)
    # JS:      text.match(/https?:\/\/[^\s)\]"'<>]+/g) || []
    # 返回 list[str]，保持出现顺序。
    ...

def url_host(url):
    """返回 URL 的 host 段（小写）。例: https://nba.hupu.com/games/123 → "nba.hupu.com"。"""
    # Python:  urllib.parse.urlparse(url).hostname.lower() or ""
    # JS:      new URL(url).hostname.toLowerCase()
    ...

def url_path(url):
    """返回 URL 的 path 段。例: https://nba.hupu.com/games/123?x=1 → "/games/123"。"""
    # Python:  urllib.parse.urlparse(url).path or ""
    # JS:      new URL(url).pathname
    ...
```

**确定性保证**：4 家 Agent 给定同一份 `match` dict、同一份 URL 候选池，**必须**返回同一个 URL（或同时 None）。`PRIORITY_PATTERNS` 表是单一真理源——任何 host/path 子串调整必须同步改本表，禁止 Agent 自行扩列表。


##### `llm_extract_events(page_md, match_key, schema)` —— Stage B LLM 事件抽取

**P0-2 fix (Oracle Round 11)**：本函数被 §Stage B Step 2b 调用但缺定义。4 家 Agent 必须按下方 prompt 字面量执行 LLM 调用，否则跨 Agent 抽出的 event 集合不一致 → `id_hash` 不一致 → 去重失效 → 重复推送同一事件。

**输入**：
- `page_md`：webfetch 拿到的 markdown（已剥 HTML，含文字直播主体）
- `match_key`：`match.home + "|" + match.away`（用于算 id_hash 时拼前缀）
- `schema`：字符串描述（仅用于 prompt 提示，实际输出格式见下方 JSON schema）

**输出**：list of event dict，每个 dict 严格如下结构：

```json
{
  "id_hash": "12-char hex",       // sha1((match_key + "|" + ts + "|" + text).encode("utf-8"))[:12]
  "ts":      "MM:SS or P-MM:SS",  // 比赛内时钟 (e.g. "Q3-08:32" / "45+2'" / "10:24")
  "team":    "home|away|null",    // 事件归属，无法判定时填 null
  "type":    "goal|foul|sub|card|key_play|other|shot|save|corner|free_kick|timeout|turnover|rebound|commentary",   // 枚举值之一（后 8 个仅 play_by_play 模式）
  "text":    "string"             // 一行人类可读描述 (≤ 80 字符)
}
```

**算法（伪代码，4 家 Agent 用同一份 prompt + 同一份解析逻辑）**：

```text
def llm_extract_events(page_md, match_key, schema):
    """
    把 webfetch 拿到的文字直播 markdown 喂给 LLM，让它按 schema 抽取事件。
    禁止：写正则裸抽 / 写 .py 脚本 / 调外部 NLP 服务。
    硬约束：4 家 Agent 必须用下方字面 prompt，连标点都不许改——
            因为 prompt 字面差异会导致同一段 markdown 抽出不同的事件文本，
            进而 id_hash 不一致 → 跨 Agent 去重失效。

    参数 `schema` 是字符串文档锚点（用于审计 trace），原文会被注入 prompt 的
    "== schema 参考 ==" 段落里。它**不**改变下方 5 字段定义——5 字段定义是这份
    prompt 的字面真理源，调用方传进来的 schema 串只是给 LLM 一个交叉引用的提示。

    （隐式依赖 `config.commentary_style`：决定使用 events_only prompt 还是 play_by_play prompt。
    commentary_style 来自主循环 §read_config 启动时读取——llm_extract_events 不自行读配置。）
    """
    # 1. 输入大小防御：超过 60KB 截断（防止 prompt 爆 LLM 上下文）
    if len(page_md) > 60_000:
        page_md = page_md[-60_000:]   # 取末尾，因为最新事件通常在页面底部

    # === events_only prompt (commentary_style == "events_only") ===
    # This is the ORIGINAL prompt, unchanged. Only used when commentary_style == "events_only".
    prompt_events_only = """你是一个体育赛事文字直播解析器。下面是一段 markdown 格式的比赛文字直播页面。

请按时间倒序扫描，抽取最近发生的所有事件，每条事件必须包含 5 个字段：

- ts:   比赛内时钟字符串。NBA 用 "Q1-11:45" / "Q4-02:13" / "OT-04:00"；足球用 "12'" / "45+2'" / "67'"；其他用页面原始时钟字符串。
- team: 事件归属方。明确属于主队填 "home"，客队填 "away"，无法判定填 null。**不要瞎猜**——宁可填 null。
- type: 必须是以下 6 个枚举值之一：
        - goal      (进球 / 得分 / 投篮命中 / 三分球)
        - foul      (犯规 / 违例)
        - sub       (换人)
        - card      (黄牌 / 红牌)
        - key_play  (关键回合 / 抢断 / 盖帽 / 助攻 / 关键传球)
        - other     (其他不便归类的)
- text: 一行人类可读描述，必须 ≤ 80 个字符。中文优先。不要复述时钟。例：「詹姆斯三分命中」「孙兴慜被换下」。
- id_hash: 留空字符串 ""，由调用方计算填入，**LLM 不要算**。

输出格式：严格的 JSON 数组，不许有任何 markdown 围栏（不要 ```json）、不许有解释文字。
顺序：按页面里的事件出现顺序（通常是时间倒序，最新在前）。
数量：最多输出 30 条，超过的略掉。
找不到任何事件 / 页面是空壳 / 解析失败 → 输出 `[]`（空数组）。

== schema 参考 ==
%SCHEMA%

== 比赛标识 ==
%MATCH_KEY%

== 文字直播页面（markdown） ==
%PAGE_MD%
"""

    # === play_by_play prompt (commentary_style == "play_by_play") ===
    # HARD CONTRACT: 4 家 Agent 不许改字符（与 events_only prompt 同等约束）
    prompt_play_by_play = """你是一个体育赛事文字直播解说员。下面是一段 markdown 格式的比赛文字直播页面。

请按时间倒序扫描，提取**所有**最近发生的比赛进程——不仅仅是进球/犯规等重大事件，还包括射门、扑救、角球、暂停、失误等常规动作，以及比赛节奏/局势的叙述。每条事件必须包含 5 个字段：

- ts:   比赛内时钟字符串。NBA 用 "Q1-11:45" / "Q4-02:13" / "OT-04:00"；足球用 "12'" / "45+2'" / "67'"；其他用页面原始时钟字符串。
- team: 事件归属方。明确属于主队填 "home"，客队填 "away"，无法判定填 null。**不要瞎猜**——宁可填 null。
- type: 必须是以下 14 个枚举值之一：
        - goal      (进球 / 得分 / 投篮命中 / 三分球 / 点球命中)
        - foul      (犯规 / 违例)
        - sub       (换人)
        - card      (黄牌 / 红牌)
        - key_play  (关键回合 / 绝杀 / 扑点)
        - shot      (射门 / 投篮未中——球权未转化为得分)
        - save      (扑救 / 盖帽——守门员扑出或篮球封盖)
        - corner    (角球)
        - free_kick (任意球 / 罚球未中)
        - timeout   (暂停)
        - turnover  (失误 / 丢球权 / 被抢断)
        - rebound   (篮板球)
        - commentary (文字解说叙述行——用于描述比赛节奏/局势/无具体技术事件时的状况)
        - other     (其他不便归类的)
- text: 一行人类可读描述，必须 ≤ 80 个字符。中文优先。风格要求：**像文字解说员一样叙述**，例：「詹姆斯突破上篮被戴维斯封盖」「中场争夺激烈，双方均无威胁进攻」。不要只写冰冷的技术统计——要有画面感。
- id_hash: 留空字符串 ""，由调用方计算填入，**LLM 不要算**。

⚠ 重要：如果页面中近期无任何具体技术事件（无射门/犯规/换人等），你**必须**至少输出 1 条 type="commentary" 的事件来描述当前比赛状态，例：「比赛进行到第65分钟，双方中场争夺，均无威胁进攻」或「第二节中段，双方陷入拉锯战」。**绝不**输出空数组 `[]`——除非页面是空壳或比赛未开始。

输出格式：严格的 JSON 数组，不许有任何 markdown 围栏（不要 ```json）、不许有解释文字。
顺序：按页面里的事件出现顺序（通常是时间倒序，最新在前）。
数量：最多输出 40 条，超过的略掉。
页面是空壳 / 解析失败 → 输出 `[]`（空数组）。

== schema 参考 ==
%SCHEMA%

== 比赛标识 ==
%MATCH_KEY%

== 文字直播页面（markdown） ==
%PAGE_MD%
"""

    # 2c. Prompt selection based on commentary_style
    if config.get("commentary_style", "events_only") == "play_by_play":
        prompt = prompt_play_by_play
    else:
        prompt = prompt_events_only

    # Apply variable substitution
    prompt = prompt.replace("%SCHEMA%", str(schema or "")).replace("%MATCH_KEY%", match_key).replace("%PAGE_MD%", page_md)

    # 3. 让宿主 Agent 自己 reason（IMPORTANT：这不是个真函数——
    #    `host_agent_self_reason()` 只是伪代码占位符，表示「把上面的 prompt 当作
    #    本 Agent 自己的下一步 reasoning 输入，让 LLM backbone 自己产出 JSON 字符串」）。
    #    具体到 4 家 Agent 的语义：
    #    - codex / cursor / opencode / claude code：在轮询循环的当前 turn 内，
    #      Agent 把 prompt 文本当作下一个推理步的输入——本质上就是 Agent 自己「读了
    #      page_md，照着 prompt 描述抽事件」，输出落到一个变量（伪代码记为 llm_raw）。
    #    禁止调外部 LLM API（OpenAI / Anthropic / Bailian 等独立 API）——因为：
    #    (a) 用户没在 config 里配 API key
    #    (b) 跨 Agent 走不同 LLM 后端会破坏「字面相同」的硬约束 → id_hash 漂移
    try:
        # llm_raw = <this Agent's own reasoning output, given `prompt` as input>
        llm_raw = host_agent_self_reason(prompt, max_tokens=4000, temperature=0.0)
    except (timeout / llm_error):
        log_warning("llm_extract_events: LLM call failed, returning []")
        return []

    # 4. 解析 + 校验
    try:
        events = json.loads(llm_raw.strip())
    except (JSONDecodeError):
        # LLM 偶尔会包 ```json ... ``` 围栏，做一次容错剥离
        stripped = strip_markdown_fences(llm_raw)
        try:
            events = json.loads(stripped)
        except (JSONDecodeError):
            log_warning("llm_extract_events: cannot parse LLM output as JSON, returning []")
            return []

    if not isinstance(events, list):
        log_warning("llm_extract_events: LLM did not return a list, returning []")
        return []

    # 5. 逐条校验 + 计算 id_hash + 强行修剪
    # 5a. VALID_TYPES selection based on commentary_style
    VALID_TYPES_EVENTS_ONLY = {"goal", "foul", "sub", "card", "key_play", "other"}
    VALID_TYPES_PLAY_BY_PLAY = {"goal", "foul", "sub", "card", "key_play", "other",
                                "shot", "save", "corner", "free_kick",
                                "timeout", "turnover", "rebound", "commentary"}
    # IMPORTANT: commentary_style comes from config read at startup (§read_config).
    # llm_extract_events reads config.commentary_style directly to select VALID_TYPES,
    # hard_cap, and commentary fallback behavior. config is assumed to be in scope
    # (module-level or closure variable, same convention as other pseudocode functions).
    valid_types = VALID_TYPES_PLAY_BY_PLAY if config.get("commentary_style", "events_only") == "play_by_play" else VALID_TYPES_EVENTS_ONLY

    hard_cap = 40 if config.get("commentary_style", "events_only") == "play_by_play" else 30   # play_by_play produces more entries
    cleaned = []
    for ev in events[:hard_cap]:
        if not isinstance(ev, dict):
            continue
        ts   = str(ev.get("ts",   "")).strip()
        team = ev.get("team")
        if team not in ("home", "away", None):
            team = None
        ev_type = str(ev.get("type", "other")).strip().lower()
        if ev_type not in valid_types:
            ev_type = "other"
        text = str(ev.get("text", "")).strip()[:80]   # 80 字符硬截断

        if not text:                             # text 为空 → 丢弃
            continue
        # commentary 类型允许 ts 为空（叙述性事件不总是有精确时钟）
        if not ts and ev_type != "commentary":
            continue

        # id_hash 算法 = §parse_search_result 已规定，4 家 Agent byte-equal
        id_hash = sha1((match_key + "|" + ts + "|" + text).encode("utf-8")).hexdigest()[:12]

        cleaned.append({
            "id_hash": id_hash,
            "ts":      ts,
            "team":    team,
            "type":    ev_type,
            "text":    text,
        })

    # 6. play_by_play commentary fallback：保证文字直播模式每轮至少有 1 条叙述
    #    若 LLM 未产出任何事件（返回空数组或全部被校验丢弃），注入一条合成 commentary。
    #    这是「文字直播每轮必推」承诺的兜底——纯靠 prompt 指令不够（LLM 可能不遵守）。
    if config.get("commentary_style", "events_only") == "play_by_play" and len(cleaned) == 0:
        fallback_text = "比赛进行中，暂无详细事件数据"
        fallback_hash = sha1((match_key + "|" + "" + "|" + fallback_text).encode("utf-8")).hexdigest()[:12]
        cleaned.append({
            "id_hash": fallback_hash,
            "ts":      "",
            "team":    None,
            "type":    "commentary",
            "text":    fallback_text,
        })

    return cleaned


# === 辅助：剥 markdown 围栏（``` 开头/结尾） =================================
def strip_markdown_fences(text):
    """
    输入: '```json\n[{"a":1}]\n```'  → 输出: '[{"a":1}]'
    输入: '```\n[1,2]\n```'           → 输出: '[1,2]'
    没围栏: 原样返回。
    """
    s = text.strip()
    if s.startswith("```"):
        # 去掉首行 ``` 或 ```json
        first_nl = s.find("\n")
        if first_nl != -1:
            s = s[first_nl+1:]
    if s.endswith("```"):
        s = s[:-3].rstrip()
    return s
```

**确定性保证**：4 家 Agent 给定同一份 `page_md` + 同一个 `match_key`，`temperature=0.0` 下 LLM 输出**应当**字符一致；即便有 1–2 字差异，经过 `cleaned` 步骤的字段标准化 + `id_hash` 计算后，集合应等价。如果某家 Agent 的 LLM 后端不支持 `temperature=0.0`，必须填它支持的最低温度（如 Codex 默认走的是 GPT-5 / Claude Sonnet 4，都支持 `0.0`）。

**为什么禁止外部 LLM API**：(a) 用户 config 里没要求填 OPENAI_API_KEY；(b) 跨 Agent 走不同后端 → prompt 解析路径不同 → events 集合不同 → id_hash 不同 → 去重失效 → 重复推送。

##### Graceful Degradation（优雅降级 / 优雅跳过）

Stage B 任何一步失败都**不**应该影响主循环的比分推送。明文规定：

| 触发条件 | 处理 | 主循环影响 |
|---------|------|-----------|
| `config.include_events == false`（master switch off） | **整段 Stage B 跳过**；所有 match `events = []` | 仅推送比分 |
| Stage A 没嗅探到 play-by-play URL | 该场 `events = []`，记录一行 log | 仅推送该场比分 |
| `webfetch` timeout / HTTP 4xx/5xx / 空 body | 该场 `events = []`，warning 级别 log | 仅推送该场比分 |
| `webfetch` 拉到只是 SPA 壳（无可解析事件） | 该场 `events = []` | 仅推送该场比分 |
| LLM 抽取后 `raw_events` 为空 | 该场 `events = []` | 仅推送该场比分 |

**核心约束**：Stage B failures **NEVER** abort the main loop。score-only push 永远继续，事件流只是**锦上添花**——拉不到就当本轮没拉到，下一轮再试。

##### Performance Budget（性能预算）

- **每场比赛每轮 1 次 `webfetch`**——不要为了「更全」连续多打几个 URL。
- 单次 `webfetch` 软超时 **≤ 15 秒**（落在主循环 60 秒 sleep chunk 的预算内）。
- 典型场景：**N ≤ 3 场**正在直播 × 5–15 秒/次 = 15–45 秒，整体落在一次 60 秒 sleep chunk 内，不会破坏 §chunked sleep 的节奏。
- **如果 N > 3**：Agent **必须裁到 3 场最相关**，否则总抓取耗时可能逼近 60 秒预算，挤压主循环其他步骤。裁剪优先级（高 → 低）：
  1. 命中用户 `follow` 过滤的比赛
  2. 主流联赛 / 主流赛事（NBA 季后赛 > 常规赛；世界杯 > 友谊赛）
  3. 其他

##### Host Agent Consistency（宿主 Agent 一致性）

四家宿主 Agent（**codex / cursor / opencode / claude code**）必须运行**字面相同**的 Stage B 伪代码——只有「`webfetch` 这一个调用换成各自的工具名」这一个区别。

具体每家 Agent 用哪个工具拉网页 → 严格参考本文件顶部的 **§各宿主 Agent 工具映射表**（第「网络搜索」列里 `webfetch` / `WebFetch` / `shell + curl` / 内置 web search 的对应关系）。**本节不重复列表**——一处真理，避免漂移。

判定 Stage B 跨 Agent 一致的硬性条件：

1. 同一场比赛的同一条事件，**4 家 Agent 算出的 `id_hash` 必须 byte-equal**（前提：四家拿到的 `home` / `away` / `ts` / `text` 一致 —— 这正是为什么 `parse_search_result` 严格规定字段格式）
2. 同一份 markdown 输入，**4 家 Agent 抽出的 events 列表元素集合应当一致**（顺序可有微小差异，集合相等即可）
3. 同样的失败条件，**4 家 Agent 都返回 `events = []`** —— 不许某家 Agent 抛错把主循环打断

#### football_status_machine（足球半场/终场状态机）

足球状态必须先归一化，再交给 `classify_state(matches)`。内部状态只允许使用：

```text
SCHEDULED
FIRST_HALF
HALFTIME
SECOND_HALF
EXTRA_TIME
PENALTY_SHOOTOUT
SUSPENDED
FINISHED
UNKNOWN_LIVE
```

状态到外部三态的映射：

| football_status | classify_state 返回 |
|-----------------|---------------------|
| `SCHEDULED` | `未开始` |
| `FINISHED` | `已完结` |
| `FIRST_HALF` / `HALFTIME` / `SECOND_HALF` / `EXTRA_TIME` / `PENALTY_SHOOTOUT` / `SUSPENDED` / `UNKNOWN_LIVE` | `正在直播` |

半场安全规则：

- `PAUSED` / `HT` / `Half Time` / `Halftime` / `中场` => `HALFTIME`
- `HALFTIME` 继续轮询，永不判 final，永不 cleanup
- 如果 web summary 同时出现 `HT` 与 `Final` / `FT`，优先 `HALFTIME`，除非结构化 API 明确返回 `FINISHED`
- `UNKNOWN_LIVE` 是直播态；宁可继续轮询，也不要过早停止

终场规则：

- 只有 `FINISHED` / `FT` / `Full-Time` / `AET` / `AP`，或强多源终场确认，才映射为 `FINISHED`
- events never decide status；goal/card/sub/commentary 等事件永不参与状态判定

```text
normalize_football_status(match):
    fields = [
        match.get("football_status", ""),
        match.get("phase", ""),
        " ".join(match.get("phase_keywords", []) or []),
        match.get("summary", ""),
        match.get("raw", ""),
    ]
    text = " ".join(fields)
    source = match.get("source", "") or "web"

    if source in {"sportmonks", "api_football", "football_data_org", "sportradar_extended"}:
        if text contains structured status in {"FINISHED", "FT", "Full-Time", "AET", "AP"}:
            return "FINISHED"
        if text contains structured status in {"SCHEDULED", "TIMED", "NS"}:
            return "SCHEDULED"

    if text contains any of {"PAUSED", "HT", "Half Time", "Halftime", "中场"}:
        return "HALFTIME"
    if text contains any of {"1H", "First Half", "上半场"}:
        return "FIRST_HALF"
    if text contains any of {"2H", "Second Half", "下半场"}:
        return "SECOND_HALF"
    if text contains any of {"ET", "Extra Time", "加时"}:
        return "EXTRA_TIME"
    if text contains any of {"PEN", "Penalty Shootout", "点球"}:
        return "PENALTY_SHOOTOUT"
    if text contains any of {"SUSPENDED", "Suspended", "中断"}:
        return "SUSPENDED"
    if text contains any of {"Live", "直播中", "进行中"} or match.get("minute"):
        return "UNKNOWN_LIVE"
    if text contains any of {"FINISHED", "FT", "Full-Time", "AET", "AP"}:
        return "FINISHED"
    if text contains any of {"未开始", "SCHEDULED"}:
        return "SCHEDULED"
    if strong_multi_source_final_confirmation(match):
        return "FINISHED"
    return "SCHEDULED"
```

#### `classify_state(matches)`

`matches` 必须先经过 `parse_search_result` 归一化（见上节）。足球优先使用 §football_status_machine；非足球先看 live-like 关键词，再看 final，最后才是未开始，避免 `HT` / `中场` 被 `FT` / `Final` 噪声误杀：

```text
football_statuses = [
    normalize_football_status(match)
    for match in matches
    if match.get("football_status") or is_football_match(match)
]
if football_statuses:
    if any status in {"FIRST_HALF", "HALFTIME", "SECOND_HALF", "EXTRA_TIME",
                      "PENALTY_SHOOTOUT", "SUSPENDED", "UNKNOWN_LIVE"}:
        return "正在直播"
    if all status == "FINISHED":
        return "已完结"
    if all status == "SCHEDULED":
        return "未开始"
    return "正在直播"

if any match contains keyword in {"Live", "直播中", "Q1", "Q2", "Q3", "Q4", "OT",
                                  "1H", "2H", "上半场", "下半场", "加时",
                                  "进行中", "中场", "HT", "Half Time", "Halftime"}:
    return "正在直播"
if any match contains keyword in {"Final", "FT", "Full-Time", "完场", "终场", "已结束"}:
    return "已完结"
return "未开始"
```

歧义时**优先视为正在直播**（误轮询比误退出代价更低）。

> **⚠ 非回归约束（NON-REGRESSION，CRITICAL）**：events MUST NOT influence state classification — state is determined SOLELY by `phase_keywords` / `phase`. 新增的 `events` 字段仅供 rendering 与 dedup 使用，**不参与**状态判定。即「事件不影响状态」：哪怕一场比赛抓到再多 goal/foul/sub 事件，只要 `phase_keywords` 没命中 Live/Final 关键词，`classify_state` 就**绝不**因 events 改变返回值。这条规则确保事件流是 Stage A 状态判定的下游旁路、而不是新数据源（详见 §extract_score_snapshot 排除列表与 §Hard constraints / break-risks）。

#### `read_state_file(path)`

```text
if file not exists: return None
try:
    return json.parse(read(path))
except (parse error / IO error):
    log_warning("state_file 损坏，按 None 处理（将在下一次成功推送时重写）")
    return None
```

#### `write_state_file_safely(path, data)`

先写入 `path + ".tmp"`，再原子重命名为 `path`，避免半写损坏：

- Unix: `mv path.tmp path`
- PowerShell: `Move-Item -Force path.tmp path`

#### state file schema —— 状态文件字段约定

`state_file` 是单文件 JSON，结构必须可被标准 `json.parse` / `json.dump` 直接序列化。本节规定文件**完整**字段集合：

```text
{
  "score":           <score_snapshot>,                 # 见 §extract_score_snapshot；用于比分去重
  "ts":              "<ISO 8601 字符串或 unix 时间戳>",  # 写入时刻，便于排查
  "seen_event_ids":  ["<id_hash>", "<id_hash>", ...],  # 已推送过的事件去重环形缓冲区
  "pbp_urls":        {"<match_key>": "<url>", ...}     # 已发现的文字直播 URL 缓存（跨轮复用）
}
```

**`seen_event_ids` 字段**：

- 类型：字符串数组，每个元素是 §parse_search_result 中定义的 `id_hash`（长度 12 的十六进制字符串）。
- 语义：**FIFO ring buffer（先进先出环形缓冲区）**——按推送时间顺序追加；超过容量上限时**从队首裁掉最早的元素**，保证数组长度始终 ≤ 上限。
- 容量上限（cap）公式：

  ```text
  cap = commentary_dedup_window × max_events_per_msg
  ```

  默认值 `30 × 6 = 180`，即默认情况下数组长度上限为 180 个 `id_hash`。
- 维护规则（每次**推送成功**后执行）：
  1. 把本轮新推送事件的 `id_hash` 依次 append 到 `seen_event_ids` 末尾。
  2. 若 `len(seen_event_ids) > cap`，从数组头部裁掉多余元素（`seen_event_ids = seen_event_ids[-cap:]`），保持 FIFO 语义。
  3. 推送失败时不更新此字段，避免「未送达事件」被错误标记为已推送（与 §sign_and_post 跳过 PERSIST 的策略一致）。
- 边界：环形缓冲区只在**单次 Skill 运行的会话期间**起作用。退出时 §cleanup_state 仍然会**整文件删除** `state_file`，所以 `seen_event_ids` 的去重是**会话内**有效的，不会污染下一次运行——这是设计如此，不是 bug。

读取兼容性：旧版本 `state_file` 没有 `seen_event_ids` 字段，`read_state_file` 必须把缺失字段当作空数组 `[]` 处理，不视为损坏。

**`pbp_urls` 字段**：

- 类型：字典，key 为 `match_key`（`home + "|" + away`），value 为文字直播页 URL 字符串。
- 语义：**跨轮复用的 URL 缓存**——一旦在某轮通过 Stage A 或 Stage A+ 发现了某场比赛的 playbyplay URL，缓存到此处，后续轮次直接使用，无需重新嗅探/搜索。
- 设计动机：每轮重新发现 URL 是造成延迟的重要原因——Stage A 搜索可能返回旧结果，Stage A+ 额外搜索增加开销。缓存后，第二轮起直接用已发现的 URL 进入 Stage B webfetch，跳过整个 URL 发现阶段。
- 维护规则：
  1. 在 §第 1.5 步中，发现 URL 后立即写入 `persisted["pbp_urls"][match_key]`。
  2. 每轮 PERSIST 步骤（推送成功后）将 `pbp_urls` 一起写盘。
  3. webfetch 失败时**不清除**缓存——可能是临时网络问题，下一轮重试同一 URL。
  4. 比赛结束（`is_finished`）时，§cleanup_state 整文件删除，pbp_urls 自然清空。
- 边界：同 `seen_event_ids`，仅在单次 Skill 运行会话期间有效。
- 读取兼容性：旧版本 `state_file` 没有 `pbp_urls` 字段，`read_state_file` 必须把缺失字段当作空字典 `{}` 处理。

#### `cleanup_state(path)`

退出前删除 `state_file`（无论正常退出还是失败退出），保证下次运行时是干净状态：

- Unix: `rm -f path`
- PowerShell: `Remove-Item -LiteralPath path -ErrorAction SilentlyContinue`

#### `resolve_state_path(rel_or_abs)`

- 如果 `state_file` 是绝对路径（`/...` 或 `X:\\...`） → 直接用
- 如果是相对路径 → **以 SKILL.md 所在目录为基准**解析（Agent 应该知道自己加载的 skill 文件路径；如果不知道，回退到 `config.json` 所在目录）

#### `sign_and_post(msg, config)` —— **签名 + 推送绑定在一起**

按平台**独立**处理。两个平台互相独立，一个失败不影响另一个：

```text
result = { feishu: null, dingtalk: null }

# ===== 飞书 =====
if config.feishu.webhook_url is non-empty:
    body = render_feishu_card(msg, config)        # JSON 对象
    if config.feishu.secret is non-empty:
        ts   = current_unix_seconds()                                 # 注意：秒级
        sign = base64(hmac_sha256(key=secret, data=ts + "\n" + secret))
        body["timestamp"] = str(ts)
        body["sign"]      = sign
    response = http_post(config.feishu.webhook_url, body)
    print response                                                     # 必须打印
    result.feishu = (response.code == 0)

# ===== 钉钉 =====
if config.dingtalk.webhook_url is non-empty:
    body = render_dingtalk_md(msg, config)
    url  = config.dingtalk.webhook_url
    if config.dingtalk.secret is non-empty:
        ts   = current_unix_milliseconds()                             # 注意：毫秒级
        sign = url_encode(base64(hmac_sha256(key=secret, data=ts + "\n" + secret)))
        url += "&timestamp=" + ts + "&sign=" + sign                    # 拼到 URL 上
    response = http_post(url, body)
    print response                                                     # 必须打印
    result.dingtalk = (response.errcode == 0)

return result
```

> **关键陷阱（极易踩坑，4 家 Agent 都要严格区分）：**
> - 飞书 timestamp 用**秒**，钉钉用**毫秒**——不要弄混
> - 飞书签名嵌入**请求体 JSON**，钉钉签名拼到**URL query**
> - 飞书成功响应是 `{"code":0,...}`；钉钉成功响应是 `{"errcode":0,...}`——判断字段名不一样
> - HMAC-SHA256 的 `key` 和 `data` 都用 secret——这是飞书规范，不是 bug

#### `all_configured_succeeded(result, config)`

```text
ok = True
if config.feishu.webhook_url is non-empty   and result.feishu   != True: ok = False
if config.dingtalk.webhook_url is non-empty and result.dingtalk != True: ok = False
return ok
```

#### `render_live(fresh, last_pushed_score, score_snapshot, config)` —— 去重 + events 门控

`render_live` 在比分去重的基础上叠加「事件门控」——即使比分没变，只要本轮存在**新事件**（`fresh` 中每个 match 的 `m.new_events` 至少有一条；`new_events` 由主循环 §主流程伪代码 在调用本函数前预先计算并附到 match dict 上，见 §主循环 push gate），就**仍然推送一条「比分无变化但有新事件」的简化卡片**。

在 `play_by_play` 模式下，即使比分未变且无新事件，仍然推送（不 SKIP）——这正是用户选择文字直播模式的预期：持续收到更新。

##### 3×2 状态表（branch matrix — commentary_style × score × events）

| commentary_style | 比分 (Score) | 新事件 (New Events) | 分支 | body 前缀 |
|------------------|--------------|---------------------|------|-----------|
| events_only | Changed | 任意 (≥0) | regular push | (沿用原前缀，不加无变化前缀) |
| events_only | Unchanged | ≥1 | events-only push | `📌 比分无变化 · 📋 N 条新事件` (N = 本轮新事件总数) |
| events_only | Unchanged | 0 | **SKIP** | — (返回字符串字面量 `"SKIP"`) |
| play_by_play | Changed | 任意 (≥0) | regular push | (沿用原前缀) |
| play_by_play | Unchanged | ≥1 | commentary push | `🎤 文字直播 · 📋 N 条新动态` (N = 本轮新事件总数) |
| play_by_play | Unchanged | 0 | **always-push** | `🎤 文字直播 · ⏱️ 比分暂无变化` |

> 注：本表只覆盖「正在直播」分支下的实时推送；未开始 / 已完结由 `render_preview` / `render_final` 处理，不进本函数。
> play_by_play 模式下**永不 SKIP**——即使比分未变且无新事件，仍推送当前比分/阶段作为"持续在线"信号。这正是用户选择文字直播模式的预期。

##### 分支伪代码（commentary_style-aware 决策树）

```text
def render_live(fresh, last_pushed_score, score_snapshot, config):
    msg = build_msg_from_fresh(fresh, kind="live")    # 见 §build_msg_from_fresh

    # 把每个 match.new_events（由主循环 push gate 预先填充，见 §主循环 push gate）
    # 平铺到 msg.events 数组——见 §msg schema (events 字段)
    # CRITICAL（P0-3, Oracle Round 11）：m 是 dict，必须用 m.get("new_events", [])。
    # **绝不**写 getattr(m, "new_events", [])——Python 的 getattr 检查 attribute 不是 dict key，
    # 对 dict 永远返默认 [] → events 永远渲染不出来 → 整个 events 特性静默失效（与 §主循环 2a/2c 同源 bug）。
    msg.events = []
    for m in fresh:
        for e in m.get("new_events", []):
            msg.events.append(e)
    new_events_count = len(msg.events)

    score_changed   = (last_pushed_score is None) or (not scores_equal(score_snapshot, last_pushed_score))
    has_new_events  = new_events_count > 0

    # === commentary_style-aware branching ===

    if config.get("commentary_style", "events_only") == "events_only":
        # --- Original 3-branch logic (unchanged) ---
        if score_changed:
            return msg
        if has_new_events:
            msg.body = "📌 比分无变化 · 📋 " + str(new_events_count) + " 条新事件\n" + msg.body
            return msg
        return "SKIP"

    else:  # play_by_play
        # --- New 3-branch logic: never SKIP ---
        if score_changed:
            return msg
        if has_new_events:
            msg.body = "🎤 文字直播 · 📋 " + str(new_events_count) + " 条新动态\n" + msg.body
            return msg
        # score unchanged + no new events → still push (play_by_play never skips)
        msg.body = "🎤 文字直播 · ⏱️ 比分暂无变化\n" + msg.body
        return msg
```

##### `scores_equal(a, b)` 语义

`scores_equal(a, b)` 比较所有进行中比赛的「主队-客队-比分-节/半场」四元组集合是否完全相等。任何一项变化即视为有变化。**只对比赛集合做集合比较，不要对消息文本做字符串 diff**（描述性文字会自然变化）。

##### `"SKIP"` 哨兵契约

- 当 `render_live` 判定「比分未变 且 无新事件」且 `commentary_style == "events_only"` 时，**必须**返回字面量字符串 `"SKIP"`（而不是返回 None / 抛异常 / 返回空 msg）。
- 当 `commentary_style == "play_by_play"` 时，**永不返回 `"SKIP"`**——即使比分未变且无新事件，仍返回 msg 对象（body 前缀加 `🎤 文字直播 · ⏱️ 比分暂无变化`）。
- 主循环（§主流程伪代码 第 2 步 RENDER 之后）**必须**在调用 `sign_and_post` 之前显式判定 `if msg == "SKIP": continue`，跳过 PERSIST 与 POST，直接进入下一个 sleep 切片。
- 该哨兵保证了 `events_only` 模式下「无变化无事件」时**不发空消息打扰群成员**，同时让 push gate 决策仍然集中在 `render_live`，而不是散落在主循环里。

#### `extract_score_snapshot(fresh)` —— 比分快照（去重 + 持久化用）

把 `fresh`（`parse_search_result` 返回的 match dict 列表）压成一个**稳定可比较的快照**，喂给 `scores_equal` 和 `write_state_file_safely`：

```text
# CRITICAL（P1-1, Oracle Round 12 一致）：m 是 dict（§parse_search_result 输出），
# 必须用 m["..."] / m.get("...") 访问。**绝不**写 m.home / m.away（getattr-on-dict 静默失败陷阱）。
return [
  {
    "home":  m.get("home", ""),
    "away":  m.get("away", ""),
    "score": m.get("score", ""),
    "phase": m.get("phase", ""),
  }
  for m in fresh
  # 不要包含 summary / source_url / raw / events 等会自然抖动的字段
]
```

要求：
1. 列表顺序需稳定（按 `home + away` 字典序排序），避免顺序抖动导致误判「有变化」
2. 字段集合固定（home / away / score / phase 四元组），与 §scores_equal 对齐
3. 输出必须可 JSON 序列化（直接喂 `write_state_file_safely`）

> **⚠ 排除 `events` 的硬性原因（非回归 / BREAK-risk）**：`events` 是按事件去重的、自然增长的 list，若进入 snapshot 会让每轮都看起来变化，导致 dedup 失效与 push 风暴（BREAK-risk: see §Hard constraints / break-risks）。即使一场比赛比分完全没变，下一轮新抓到一条 commentary 行也会让 events list 长度 +1、内容微变，snapshot 立刻判「有变化」→ 触发推送 → 几秒之后再触发 → 群里被刷爆。**事件去重的唯一正路**是主循环里基于 `seen_event_ids` 的 ring buffer 过滤（见 §state file schema 与 §主循环 push gate），**不是**塞进 snapshot。
>
> **再次强调（重申已有规则）**：`scores_equal` MUST NOT do string-diff on body/events; 它**只是**对所有进行中比赛的 `(home, away, score, phase)` 四元组做集合比较——任何对 `summary` / `events` / `body` 的字符串 diff 都会破坏 dedup 语义（见 §scores_equal 语义节）。

#### `render_preview(matches)` —— 赛事前瞻

```text
return build_msg_from_fresh(matches, kind="preview")   # template="orange"
```

#### `render_final(fresh, note=None)` —— 比赛结束总结

```text
msg = build_msg_from_fresh(fresh, kind="final")        # template="green"
if note is not None and not empty:
    msg.body = msg.body + "\n\n⚠️ " + note             # 例如 "用户主动停止播报"
return msg
```

#### `build_msg_from_fresh(data, kind)` —— 抽象消息组装器

将 `parse_search_result` 已归一化的 `data`（**match dict 列表**，每个含 `home` / `away` / `score` / `phase` / `phase_keywords` / `summary` 等字段，schema 见 §parse_search_result）组装为标准 `msg` 对象（`{title, body, footer, template}`）。组装时根据 `kind`（"live" / "preview" / "final"）选模板和文案。**隐式依赖 `config`**：`kind == "live"` 时需要 `config.commentary_style` 来选择标题（"实时播报" vs "文字直播"），以及 `poll_interval` 来计算 footer 更新间隔。`config` 通过模块级 / 闭包变量访问，与 `render_live` / `render_feishu_card` / `render_dingtalk_md` 同一约定（见 §伪代码约定）。具体内容参考 §消息格式规范 章节里的飞书/钉钉示例：

| `kind` | `commentary_style` | `msg.title` 示例 | `msg.template` | `msg.footer` |
|--------|-------------------|-----------------|---------------|--------------|
| `"live"` | events_only | `"🏀 NBA 实时播报"` / `"⚽ 世界杯 实时播报"` | `"blue"` | `"🤖 上班看球播报员 \| 下次更新: {ceil(poll_interval/60)}分钟后"` |
| `"live"` | play_by_play | `"🏀 NBA 文字直播"` / `"⚽ 世界杯 文字直播"` | `"blue"` | `"🤖 上班看球播报员 \| 下次更新: {ceil(poll_interval/60)}分钟后"` |
| `"preview"` | any | `"📋 比赛前瞻"` | `"orange"` | `"🤖 上班看球播报员 \| 比赛尚未开始，敬请期待"` |
| `"final"` | any | `"🏀 NBA 比赛结束"` / `"⚽ 世界杯 比赛结束"` | `"green"` | `"🤖 上班看球播报员 \| 播报结束"` |

`msg.title` 选择逻辑：当 `kind == "live"` 且 `config.get("commentary_style", "events_only") == "play_by_play"` 时，标题使用「文字直播」；否则使用「实时播报」。`msg.footer` 中的更新间隔必须从实际 `poll_interval` 值动态计算（而非硬编码「3分钟」），以适应 play_by_play 模式下可能更短的轮询间隔。

#### `render_feishu_card(msg, config)` / `render_dingtalk_md(msg, config)` —— 抽象消息 → 平台 JSON

`msg` 是一个抽象消息对象，由 `render_live` / `render_preview` / `render_final` 产出，包含字段：

| 字段 | 类型 | 含义 |
|------|------|------|
| `msg.title` | string | 卡片标题文本，如 "🏀 NBA 实时播报" |
| `msg.body` | string | 卡片正文（Markdown 或纯文本）。承载比分 + 节次/阶段摘要，**语义不变**——是字符串形式的「比分快照 + phase」概览。 |
| `msg.footer` | string | 页脚文本，如 "🤖 上班看球播报员 \| 下次更新: 2分钟后" |
| `msg.template` | string | 飞书卡片 header 颜色：`blue`（直播中）/ `green`（已结束）/ `orange`（前瞻） |
| `msg.events` | array&lt;event-item&gt; | **本轮要展示的新事件列表**，与 `msg.body` **平级 (PARALLEL)** 而非嵌套——`body` 仍是比分+phase 摘要字符串，`events` 是结构化事件数组。每个 event-item 的 shape 与 §parse_search_result 中 `events` 字段定义的 event-item **完全一致**（`id_hash` / `ts` / `team` / `type` / `text`，schema 单一真理见 §parse_search_result）。**MAY be `[]`**（空列表）——例如 `kind="preview"` / `"final"` 不带事件，或本轮 dedup 后无新事件且比分变化触发推送时。两个平台渲染器（`render_feishu_card` 与 `render_dingtalk_md`）都消费此字段。 |

**`msg.events` 字段语义补充**（即 `msg["events"]`）：

- **来源**：`render_live` 把 `fresh` 中每个 match 的 `m.new_events`（已经过主循环 push gate 的 dedup + trim，见 §主循环 push gate）平铺到 `msg.events`。
- **顺序**：保持平铺时的顺序（一般是按 match 顺序、再按事件抓取顺序），渲染器**不**重排。
- **空列表合法**：`msg.events == []` 是合法状态，渲染器必须优雅处理（飞书卡片省略事件 div，钉钉省略事件 bullet 段）——见下方 §render_feishu_card 与 §render_dingtalk_md 的具体规则。
- **不重复定义 schema**：event-item 字段（`id_hash` / `ts` / `team` / `type` / `text`）的所有约束在 §parse_search_result 已规定，本节**不**重复——避免双源真理。

##### Event 类型 → emoji 映射（single source of truth）

下表是事件类型到展示 emoji 的统一映射，**飞书卡片（`render_feishu_card`）与钉钉 Markdown（`render_dingtalk_md`）都引用本表**——只在这一处定义，避免双平台漂移。

| `event.type` | emoji | 含义 / 适用场景 |
|--------------|-------|-----------------|
| `goal` | ⚽ (足球) / 🏀 (篮球) | 进球 / 得分（含投篮命中、三分球、篮球得分行、足球点球转化等任何分数有变动的事件）。Agent 按本场比赛 `match_type` 选择：足球用 ⚽，篮球用 🏀。 |
| `foul` | 🟨 | 犯规（含技术犯规、普通犯规） |
| `card` | 🟥 | 红牌 / 重大处罚（与 foul 区分：foul = 黄牌或以下，card = 红牌级别） |
| `sub` | 🔄 | 换人 |
| `key_play` | ⭐ | 关键回合（关键三分、绝杀、扑点等） |
| `other` | • | 其他杂项事件，使用通用项目符号 |
| `shot` | 🎯 | 射门 / 投篮未中（play_by_play only；球权未转化为得分） |
| `save` | 🧤 | 扑救 / 封盖（play_by_play only；守门员扑出或篮球盖帽） |
| `corner` | 📐 | 角球（play_by_play only） |
| `free_kick` | 🦶 | 任意球 / 罚球未中（play_by_play only） |
| `timeout` | ⏸️ | 暂停（play_by_play only；篮球暂停 / 足球医疗暂停等） |
| `turnover` | ❌ | 失误 / 丢球权（play_by_play only；传球失误、被抢断等） |
| `rebound` | 🔁 | 篮板球（play_by_play only） |
| `commentary` | 📝 | 文字解说叙述行（play_by_play only；兜底类型——当无具体技术事件时描述比赛节奏/局势） |

> 实现注：`goal` 行展示时由渲染器根据 `match_type`（或 `match.home` / `match.away` 联赛上下文）二选一；其余 emoji 跨赛事一致。`match_type` 是主循环启动时由用户选择（"世界杯" / "NBA"）的变量（见 §主流程伪代码 `match_type = ask_user_if_missing(...)`），不在 `parse_search_result` 输出的 match dict 中。渲染器通过与 `config` 同样的方式访问 `match_type`（模块级 / 闭包变量；`render_feishu_card(msg, config)` / `render_dingtalk_md(msg, config)` 所在作用域应能访问 `match_type`）。若无法访问 `match_type`，可通过队名上下文推断（含 "Lakers"/"Celtics" 等 NBA 队名 → 篮球 🏀；其余 → 足球 ⚽）。标注 "play_by_play only" 的 8 个类型在 `events_only` 模式下不会出现（验证阶段会映射到 `other`），渲染器不需要为 `events_only` 模式处理这些 emoji。`commentary` 类型在 `play_by_play` 模式下是**兜底类型**——当无具体技术事件时，LLM 仍会产出一行 `commentary` 描述当前比赛节奏/局势，保证每次轮询都有推送内容。

**`render_feishu_card(msg, config)`** → 按 §飞书消息格式 组装为「N 元素」卡片（N ∈ {3, 4}）：

```text
{
  "msg_type": "interactive",
  "card": {
    "header": { "title": { "tag": "plain_text", "content": msg.title }, "template": msg.template },
    // P2-6 (Oracle Round 10)：elements 数组**必须**按下方伪代码动态拼接，不能照搬下面的 4-元素 JSON 模板
    // 当 msg.events 为空。否则会渲染出空标题「📋 **实时事件**」无 bullet —— 视觉 bug。
    "elements": <DYNAMICALLY BUILT, see pseudo-code below>
  }
}

# 元素拼接伪代码（agent 必须按此实现）：
elements = []
elements.append({ "tag": "div", "text": { "tag": "lark_md", "content": msg.body } })  # Element 1: 比分（始终存在）
if len(msg.events) > 0:
    if config.get("commentary_style", "events_only") == "play_by_play":
        # play_by_play: narrative header + bullets with type-emoji
        # Separate commentary-type events (narrative lines) from action events
        commentary_lines = [e for e in msg.events if e.get("type") == "commentary"]
        action_lines     = [e for e in msg.events if e.get("type") != "commentary"]
        bullet_parts = []
        if commentary_lines:
            # Narrative lines go first, rendered with 📝 marker
            for e in commentary_lines:
                bullet_parts.append("- 📝 " + e["text"])
        for e in action_lines:
            bullet_parts.append("- " + e["ts"] + " " + emoji_for(e) + " " + e["text"])
        elements.append({ "tag": "div", "text": { "tag": "lark_md",
            "content": "🎤 **文字直播**\n" + "\n".join(bullet_parts)
        } })                                                                              # Element 2: 文字直播（play_by_play）
    else:
        # events_only: original format
        bullet = "\n".join("- " + e["ts"] + " " + emoji_for(e) + " " + e["text"]  for e in msg.events)
        elements.append({ "tag": "div", "text": { "tag": "lark_md",
            "content": "📋 **实时事件**\n" + bullet
        } })                                                                              # Element 2: 事件（events_only）
elements.append({ "tag": "hr" })                                                       # 分隔线
elements.append({ "tag": "note", "elements": [{
    "tag": "plain_text",
    "content": msg.footer    # e.g. "🤖 上班看球播报员 | 下次更新: 1.5分钟后" —— 与 §msg schema (msg.footer) byte-equal
}] })                                                                                  # 注脚

# 等价 JSON 模板（仅 msg.events 非空时整 4 元素，msg.events 空时退化为 3 元素）：
# events_only 模式：
{
  "elements": [
    { "tag": "div", "text": { "tag": "lark_md", "content": msg.body } },
    { "tag": "div", "text": { "tag": "lark_md", "content":
        "📋 **实时事件**\n"
        + "\n".join("- " + e.ts + " " + emoji_for(e) + " " + e.text  for e in msg.events)
    } },
    { "tag": "hr" },
    { "tag": "note", "elements": [{ "tag": "plain_text", "content": msg.footer }] }
  ]
}
# play_by_play 模式：
{
  "elements": [
    { "tag": "div", "text": { "tag": "lark_md", "content": msg.body } },
    { "tag": "div", "text": { "tag": "lark_md", "content":
        "🎤 **文字直播**\n"
        + (narrative_lines: "- 📝 " + e.text for commentary-type events)
        + (action_lines: "- " + e.ts + " " + emoji_for(e) + " " + e.text for non-commentary events)
    } },
    { "tag": "hr" },
    { "tag": "note", "elements": [{ "tag": "plain_text", "content": msg.footer }] }
  ]
}
```

`emoji_for(e)` 严格遵循上方 §Event 类型 → emoji 映射 表：根据 `e.type` 取对应 emoji；`goal` 类型再按比赛是足球还是篮球二选一。

**编排规则**：

1. 当 `msg.events == []` → **省略 Element 2**，elements 数组只剩 3 项（div / hr / note），与历史卡片**完全一致（向后兼容）**。
2. 当 `msg.events` 非空 → 在 Element 1 之后、Element 3（`hr`）之前插入事件 div，elements 数组为 4 项。
3. 事件 div 的 `lark_md` 正文以 `📋 **实时事件**\n` 起头，随后每条事件占一行 bullet：`- {ts} {emoji} {text}`（`ts` 为空字符串时 bullet 仍保留两个空格防止排版崩裂，由渲染器实现兜底）。
4. 事件展示顺序与 `msg.events` 数组顺序一致——`render_live` 已保证按 match → event 抓取序铺平，渲染器不重排。

如果 `secret` 非空，由 `sign_and_post` 在该 JSON 顶层追加 `timestamp` 和 `sign` 字段。

**`render_dingtalk_md(msg, config)`** → 按 §钉钉消息格式 组装为：

```text
{
  "msgtype": "markdown",
  "markdown": {
    "title": msg.title,
    "text": "## " + msg.title + "\n\n"
            + msg.body
            + (when msg.events is non-empty:
                  (config.get("commentary_style", "events_only") == "play_by_play" ?
                      "\n\n🎤 **文字直播**\n"
                      + commentary_first_format(msg.events)
                    : "\n\n📋 **实时事件**\n"
                      + "\n".join("- " + e.ts + " " + emoji_for(e) + " " + e.text  for e in msg.events))
              else: "")
            + "\n\n---\n\n> " + msg.footer
  }
}
```

其中 `commentary_first_format(events)` 将 `commentary` 类型事件排在最前（用 📝 标记），其余事件按原格式：
```text
def commentary_first_format(events):
    commentary_lines = [e for e in events if e.get("type") == "commentary"]
    action_lines     = [e for e in events if e.get("type") != "commentary"]
    parts = []
    for e in commentary_lines:
        parts.append("- 📝 " + e["text"])
    for e in action_lines:
        parts.append("- " + e["ts"] + " " + emoji_for(e) + " " + e["text"])
    return "\n".join(parts)
```

拼接顺序固定为 `title → body → events 段（可选）→ 分隔线 + footer`。`emoji_for(e)` 仍然引用上方 §Event 类型 → emoji 映射 表（与飞书卡片**同一份映射，不要复制粘贴定义**）。当 `msg.events == []` 时整段事件块省略，`text` 退化为「title + body + footer」三段，与历史 markdown 输出**完全一致**。

##### 钉钉 18000 字节 size guard（防超长截断）

钉钉 markdown `text` 字段有约 ~20KB 的硬上限（实测 server 侧裁剪点不稳定，约在 19~20KB 之间），超长会被服务端拒收并返回 `errcode != 0`。`render_dingtalk_md` 在返回前**必须**做尺寸守卫：

> **play_by_play 模式注意事项**：`play_by_play` 模式事件更多（推荐 `max_events_per_msg=8-12`），size guard 更容易触发，必须确保截断逻辑在两种模式下都正确运行。算法不变，只是可能需要 pop 更多条目。

- **阈值**：渲染后 `len(rendered_text) <= 18000`（按 UTF-8 字节数；保守留 2KB 安全边际给签名 / URL 编码 / 钉钉服务端额外裁剪）。
- **超额时的截断算法**（迭代丢弃**最旧**事件）：

  ```text
  def render_dingtalk_md(msg, config):
      events_local = list(msg.events)   # 工作副本，不修改 msg.events 本身

      def assemble(events_to_render, truncated_count):
          ev_block = ""
          if len(events_to_render) > 0:
              if config.get("commentary_style", "events_only") == "play_by_play":
                  # play_by_play: commentary-first format with 🎤 header
                  commentary_lines = [e for e in events_to_render if e.get("type") == "commentary"]
                  action_lines     = [e for e in events_to_render if e.get("type") != "commentary"]
                  parts = []
                  for e in commentary_lines:
                      parts.append("- 📝 " + e["text"])
                  for e in action_lines:
                      parts.append("- " + e["ts"] + " " + emoji_for(e) + " " + e["text"])
                  ev_block = "\n\n🎤 **文字直播**\n" + "\n".join(parts)
              else:
                  # events_only: original format
                  ev_block = "\n\n📋 **实时事件**\n" + "\n".join(
                      "- " + e.ts + " " + emoji_for(e) + " " + e.text
                      for e in events_to_render
                  )
          if truncated_count > 0:
              ev_block += "\n\n…(剩余 " + str(truncated_count) + " 条事件已截断 / " \
                       + str(truncated_count) + " more events truncated)"
          return ("## " + msg.title + "\n\n" + msg.body
                  + ev_block + "\n\n---\n\n> " + msg.footer)

      truncated = 0
      text = assemble(events_local, truncated)

      # a. 迭代丢队首（最旧）事件直到落入阈值
      # b. 每次 pop 后立刻重组并附上「事件已截断」标记
      # c. 重新检查 size，仍超则继续 pop
      while len(text.encode("utf-8")) > 18000 and len(events_local) > 0:
          events_local.pop(0)            # 丢队首 = 最旧
          truncated += 1
          text = assemble(events_local, truncated)

      # d. 终极兜底：连「title + body + footer」（events 全空）都仍然超 18000，
      #    那是 pre-existing 行为（body 本身就超长），本函数不再额外截 body——
      #    直接返回 events 全空的版本，让上游沿用原有超长容错（与 commits #1/#2 之前一致）。
      if len(text.encode("utf-8")) > 18000:
          text = ("## " + msg.title + "\n\n" + msg.body
                  + "\n\n---\n\n> " + msg.footer)

      return { "msgtype": "markdown", "markdown": { "title": msg.title, "text": text } }
  ```

- **截断标记文案**（用户可感知）：固定为 `"\n\n…(剩余 N 条事件已截断 / N more events truncated)"`——中英双语单行文案，`N` 为本次被丢弃的旧事件总数。
- **丢弃顺序**：始终从**队首**（最旧）pop——这与 §state file schema `seen_event_ids` 的 FIFO 语义、以及 §主循环 push gate 的「保留最近 N 条」策略完全一致。

钉钉的签名是拼接到 URL query 上，不修改请求体。

#### `notify_stop_hint_once()` / `user_said_stop()` / 优雅终止

⚠️ **物理限制**：在 `Start-Sleep -Seconds 180` / `sleep 180` 前台阻塞期间，Agent **无法**接收用户输入。所以「立刻终止」物理上不可能。

实际行为：

1. **首次启动**: Agent 必须给用户发一句提示——「您可以随时发送『停止 / 结束 / stop』终止播报。由于本 Skill 用前台 sleep 等待 2 分钟（内部按 60 秒切片），您发送停止后**最长等待一个 sleep 切片（约 60 秒）**才会真正退出，这是设计如此。」
2. **每轮 SLEEP 之前** Agent 检查一次最新对话历史里是否含「停止 / 结束 / stop」字样
3. **每个 60 秒 sleep 切片之间** Agent 也再检查一次（见 §chunked sleep），从而把停止响应延迟从 120 秒压到 ≤60 秒
4. **如果发现** → 立即发送最终赛果并退出，**不进入下一个切片或下一轮 sleep**
5. **如果当前已经在 sleep 切片中** → 用户的「停止」要等当前切片自然结束、控制权回到 Agent 之后才会被读到，这是预期行为

### 跨平台 sleep 实现

`sleep_in_foreground(N)` 是所有 4 家 Agent 必须支持的同步阻塞调用。**禁止**用任何异步定时器实现。Agent 应该按以下顺序选择：

> ⚠ **重要：下表只列出底层 sleep 命令本身（按 60 秒切片，单次调用一段）。2 分钟（120s）= 调用 2 次 60 秒。绝不直接用 `Start-Sleep -Seconds 120` / `sleep 120` 单次跑满 2 分钟（会超过 shell 工具默认 timeout 触发杀进程）。详见下方 §宿主 shell 工具 timeout 兜底（chunked sleep）。**

| 环境 | 单段命令（60s） | 通过哪个 shell 工具调用 |
|------|----------------|----------------------|
| **Windows PowerShell 5.1** (opencode/Cursor on Windows 默认) | `Start-Sleep -Seconds 60` × 2 次 | `bash` / `Bash` / `shell` |
| **Windows + Git Bash / WSL** | `sleep 60` × 2 次 | `bash` / `Bash` / `shell` |
| **macOS / Linux** | `sleep 60` × 2 次 | `bash` / `Bash` / `shell` |
| **跨平台兜底（POSIX 不可用时）** | `python -c "import time; time.sleep(60)"` × 2 次 | shell 工具 |

**检测顺序**（伪代码 + 具体探测命令）：

```text
# Step 1: 探测是否在 PowerShell 里
#   - 在 shell 工具里跑: $PSVersionTable.PSVersion.Major
#   - 如果返回数字 (5 或更高) → PowerShell
#   - 如果返回错误 / 空 → 不是 PowerShell
if running_in_powershell():
    run("Start-Sleep -Seconds {N}")        # 必须前台执行

# Step 2: 否则探测 POSIX sleep
#   - 在 shell 工具里跑: command -v sleep
elif command_exists("sleep"):
    run("sleep {N}")

# Step 3: 兜底
else:
    run("python -c \"import time; time.sleep({N})\"")
```

**绝对不许**用 PowerShell 的 `Start-Job`、bash 的 `sleep N &`、Node.js 的 `setTimeout`、Python 的 `asyncio.sleep` 调度到后台 —— 这些都违反 §CRITICAL RULE #4。

#### ⚠ 宿主 shell 工具 timeout 兜底（chunked sleep）

**重要事实**：很多 Agent 的 shell/bash 工具有内置 timeout 上限（例如 OpenCode 的 `bash` 工具默认 120 秒、最长 600 秒；某些 Agent 单次命令上限只有 60s/90s）。一次性 `sleep 180` 会被宿主**强行 kill**，导致轮询节奏错乱、状态机错位。

**所以 `sleep_in_foreground(N)` 必须切片成不超过 60 秒的小段**，每段在 shell 工具里独立跑一次（不是一条命令里 `sleep 60; sleep 60; sleep 60` —— 那仍是 180s 单调用），而是**调用 shell 工具 3 次，每次只 sleep 60 秒**：

```text
def sleep_in_foreground(total_seconds):
    chunk = 60                              # 安全阈值，远低于任何 Agent 的 shell timeout
    remaining = total_seconds
    while remaining > 0:
        this_round = min(chunk, remaining)
        if running_in_powershell():
            run("Start-Sleep -Seconds {this_round}")     # 一次 shell 工具调用
        elif command_exists("sleep"):
            run("sleep {this_round}")                    # 一次 shell 工具调用
        else:
            run("python -c \"import time; time.sleep({this_round})\"")
        remaining -= this_round
        # 注意：每个 chunk 之间检查一次「停止」字样（CRITICAL RULE #5），
        # 这样用户说停止后最长 60 秒内就能响应，而不是非得等满 120 秒
```

这样做有 3 个好处：
1. **不被 shell 工具 timeout kill** —— 每段 60s 远低于任何 Agent 上限
2. **停止响应更快** —— 用户说「停止」最长等 60s（chunk 间检查），不再是 120s
3. **跨 Agent 一致** —— codex/cursor/opencode/claude code 都按同一节奏切片

**仍然禁止**：把 chunk 改成 `Start-Job` / `&` / `setTimeout` 异步触发 —— 切片必须**串行同步**，每个 chunk 必须等上一个 chunk 完整返回后才开始下一个。

### 跨平台 HTTP 底层命令参考（供 `sign_and_post` 内部使用）

`sign_and_post` 在构造好请求体（飞书）/ URL（钉钉）后，根据当前 shell 选择以下命令发送 HTTP POST。**两条都必须打印 HTTP 响应体**，由 `sign_and_post` 解析 `code == 0`（飞书）/ `errcode == 0`（钉钉）来填充 `result.feishu` / `result.dingtalk`。

**Unix / macOS / WSL / Git Bash**:

```bash
RESPONSE=$(curl -sS -X POST -H "Content-Type: application/json" \
    --data-binary @payload.json \
    "${WEBHOOK_URL}")
echo "[webhook response] $RESPONSE"   # 必须打印
```

**Windows PowerShell 5.1**:

```powershell
$body     = Get-Content -LiteralPath payload.json -Raw -Encoding UTF8
$response = Invoke-RestMethod -Uri $WEBHOOK_URL -Method Post `
    -ContentType 'application/json; charset=utf-8' `
    -Body ([System.Text.Encoding]::UTF8.GetBytes($body))
Write-Output ("[webhook response] " + ($response | ConvertTo-Json -Compress))
```

**两条都必须打印 HTTP 响应体**（飞书成功 `{"code":0,...}`；钉钉成功 `{"errcode":0,...}`）。响应非成功 → 计入 `fail_count`。

### 比赛状态判断规则

| 状态 | 触发条件 | 处理 |
|------|---------|------|
| **正在直播 (LIVE)** | 搜索结果含「直播中 / Live / Q1-Q4 / 1H-2H / 上半场-下半场」字样且无「Final / 完场 / FT」 | 进入轮询 |
| **已完结 (FT/Final)** | 搜索结果含「Final / 完场 / FT / Full-Time / 最终」 | 一次性总结后退出 |
| **未开始 (Scheduled)** | 搜索结果只含未来开赛时间，无任何「Live」字样 | 发送前瞻后退出 |

歧义时**优先视为正在直播**（误轮询比误退出代价更低）。

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
            "content": "🤖 上班看球播报员 | 下次更新: 2分钟后"
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

**消息体示例 — 实时播报（含事件列表，4 元素布局）**:

当 `include_events=true` 且本轮成功抓到事件时，飞书卡片采用 **4 元素布局**（`div-score` / `div-events` / `hr` / `note`）。`div-events` 的 `lark_md` 正文以 `📋 **实时事件**\n` 起头，每条事件占一行 bullet。`emoji_for(e)` 严格遵循 §Event 类型 → emoji 映射 表（合法 type 仅有 6 个：`goal`/⚽ 或 🏀、`foul`/🟨、`card`/🟥、`sub`/🔄、`key_play`/⭐、`other`/•）。**绝不**虚构 `yellow_card` / `red_card` / `score` 等枚举外类型——红黄牌统一用 `card` + text 描述区分（如 `text="梅西吃到本场第二张黄牌被罚下"`）；篮球得分（投篮命中 / 三分 / 篮球得分行）统一用 `goal`，**不要**用 `score`（已从枚举里移除）。

```json
{
  "timestamp": "1700003600",
  "sign": "yyyyyyyyyyyyy",
  "msg_type": "interactive",
  "card": {
    "header": {
      "title": { "tag": "plain_text", "content": "⚽ 世界杯 实时播报" },
      "template": "blue"
    },
    "elements": [
      {
        "tag": "div",
        "text": {
          "tag": "lark_md",
          "content": "**阿根廷 vs 法国**\n🕐 下半场 73'\n🏟️ Lusail Stadium\n\n📊 **当前比分**: 阿根廷 2 - 1 法国"
        }
      },
      {
        "tag": "div",
        "text": {
          "tag": "lark_md",
          "content": "📋 **实时事件**\n- 71' ⚽ 梅西禁区右肋抽射破门，阿根廷再下一城\n- 68' 🔄 法国换人：吉鲁下，科洛·穆阿尼上\n- 64' 🟨 德保罗战术犯规吃到本场首张黄牌\n- 59' ⭐ 姆巴佩单刀被大马丁神扑没收"
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
            "content": "🤖 上班看球播报员 | 下次更新: 2分钟后"
          }
        ]
      }
    ]
  }
}
```

> 当 `msg.events == []`（Stage B 未抓到任何事件）时，**省略 `div-events` 这一项**，整张卡片退化为前面的 3 元素布局，与历史输出完全一致。

**无签名时**: 省略 `timestamp` 和 `sign` 字段，直接发送 `msg_type` 和 `card`。

**消息体示例 — 文字直播模式（play_by_play，4 元素布局）**:

当 `commentary_style="play_by_play"` 时，飞书卡片结构与 events_only 相同（4 元素），但：
- Element 2 标题改为 `🎤 **文字直播**`（而非 `📋 **实时事件**`）
- `commentary` 类型事件排在最前，用 📝 标记，不显示时钟前缀
- 其余事件仍按 `ts + emoji + text` 格式

```json
{
  "timestamp": "1700005400",
  "sign": "zzzzzzzzzzzzz",
  "msg_type": "interactive",
  "card": {
    "header": {
      "title": { "tag": "plain_text", "content": "⚽ 世界杯 文字直播" },
      "template": "blue"
    },
    "elements": [
      {
        "tag": "div",
        "text": {
          "tag": "lark_md",
          "content": "**阿根廷 vs 法国**\n🕐 下半场 73'\n\n📊 **当前比分**: 阿根廷 2 - 1 法国"
        }
      },
      {
        "tag": "div",
        "text": {
          "tag": "lark_md",
          "content": "🎤 **文字直播**\n- 📝 双方中场争夺激烈，法国控球率略占优势\n- 71' ⚽ 梅西禁区右肋抽射破门，阿根廷再下一城\n- 68' 🔄 法国换人：吉鲁下，科洛·穆阿尼上\n- 65' 🎯 姆巴佩远射偏出\n- 64' 🟨 德保罗战术犯规\n- 59' 🧤 大马丁扑出姆巴佩单刀\n- 57' 📐 法国获得右侧角球"
        }
      },
      { "tag": "hr" },
      {
        "tag": "note",
        "elements": [
          { "tag": "plain_text", "content": "🤖 上班看球播报员 | 下次更新: 1.5分钟后" }
        ]
      }
    ]
  }
}
```

**比分未变、有 commentary 事件时的文字直播推送**:

```json
{
  "msg_type": "interactive",
  "card": {
    "header": {
      "title": { "tag": "plain_text", "content": "⚽ 世界杯 文字直播" },
      "template": "blue"
    },
    "elements": [
      {
        "tag": "div",
        "text": {
          "tag": "lark_md",
          "content": "🎤 文字直播 · 📋 1 条新动态\n\n**阿根廷 vs 法国**\n🕐 下半场 80'\n\n📊 **当前比分**: 阿根廷 2 - 1 法国"
        }
      },
      {
        "tag": "div",
        "text": {
          "tag": "lark_md",
          "content": "🎤 **文字直播**\n- 📝 比赛进入最后十分钟，阿根廷收缩防守，法国持续施压但暂无威胁射门"
        }
      },
      { "tag": "hr" },
      {
        "tag": "note",
        "elements": [
          { "tag": "plain_text", "content": "🤖 上班看球播报员 | 下次更新: 1.5分钟后" }
        ]
      }
    ]
  }
}
```

**比分未变、无新事件时的文字直播推送**（3 元素布局——事件 div 省略）:

当 `commentary_style="play_by_play"` 且本轮无新事件（LLM 也未返回任何事件）时，飞书卡片退化为 3 元素（比分 div + hr + note），body 前缀为 `🎤 文字直播 · ⏱️ 比分暂无变化`。

```json
{
  "msg_type": "interactive",
  "card": {
    "header": {
      "title": { "tag": "plain_text", "content": "⚽ 世界杯 文字直播" },
      "template": "blue"
    },
    "elements": [
      {
        "tag": "div",
        "text": {
          "tag": "lark_md",
          "content": "🎤 文字直播 · ⏱️ 比分暂无变化\n\n**阿根廷 vs 法国**\n🕐 下半场 85'\n\n📊 **当前比分**: 阿根廷 2 - 1 法国"
        }
      },
      { "tag": "hr" },
      {
        "tag": "note",
        "elements": [
          { "tag": "plain_text", "content": "🤖 上班看球播报员 | 下次更新: 1.5分钟后" }
        ]
      }
    ]
  }
}
```

> 注意：此场景下 `msg.events == []`，事件 div 被省略，与 `events_only` 模式的 3 元素退化布局一致（见 §编排规则第 1 条）。

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
    "text": "## 🏀 NBA 实时播报\n\n**洛杉矶湖人 vs 金州勇士**\n\n- 🕐 第一节 8:32\n- 🏟️ Crypto.com Arena\n\n---\n\n### 📊 当前比分\n\n**湖人 18 - 15 勇士**\n\n---\n\n### 📈 节间数据\n\n**湖人**: 命中率 45%, 篮板 8, 助攻 4\n\n**勇士**: 命中率 40%, 篮板 6, 助攻 3\n\n---\n\n> 🤖 上班看球播报员 | 下次更新: 2分钟后"
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

**消息体示例 — 实时播报（含事件列表）**:

当 `include_events=true` 且本轮成功抓到事件时，钉钉 markdown `text` 字段会在 body（比分 / 阶段）后追加 **`📋 **实时事件**`** 段，每条事件占一行 bullet（最多 `max_events_per_msg` 条）。emoji 引用 §Event 类型 → emoji 映射 表，与飞书卡片同一份映射。

```json
{
  "msgtype": "markdown",
  "markdown": {
    "title": "⚽ 世界杯 实时播报",
    "text": "## ⚽ 世界杯 实时播报\n\n**阿根廷 vs 法国**\n\n- 🕐 下半场 73'\n- 🏟️ Lusail Stadium\n\n---\n\n### 📊 当前比分\n\n**阿根廷 2 - 1 法国**\n\n---\n\n📋 **实时事件**\n\n- 71' ⚽ 梅西禁区右肋抽射破门，阿根廷再下一城\n- 68' 🔄 法国换人：吉鲁下，科洛·穆阿尼上\n- 64' 🟨 德保罗战术犯规吃到本场首张黄牌\n- 59' ⭐ 姆巴佩单刀被大马丁神扑没收\n\n---\n\n> 🤖 上班看球播报员 | 下次更新: 2分钟后"
  }
}
```

> 当 `msg.events == []` 时，整段「📋 **实时事件**」或「🎤 **文字直播**」省略，`text` 退化为「title + body + footer」三段，与上方第一个示例的结构一致。渲染前还要走 §钉钉 18000 字节 size guard：超过 18000 字节按 FIFO 丢队首事件，必要时附「已截断 N 条」标记。

**消息体示例 — 文字直播模式（play_by_play）**:

当 `commentary_style="play_by_play"` 时，钉钉 markdown 格式与 events_only 类似，但事件段标题改为 `🎤 **文字直播**`，且 `commentary` 类型事件用 📝 标记排在前面。

```json
{
  "msgtype": "markdown",
  "markdown": {
    "title": "⚽ 世界杯 文字直播",
    "text": "## ⚽ 世界杯 文字直播\n\n🎤 文字直播 · 📋 7 条新动态\n\n**阿根廷 vs 法国**\n\n- 🕐 下半场 73'\n\n---\n\n### 📊 当前比分\n\n**阿根廷 2 - 1 法国**\n\n---\n\n🎤 **文字直播**\n\n- 📝 双方中场争夺激烈，法国控球率略占优势\n- 71' ⚽ 梅西禁区右肋抽射破门，阿根廷再下一城\n- 68' 🔄 法国换人：吉鲁下，科洛·穆阿尼上\n- 65' 🎯 姆巴佩远射偏出\n- 64' 🟨 德保罗战术犯规\n- 59' 🧤 大马丁扑出姆巴佩单刀\n- 57' 📐 法国获得右侧角球\n\n---\n\n> 🤖 上班看球播报员 | 下次更新: 1.5分钟后"
  }
}
```

---

## 重要规则（执行 checklist）

| # | 规则 | 备注 |
|---|------|------|
| 1 | **每次执行前先 read config.json** | 获取 webhook 地址 / 轮询间隔 / 失败上限 |
| 2 | **默认优先用 LLM web_search** | 不要依赖未配置的固定 API、不要假设 API Key。仅足球场景可按 `providers.football.priority` 尝试已启用且有 key 的可选 provider；缺失、关闭或失败时 provider fallback 到 `web_search` / `webfetch`。 |
| 3 | **三态分流** | 已完结 → 一次性总结；未开始 → 一次性前瞻；正在直播 → 进入轮询 |
| 4 | **比分无变化时的推送策略** | `events_only`：比分无变化但有新事件时推送（加注「比分无变化」）；无新事件则跳过。`play_by_play`：每轮必推，即使比分未变也有叙述更新。 |
| 5 | **错误重试 + 上限** | 单步失败立即重试一次；连续 `max_consecutive_failures` 次彻底失败则停止并通知用户 |
| 6 | **优雅终止** | 用户说「停止 / 结束 / stop」时，Agent 在**当前 sleep 切片结束、控制权回到自己后**立刻发送最终赛果并退出（**不进入下一个切片或下一轮 sleep**）。首次启动时必须告知用户：发停止后最长等待一个 sleep 切片（约 60 秒）才会真正退出 —— 这是前台 sleep 切片的物理限制，详见 §优雅终止 + §chunked sleep。 |
| 7 | **多场比赛汇总** | 同一时间多场比赛**合并到一条消息**里发送，避免刷屏 |
| 8 | **必须前台 sleep** | `Start-Sleep -Seconds N`（PowerShell）或 `sleep N`（Unix），**禁止任何形式的后台化** |
| 9 | **不创建调度脚本** | 不要 `write` 任何 `.py` / `.sh` / `.ps1` 来跑循环；循环必须由 Agent 自身在对话里跑 |
| 10 | **语言** | 所有播报内容使用 `config.language`（默认 zh-CN） |
| 11 | **commentary_style 决定推送频率** | `events_only` = 仅在比分变化或出现新事件时推送；`play_by_play` = 每轮必推，即使比分未变。用户选择文字直播模式时期望的就是持续更新。 |
| 12 | **play_by_play 的 commentary 兜底** | `play_by_play` 模式下，如果本轮无具体技术事件，LLM **必须**产至少 1 条 `type="commentary"` 事件描述当前局势。这保证每轮推送都有内容，不会发空消息。 |
| 13 | **Stage A+ Fallback 提升实时性** | 当 Stage A 主搜索没找到文字直播 URL 时，若 `search_freshness_boost=true`（默认），Agent **必须**用 `build_pbp_search_query` 发起 site: 限定搜索精准定位中文实时源（虎扑/新浪），而非直接 `events=[]` 静默回退。这是解决"进球后长时间检测不到"的核心机制。每场每轮最多 1 次额外搜索，搜索失败时优雅回退 `events=[]`。 |

---

## 使用示例

```
用户: 帮我看下现在有什么NBA比赛
Agent: [read config.json] → [web_search "NBA 比分 直播 NBA live scores today"]
       → 检测到 2 场进行中
       → [render_live + sign_and_post] (一条消息汇总两场)
       → [Start-Sleep -Seconds 180] ← Windows PowerShell 同步阻塞
       → 醒来 → [web_search] → [Stage A+ fallback if needed] → [post] → [sleep] → ... 循环

用户: 世界杯有比赛吗
Agent: [read config.json] → [web_search "世界杯 比分 直播 FIFA World Cup live scores today"]
       → 检测到 1 场未开始
       → [render_preview + sign_and_post]
       → 退出（不进入轮询）

用户: 停止播报
Agent: [render_final + sign_and_post] → 退出循环
```

---

## 平台兼容性说明

本 Skill 是一个纯 Markdown 指令文件，不依赖任何特定平台的 API 或 SDK。Agent 在执行时：

- 使用平台自带的 **web_search / webfetch** 工具获取比赛数据（参见 §各宿主 Agent 工具映射表）
- 使用平台自带的 **shell** 工具同步阻塞 sleep + 调用 curl / Invoke-RestMethod
- 使用平台自带的 **文件读写** 工具读写 config.json + state_file

**已测试兼容的平台**:
- opencode
- Claude Code (claude)
- Codex (openai/codex)
- Cursor (cursor-agent)
- OpenClaw
- Trae

> **再次强调**：本 Skill **不会**让 Agent 后台化。轮询期间该 Agent 对话本身是阻塞的，但用户的 codex/cursor/opencode/claude code 应用本身（在新对话/新窗口）依然可以正常工作。如果你需要 Agent 在轮询的同时干别的活，请新开一个 Agent 对话/实例，**不要**修改本 Skill 让它后台化。

---

## 边界情况 / FAQ（commentary 相关）

下列三种情况是 Stage A / Stage B / 推送决策必须**显式优雅处理**的边界，不是 bug，是设计如此：

**Q1: Stage A 嗅探不到 play-by-play URL 怎么办？**

A: **Stage A+ Fallback 会自动补搜**（当 `search_freshness_boost=true` 时，这是默认值）。完整流程：

1. Stage A 主搜索（`build_query`）→ `pick_playbyplay_url_from_stage_a` 返回 None
2. **Stage A+ Fallback**：用 `build_pbp_search_query` 构建 site: 限定搜索（如 `site:nba.hupu.com/games 湖人 勇士 文字直播`），精准定位中文实时源
3. Stage A+ 也找不到 URL → `events=[]` 静默回退；推送照旧（仅比分）
4. **绝对不允许**因此跳过该轮推送——比分本身就是有用信息

Stage A+ 的约束（防止搜索预算爆炸）：
- 每场每轮**最多 1 次**额外 `web_search`
- 仅在 Stage A 主搜索没找到 URL 时触发
- 受 `search_freshness_boost` 配置控制，设为 `false` 可关闭（回到旧行为：直接 `events=[]`）
- Stage A+ 搜索本身失败时优雅回退 `events=[]`，不阻塞主循环

如果 `search_freshness_boost=false`（旧行为）：Stage A 嗅探不到 → 直接 `events=[]`，不再补搜。日志里记一行 `no play-by-play source for {home} vs {away}, skip Stage B` 即可，渲染器看到 `msg.events == []` 会自动走 3 元素飞书卡片 / 「title + body + footer」钉钉 markdown 退化路径。

**Q2: Stage B `webfetch` 在单步预算内超时 / 失败怎么办？**

A: webfetch 在 60s 切片预算内超时 → `events=[]` 优雅跳过当前 match；不阻塞主循环。同理覆盖：HTTP 5xx、页面被反爬挡掉、LLM 提取阶段返回空 / 不合法 schema、URL 临时挂掉等所有 Stage B 失败场景。**绝对不允许**因为 commentary 抓取失败就漏掉这一轮的比分推送，更不允许让单场比赛的失败拖垮整个轮询循环。本场比赛下一轮（2 分钟后）会重新尝试 Stage B。

**Q3: 比分没变，事件去重后也是空，要发吗？**

A: 取决于 `commentary_style`：

- 比分变了 → 必须推（即使没新事件）
- 比分没变但有新事件 → 必须推（事件本身是新信息）
- **比分没变 AND 没有任何新事件** → 取决于 commentary_style：
  - `events_only` → 跳过本轮推送，避免刷屏
  - `play_by_play` → **仍然推送**（文字直播模式每轮必推，见 Q4）

这是 `events_only` 模式下的唯一 skip 分支。`play_by_play` 模式无 skip 分支。判断逻辑见 §主循环 push gate 与 §render_live 3×2 状态表。

**Q4: play_by_play 模式下比分没变也没新事件，为什么要发？**

A: 这是 play_by_play 模式的设计意图。用户选择文字直播，就是想要像文字解说员一样持续收到更新——即使没有进球/犯规，"比赛进行到第80分钟，双方僵持"本身就是有价值的信息。如果不想持续推送，请使用 `commentary_style: "events_only"`。

**Q5: play_by_play 模式会不会刷屏太频繁？**

A: 有三层控制：(1) `poll_interval_seconds` 在 play_by_play 模式下自动从 180s 降为 90s（可通过 `play_by_play_default_poll_interval_seconds` 配置或手动设 `poll_interval_seconds` 自定义）；(2) `max_events_per_msg` 控制每条消息的事件数上限；(3) 钉钉 18KB size guard 自动截断过长消息。如果仍然觉得太频繁，调大 `poll_interval_seconds` 或切回 `events_only` 模式即可。

**Q6: 为什么进球后很长时间才检测到？`search_freshness_boost` 有什么用？**

A: 这是数据源延迟问题，有两层原因：

1. **搜索引擎缓存**：`web_search` 返回的往往是 ESPN/Google 体育聚合页，这些页面对比赛事件的更新延迟 3–10 分钟（中文源更慢）。进球已经发生了，但搜索引擎返回的仍是旧比分。
2. **文字直播 URL 缺失**：Stage A 通用搜索（`build_query`）主要返回比分聚合页，很少包含虎扑/新浪的文字直播 URL。没有 URL → Stage B 跳过 → `events=[]` → 用户看到"进球后几轮播报都无事件"。

**`search_freshness_boost=true`（默认）启用 Stage A+ Fallback**：当 Stage A 主搜索没找到文字直播 URL 时，自动用 `site:` 限定搜索精准定位中文实时源（如 `site:nba.hupu.com/games 湖人 勇士 文字直播`）。虎扑/新浪文字直播页的更新延迟仅 ~30s，远快于搜索引擎聚合页的 3–10min。

另外，`build_query` 已改为中英混合查询（如 `"NBA 比分 直播 NBA live scores today"`），中文关键词在前有助于搜索引擎优先返回中文实时源。

如果不想增加额外搜索开销，设 `"search_freshness_boost": false` 可关闭 Stage A+ fallback，回到旧行为。

**Q7: URL 缓存和比分变化事件合成是什么？**

A: 这是两个互补的实时性优化机制：

1. **URL 缓存（`pbp_urls`）**：一旦通过 Stage A 或 Stage A+ 发现了某场比赛的文字直播 URL，缓存到 state file 中。后续轮次直接从缓存取 URL 进入 Stage B webfetch，跳过整个 URL 发现阶段。这意味着第二轮起，Agent 可以更快地拿到实时数据——不需要每次都重新搜索和嗅探 URL。

2. **比分变化事件合成（Score-Change Event Synthesis）**：这是最后一道防线。即使 Stage B 完全失败（webfetch 超时、反爬、JS-only 页面等），只要比分从 0-0 变成 1-0，系统自动合成一条「某队进球！比分 1-0」的 goal 事件。用户不会只看到干巴巴的比分变化，而是能看到一条有意义的进球推送。合成事件的 `id_hash` 使用 `"syn:"` 前缀，不会与 Stage B 的真实事件冲突。如果下一轮 Stage B 成功抓到真实进球事件（有精确时间戳和详细描述），用户可能看到同一次进球的两条推送——但宁可多推一条，不可漏推。

这两个机制与 Stage A+ Fallback 一起构成了三层实时性保障：
- **第一层**：`build_query` 中英混合查询 → 提高首轮中文源命中率
- **第二层**：Stage A+ `site:` 精准搜索 → 首轮未命中时补搜
- **第三层**：URL 缓存 → 第二轮起跳过 URL 发现阶段
- **第四层**：比分变化事件合成 → 所有搜索/抓取都失败时的兜底

---
