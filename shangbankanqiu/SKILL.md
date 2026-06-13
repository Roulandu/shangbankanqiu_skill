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
| 3 | **赛况必须靠 LLM web_search** | 不调用任何固定的体育 API（不要假设 API Key / Sportradar / ESPN API 等）。每次轮询都让 Agent 用自带的 web_search 工具实时搜。 |
| 4 | **必须在前台 sleep** | 等待必须在前台执行 `Start-Sleep -Seconds 180` (Windows) 或 `sleep 180` (Unix)，**禁止 nohup / setTimeout / 异步回调**。Agent 的对话循环在 sleep 期间必须真实阻塞。 |
| 5 | **跨平台一致** | 4 家宿主 Agent (codex / cursor / opencode / claude code) 必须按 §标准协议 走出**步骤一致**的对话流（搜→渲染→发→sleep→搜）。 |
| 6 | **绝不静默失败** | webhook 调用必须打印 HTTP 响应；连续 N 轮失败必须停止轮询并向用户报告，不得继续 sleep。 |

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

**核心定位**: 本 Skill **不是后台服务**，而是一段让宿主 Agent 在**当前对话**里反复执行的「播报循环指令」。Agent 干完一轮就 sleep 3 分钟，醒来再干一轮，直到比赛结束或用户喊停。在该对话内 Agent 是真的阻塞的，但用户开新的对话/新窗口/新 Agent 实例依然可以正常使用 codex / cursor / opencode / claude code 做别的事——这就是「应用可以一致运行」的含义。

---

## 各宿主 Agent 工具映射表

为保证 4 家 Agent 走出一致结果，本表列出**每家 Agent 必须使用的工具名**。Agent 在执行本 Skill 时**只能**用本表中的工具——不要自己引入额外工具。

| Agent | 网络搜索 | Shell 执行 | 文件读取 | 文件写入 |
|-------|---------|-----------|---------|---------|
| **opencode** | `websearch_web_search_exa`（首选）/ `webfetch`（兜底直拉网页）/ `task(subagent_type="librarian")`（复杂查证） — 具体工具名以宿主提供的为准，**不要**自己 import 任何 SDK | `bash`（Windows = PowerShell 5.1） | `read` | `write`（覆盖式）/ `edit`（增量改） |
| **Claude Code (claude)** | `WebSearch` + `WebFetch` | `Bash` | `Read` | `Write` / `Edit` |
| **Codex (openai/codex)** | `web_search`（内置） | `shell` | `shell cat ...` | **`shell` + heredoc/echo**（state_file 是简单 JSON，用 `cat > file <<'EOF' ... EOF` 或 `printf '%s' '...' > file`；**不要**用 `apply_patch`，那是 diff 工具） |
| **Cursor (cursor-agent)** | 内置 web search | 内置 terminal | 内置 read_file | 内置 edit_file |

> **如果你的 Agent 没有 web_search 工具**：使用 `webfetch` 直接 GET 一个赛况聚合页（例如 `https://www.google.com/search?q=NBA+live+scores+today` 或 `https://www.flashscore.com/`），再让 LLM 解析返回的 HTML/Markdown。**绝不**降级为调用固定 API。

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
  "max_polls": 60,
  "max_consecutive_failures": 3,
  "state_file": ".shangbankanqiu_state.json",
  "language": "zh-CN"
}
```

| 字段 | 含义 | 默认 |
|------|------|------|
| `poll_interval_seconds` | 两轮之间 sleep 的秒数，建议不低于 60 | 180 |
| `max_polls` | 单次 Skill 运行最多轮询多少次（防止无限循环） | 60 |
| `max_consecutive_failures` | 连续失败几次就终止 | 3 |
| `state_file` | 跨轮去重用的临时状态文件（存上次比分），相对 skill 目录 | `.shangbankanqiu_state.json` |
| `language` | 播报语言 | `zh-CN` |

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
raw     = web_search(build_query(match_type, follow))   # 见 §build_query
matches = parse_search_result(raw)                      # 见 §parse_search_result
state   = classify_state(matches)                       # 见 §classify_state

if state == "未开始":
    sign_and_post(render_preview(matches), config)      # 见 §sign_and_post
    EXIT("已发送赛事前瞻，本次 skill 运行结束")

if state == "已完结":
    sign_and_post(render_final(matches), config)
    EXIT("已发送最终赛果总结，本次 skill 运行结束")

# state == "正在直播" → 进入轮询
prev              = read_state_file(state_path)         # 见 §read_state_file (返回 None / dict)
last_pushed_score = prev.score if prev else None
fail_count        = 0
poll_idx          = 0
poll_interval     = max(config.poll_interval_seconds, 60)   # 强制最小 60 秒

# ========== 轮询循环（在【当前对话】内同步执行；禁止后台化） ==========
while poll_idx < config.max_polls:
    poll_idx += 1

    # ---- 第 1 步: SEARCH ----
    try:
        raw   = web_search(build_query(match_type, follow))
        fresh = parse_search_result(raw)                # 见 §parse_search_result
    except Exception:
        fail_count += 1
        if fail_count >= config.max_consecutive_failures:
            cleanup_state(state_path)
            notify_user("连续 N 次搜索失败，停止轮询"); EXIT
        sleep_in_foreground(poll_interval)
        continue

    # ---- 第 2 步: RENDER ----
    state          = classify_state(fresh)                # 每轮都要重判，状态会变化（直播→完结）
    score_snapshot = extract_score_snapshot(fresh)        # 见 §extract_score_snapshot
    msg            = render_live(fresh, last_pushed_score, score_snapshot)   # 见 §render_live (dedup)
    is_finished    = (state == "已完结")

    # ---- 第 3 步: POST + SIGN ----
    push_result = sign_and_post(msg, config)              # {feishu: bool|null, dingtalk: bool|null}
    all_ok      = all_configured_succeeded(push_result, config)
    if not all_ok:
        fail_count += 1
        if fail_count >= config.max_consecutive_failures:
            cleanup_state(state_path)
            notify_user("Webhook 推送连续 N 次失败"); EXIT
        # 推送失败时跳过 PERSIST，避免 last_pushed_score 被未送达的比分污染
    else:
        fail_count = 0
        # ---- 第 4 步: PERSIST (仅在推送成功时执行) ----
        write_state_file_safely(state_path, { "score": score_snapshot, "ts": now() })
        last_pushed_score = score_snapshot

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

#### `build_query(match_type, follow)`

```text
base = {
  "世界杯": "FIFA World Cup live scores today",
  "NBA":   "NBA live scores today",
}[match_type]

if follow is non-empty:
    return base + " " + follow            # 例: "NBA live scores today Lakers"
else:
    return base                           # 不要拼接空的 "+"，避免查询畸形
```

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
  "summary":       "湖人主场 98-92 凯尔特人，第三节还剩 4:21",  # 一句话中文摘要
  "source_url":    "https://...",          # 来源（可选，记录用）
  "raw":           "<原始片段>"             # 透传原文，便于 LLM 兜底
}
```

**LLM-side 解析指引**（Agent 必须按此顺序尝试）：

1. **关键词扫描** —— 在原始文本里 grep 中英文关键词（见 `classify_state` 的关键词列表 + 比分正则 `\d{1,3}\s*[-:–]\s*\d{1,3}`），命中即填进对应字段
2. **多比赛分割** —— 一次搜索常返回多场比赛；用空行 / `<li>` / Markdown 列表项 / 队名对边界切分，每场各自走 1 步
3. **LLM 兜底** —— 如果正则/关键词覆盖不到（例如返回的是带表格的 HTML），让 LLM 自己读 `raw` 然后填充上述字典；这是允许且推荐的，因为 LLM 本来就在循环里
4. **歧义保护** —— `phase_keywords` 同时命中「Final」+「Live」时优先 `Final`；只有比分没有阶段词时填 `phase="进行中"` 并把 `Live` 加进 keywords，让 `classify_state` 判直播
5. **空结果** —— 如果原始返回里**完全没有比赛**（搜索没命中、或当天没赛事），返回 `[]`；调用方应按「未开始」分支处理或直接告诉用户「今天没比赛」

**禁止**：
- 不要让 Agent 自己写 `.py` 解析脚本——用 LLM 直接读 `raw` 比写脚本更稳
- 不要硬编码 ESPN/官方 API 的 JSON schema——本 Skill 不打那些 API
- 不要在 `parse_search_result` 里就发推送——它只做解析，副作用为零

#### `classify_state(matches)`

按优先级判定（**先看完结，再看直播，最后才是未开始**——避免误把直播当未开始）。`matches` 必须先经过 `parse_search_result` 归一化（见上节），下面的关键词扫描走每个 match 的 `phase_keywords` + `phase` 字段：

```text
if any match contains keyword in {"Final", "FT", "Full-Time", "完场", "终场", "已结束"}:
    return "已完结"
if any match contains keyword in {"Live", "直播中", "Q1", "Q2", "Q3", "Q4", "OT",
                                  "1H", "2H", "上半场", "下半场", "加时", "进行中"}:
    return "正在直播"
return "未开始"
```

歧义时**优先视为正在直播**（误轮询比误退出代价更低）。

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
    body = render_feishu_card(msg)        # JSON 对象
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
    body = render_dingtalk_md(msg)
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

#### `render_live(fresh, last_pushed_score, score_snapshot)` —— 去重逻辑

```text
msg = build_msg_from_fresh(fresh, kind="live")    # 见 §build_msg_from_fresh

if last_pushed_score is not None and scores_equal(score_snapshot, last_pushed_score):
    msg.body = "📌 比分无变化\n" + msg.body         # 前缀注入 body，不破坏 msg 结构

return msg
```

`scores_equal(a, b)` 比较所有进行中比赛的「主队-客队-比分-节/半场」四元组集合是否完全相等。任何一项变化即视为有变化。**只对比赛集合做集合比较，不要对消息文本做字符串 diff**（描述性文字会自然变化）。

#### `extract_score_snapshot(fresh)` —— 比分快照（去重 + 持久化用）

把 `fresh`（`parse_search_result` 返回的 match dict 列表）压成一个**稳定可比较的快照**，喂给 `scores_equal` 和 `write_state_file_safely`：

```text
return [
  { "home": m.home, "away": m.away, "score": m.score, "phase": m.phase }
  for m in fresh
  # 不要包含 summary / source_url / raw 等会自然抖动的字段
]
```

要求：
1. 列表顺序需稳定（按 `home + away` 字典序排序），避免顺序抖动导致误判「有变化」
2. 字段集合固定（home / away / score / phase 四元组），与 §scores_equal 对齐
3. 输出必须可 JSON 序列化（直接喂 `write_state_file_safely`）

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

将 `parse_search_result` 已归一化的 `data`（**match dict 列表**，每个含 `home` / `away` / `score` / `phase` / `phase_keywords` / `summary` 等字段，schema 见 §parse_search_result）组装为标准 `msg` 对象（`{title, body, footer, template}`）。组装时根据 `kind`（"live" / "preview" / "final"）选模板和文案。具体内容参考 §消息格式规范 章节里的飞书/钉钉示例：

| `kind` | `msg.title` 示例 | `msg.template` | `msg.footer` |
|--------|-----------------|---------------|--------------|
| `"live"` | `"🏀 NBA 实时播报"` / `"⚽ 世界杯 实时播报"` | `"blue"` | `"🤖 上班看球播报员 \| 下次更新: 3分钟后"` |
| `"preview"` | `"📋 比赛前瞻"` | `"orange"` | `"🤖 上班看球播报员 \| 比赛尚未开始，敬请期待"` |
| `"final"` | `"🏀 NBA 比赛结束"` / `"⚽ 世界杯 比赛结束"` | `"green"` | `"🤖 上班看球播报员 \| 播报结束"` |

`msg.body` 用 Markdown 编写，包含对阵双方、比分、阶段（节/半场）、关键数据。多场比赛时合并到同一个 body 内（用 `---` 分隔），避免刷屏。

#### `render_feishu_card(msg)` / `render_dingtalk_md(msg)` —— 抽象消息 → 平台 JSON

`msg` 是一个抽象消息对象，由 `render_live` / `render_preview` / `render_final` 产出，包含字段：

| 字段 | 类型 | 含义 |
|------|------|------|
| `msg.title` | string | 卡片标题文本，如 "🏀 NBA 实时播报" |
| `msg.body` | string | 卡片正文（Markdown 或纯文本） |
| `msg.footer` | string | 页脚文本，如 "🤖 上班看球播报员 \| 下次更新: 3分钟后" |
| `msg.template` | string | 飞书卡片 header 颜色：`blue`（直播中）/ `green`（已结束）/ `orange`（前瞻） |

**`render_feishu_card(msg)`** → 按 §飞书消息格式 组装为：

```json
{
  "msg_type": "interactive",
  "card": {
    "header": { "title": { "tag": "plain_text", "content": msg.title }, "template": msg.template },
    "elements": [
      { "tag": "div", "text": { "tag": "lark_md", "content": msg.body } },
      { "tag": "hr" },
      { "tag": "note", "elements": [{ "tag": "plain_text", "content": msg.footer }] }
    ]
  }
}
```

如果 `secret` 非空，由 `sign_and_post` 在该 JSON 顶层追加 `timestamp` 和 `sign` 字段。

**`render_dingtalk_md(msg)`** → 按 §钉钉消息格式 组装为：

```json
{
  "msgtype": "markdown",
  "markdown": {
    "title": msg.title,
    "text": "## " + msg.title + "\n\n" + msg.body + "\n\n---\n\n> " + msg.footer
  }
}
```

钉钉的签名是拼接到 URL query 上，不修改请求体。

#### `notify_stop_hint_once()` / `user_said_stop()` / 优雅终止

⚠️ **物理限制**：在 `Start-Sleep -Seconds 180` / `sleep 180` 前台阻塞期间，Agent **无法**接收用户输入。所以「立刻终止」物理上不可能。

实际行为：

1. **首次启动**: Agent 必须给用户发一句提示——「您可以随时发送『停止 / 结束 / stop』终止播报。由于本 Skill 用前台 sleep 等待 3 分钟（内部按 60 秒切片），您发送停止后**最长等待一个 sleep 切片（约 60 秒）**才会真正退出，这是设计如此。」
2. **每轮 SLEEP 之前** Agent 检查一次最新对话历史里是否含「停止 / 结束 / stop」字样
3. **每个 60 秒 sleep 切片之间** Agent 也再检查一次（见 §chunked sleep），从而把停止响应延迟从 180 秒压到 ≤60 秒
4. **如果发现** → 立即发送最终赛果并退出，**不进入下一个切片或下一轮 sleep**
5. **如果当前已经在 sleep 切片中** → 用户的「停止」要等当前切片自然结束、控制权回到 Agent 之后才会被读到，这是预期行为

### 跨平台 sleep 实现

`sleep_in_foreground(N)` 是所有 4 家 Agent 必须支持的同步阻塞调用。**禁止**用任何异步定时器实现。Agent 应该按以下顺序选择：

> ⚠ **重要：下表只列出底层 sleep 命令本身。实际调用时必须按下方 §宿主 shell 工具 timeout 兜底（chunked sleep）切成 60 秒段，每段一次 shell 工具调用——不要直接用 `Start-Sleep -Seconds 180` / `sleep 180` 单次跑满 3 分钟。**

| 环境 | 命令 | 通过哪个 shell 工具调用 |
|------|------|----------------------|
| **Windows PowerShell 5.1** (opencode/Cursor on Windows 默认) | `Start-Sleep -Seconds 180` | `bash` / `Bash` / `shell` |
| **Windows + Git Bash / WSL** | `sleep 180` | `bash` / `Bash` / `shell` |
| **macOS / Linux** | `sleep 180` | `bash` / `Bash` / `shell` |
| **跨平台兜底（POSIX 不可用时）** | `python -c "import time; time.sleep(180)"` | shell 工具 |

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
        # 这样用户说停止后最长 60 秒内就能响应，而不是非得等满 180 秒
```

这样做有 3 个好处：
1. **不被 shell 工具 timeout kill** —— 每段 60s 远低于任何 Agent 上限
2. **停止响应更快** —— 用户说「停止」最长等 60s（chunk 间检查），不再是 180s
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

## 重要规则（执行 checklist）

| # | 规则 | 备注 |
|---|------|------|
| 1 | **每次执行前先 read config.json** | 获取 webhook 地址 / 轮询间隔 / 失败上限 |
| 2 | **搜索优先用 LLM web_search** | 不要依赖固定 API、不要假设 API Key |
| 3 | **三态分流** | 已完结 → 一次性总结；未开始 → 一次性前瞻；正在直播 → 进入轮询 |
| 4 | **比分无变化也要发** | 在 markdown / 卡片中加注「比分无变化」让群里知道播报仍在运行 |
| 5 | **错误重试 + 上限** | 单步失败立即重试一次；连续 `max_consecutive_failures` 次彻底失败则停止并通知用户 |
| 6 | **优雅终止** | 用户说「停止 / 结束 / stop」时，Agent 在**当前 sleep 切片结束、控制权回到自己后**立刻发送最终赛果并退出（**不进入下一个切片或下一轮 sleep**）。首次启动时必须告知用户：发停止后最长等待一个 sleep 切片（约 60 秒）才会真正退出 —— 这是前台 sleep 切片的物理限制，详见 §优雅终止 + §chunked sleep。 |
| 7 | **多场比赛汇总** | 同一时间多场比赛**合并到一条消息**里发送，避免刷屏 |
| 8 | **必须前台 sleep** | `Start-Sleep -Seconds N`（PowerShell）或 `sleep N`（Unix），**禁止任何形式的后台化** |
| 9 | **不创建调度脚本** | 不要 `write` 任何 `.py` / `.sh` / `.ps1` 来跑循环；循环必须由 Agent 自身在对话里跑 |
| 10 | **语言** | 所有播报内容使用 `config.language`（默认 zh-CN） |

---

## 使用示例

```
用户: 帮我看下现在有什么NBA比赛
Agent: [read config.json] → [web_search "NBA live scores today"]
       → 检测到 2 场进行中
       → [render_live + sign_and_post] (一条消息汇总两场)
       → [Start-Sleep -Seconds 180] ← Windows PowerShell 同步阻塞
       → 醒来 → [web_search] → [post] → [sleep] → ... 循环

用户: 世界杯有比赛吗
Agent: [read config.json] → [web_search "FIFA World Cup live scores"]
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
