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
| 7 | **BREAK-risk: events 不得进入 snapshot/dedup** | `events` 字段是按事件去重的、自然增长的 list。若把它塞进 `extract_score_snapshot` 的快照、或让 `scores_equal` 对它做 string-diff，会导致每轮快照都「看起来变化」→ dedup 失效 → 推送风暴刷爆群。事件去重的唯一正路是主循环里基于 `seen_event_ids` ring buffer 过滤，**禁止**塞进 snapshot（见 §extract_score_snapshot 排除列表与 §scores_equal 语义）。 |
| 8 | **BREAK-risk: 钉钉 markdown 20KB 上限** | 钉钉 markdown `text` 字段约 ~20KB 硬上限，超长会被服务端拒收并返回 `errcode != 0`。冗长的 commentary 直接拼接极易超限——`render_dingtalk_md` **必须**走 18000 字节 size guard（见 T10 / §钉钉 18000 字节 size guard），按 FIFO 丢队首事件直到落入阈值，并附「已截断 N 条」标记。 |
| 9 | **BREAK-risk: 真实配置键禁止 `_` 前缀** | `read_config` 通过 `strip_comment_keys` **递归剥离**所有以 `_` 开头的键（见 §read_config）。把 `include_events` / `max_events_per_msg` / `commentary_dedup_window` 等真实配置键写成 `_include_events` 之类会被**静默剥离**——配置看似已填、运行时全是默认值，极难排查。**只有** `_comment_*` 形态的注释兄弟键才能用 `_` 前缀，真实配置键必须**字面量**拼写。 |

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
  "poll_interval_seconds": 180,
  "max_polls": 60,
  "max_consecutive_failures": 3,
  "state_file": ".shangbankanqiu_state.json",
  "language": "zh-CN",
  "include_events": true,
  "max_events_per_msg": 6,
  "commentary_dedup_window": 30
}
```

| 字段 | 类型 | 默认 | 范围 | 含义 |
|------|------|------|------|------|
| `poll_interval_seconds` | integer | 180 | ≥ 60 | 两轮之间 sleep 的秒数，建议不低于 60 |
| `max_polls` | integer | 60 | ≥ 1 | 单次 Skill 运行最多轮询多少次（防止无限循环） |
| `max_consecutive_failures` | integer | 3 | ≥ 1 | 连续失败几次就终止 |
| `state_file` | string | `.shangbankanqiu_state.json` | — | 跨轮去重用的临时状态文件（存上次比分），相对 skill 目录 |
| `language` | string | `zh-CN` | — | 播报语言 |
| `include_events` | boolean | `true` | `true` / `false` | 是否在比分推送的同时抓取并广播比赛内事件（goal / foul / sub / key_play 等）。`false` 时整段 Stage B 跳过、所有 match `events=[]`，本 Skill 退化为纯比分播报，详见 §Stage B Graceful Degradation。 |
| `max_events_per_msg` | integer | 6 | 1–20 | 每次推送最多附带的事件条数。钉钉 markdown `text` 字段约 **20KB** 硬上限——推荐 4–8（再多容易被钉钉服务端截断；详见 §钉钉 18000 字节 size guard）。 |
| `commentary_dedup_window` | integer | 30 | 1–200 | `seen_event_ids` ring buffer 记多少**轮**的事件 id_hash；buffer 容量 `cap = commentary_dedup_window × max_events_per_msg`。默认 30 × 6 = 180 条 id_hash（state 文件约 9KB）。详见 §state file schema。 |

> **⚠ 配置键命名警告（重申已有规则）**：上表所有「真实键」**绝不**以下划线 `_` 开头；只有 `_comment_*` 形态的注释兄弟键才会被 `strip_comment_keys` 剥离（见 §read_config）。把真实键写成 `_include_events` / `_max_events_per_msg` 之类**会被静默剥离**——配置看起来填了但运行时全是默认值，极难排查。`include_events` / `max_events_per_msg` / `commentary_dedup_window` 三个新键**必须**按本表的字面量拼写出现在 `config.json` 顶层。

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
persisted         = prev if isinstance(prev, dict) else {}   # 永远是 dict，方便 mutate；空 dict 表示「首轮无历史」
last_pushed_score = persisted.get("score")              # None 表示首轮
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

    # ---- 第 1.5 步: STAGE B（事件抓取）—— 见 §Stage B Event Extraction ----
    # P2-1 (Oracle Round 10)：Stage B 必须由主循环显式调用；不能把它当成「另一处文档」就跳过。
    # 任何 Stage B 子步异常一律 graceful=[]——**绝不**让 Stage B 失败影响第 2 步的 push gate。
    if config.include_events:
        for m in fresh:
            # 1.5a. 仅对正在直播的场次抓事件；未开始/已完结跳过。
            # P2-2：match dict 没有 .state 字段；用 phase_keywords（§parse_search_result）判断。
            phase_kws = m.get("phase_keywords", []) or []
            live_kws  = {"Live", "直播中", "Q1", "Q2", "Q3", "Q4", "OT", "1H", "2H",
                         "上半场", "下半场", "加时", "进行中", "中场"}
            if not any(kw in phase_kws for kw in live_kws):
                m["events"] = []
                continue
            # 1.5b. 从 Stage A snippets 选 play-by-play URL（按 §Stage A 5 行优先级表）
            try:
                pbp_url = pick_playbyplay_url_from_stage_a(m)
                if pbp_url is None:
                    m["events"] = []
                    continue
                page_md    = webfetch(pbp_url, format="markdown", timeout_seconds=15)
                # P0-2 (Oracle Round 12): 函数签名是 (page_md, match_key, schema)；不接受 cap。
                # cap 截断由调用方在下一行做（raw_events[-cap_b:]）。
                raw_events = llm_extract_events(
                    page_md,
                    match_key = m.get("home", "") + "|" + m.get("away", ""),
                    schema    = "see §parse_search_result event schema (id_hash + ts + team + type + text)",
                )
                cap_b      = 2 * config["max_events_per_msg"]    # 过取 2× 防丢，默认 12
                m["events"] = raw_events[-cap_b:]
            except Exception:
                m["events"] = []                          # graceful=[]，绝不 raise
    else:
        for m in fresh:
            m["events"] = []                              # include_events=false → 退化成纯比分模式

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

    # 2b. push gate —— USER DECISION (R1, binding):「有新事件也触发 push」
    #      score 未变 + 无新事件 → skip 整轮（不推送、不更新 state）
    #      其它三种组合 → 推送（具体分支见 §render_live 2x2 状态表）
    if not score_changed and not has_new_events:
        log("no score change and no new events — skip push")
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
    msg = render_live(fresh, last_pushed_score, score_snapshot)

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
        # 4c. 整体写盘（score + ts + seen_event_ids 一次性原子持久化）
        persisted["score"] = score_snapshot
        persisted["ts"]    = now()
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
base = {
  "世界杯": "FIFA World Cup live scores today",
  "NBA":   "NBA live scores today",
}[match_type]

if follow is non-empty:
    return base + " " + follow            # 例: "NBA live scores today Lakers"
else:
    return base                           # 不要拼接空的 "+"，避免查询畸形
```

**Stage A 双重职责（当 `config.include_events=true` 时）**：本函数的签名和 base 字符串保持不变；但当 `include_events=true` 时，调用 `web_search(build_query(...))` 之后，Agent **必须额外做一件事**：扫描搜索结果的 snippet / 摘要 / 链接列表，**留住**任何看起来像「文字直播 / 文字实录 / play-by-play / commentary」页面的 URL，作为后续 Stage B（见下方 §Stage B: Event Extraction）的事件抓取入口。被 Stage A 截获的 URL 应该在 match dict 里以 `source_url` 字段透传到 Stage B（或由 Agent 内部保留在临时变量里，二选一，按宿主 Agent 习惯实现），**不要重新发起一次 web_search**。

**URL 优先级链**（Agent 嗅探时按此顺序匹配，命中第一条就停）：

1. `nba.hupu.com/games/playbyplay/{id}` — 中文 NBA 文字实录（首选）
2. `espn.com/nba/playbyplay/_/gameId/{id}` — 英文 NBA play-by-play（NBA 兜底）
3. `espn.com/soccer/commentary/_/gameId/{id}` — 英文世界杯 / 足球 commentary
4. `match.sports.sina.com.cn/livecast/g/live.php?id={id}` — 新浪世界杯直播间（中文，中等可靠度）
5. `nba.sina.com.cn/...` / `zhibo8.cc/...` — discovery-only（JS 重，纯 GET 多半只拿到壳，仅作发现入口，**不**作为主要事件来源）

完整的 shape / pitfalls / fallback 规则全部写在下方 §Stage B 的 source priority chain 表里，**这里不重复**，避免双源真理。

如果 `include_events=false`（默认值或用户显式关闭），Stage A **跳过** URL 嗅探，搜索结果只用于比分/状态判断，下游 Stage B 整段不执行。这是 §Graceful Degradation 的第一档。

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
  "raw":           "<原始片段>",            # 透传原文，便于 LLM 兜底
  "events": [                              # 关键事件列表；本场比赛在搜索结果里能识别到的事件
    {
      "id_hash": "<sha1((match_key + '|' + ts + '|' + text).encode('utf-8'))[:12]>",  # 去重键，REQUIRED；算法见下方（必须 byte-equal Stage B）
      "ts":      "21'" | "Q3 4:32" | "HT" | "",         # 比赛时钟字符串，REQUIRED（可为 ""）
      "team":    "home" | "away" | "",                  # 归属队；中性事件留 ""
      "type":    "goal" | "foul" | "sub" | "card" | "key_play" | "other",   # 6 个枚举（与 §llm_extract_events VALID_TYPES byte-equal）
      "text":    "≤80 个中文字符的单行描述"
    }
  ]
}
```

**`events` 字段说明** (schema field name = `events:` array of event dicts)：

- `events` **可以是空列表 `[]`**，含义：本场目前无可识别事件，或抓取/解析事件失败。空列表是合法值，不视为解析错误。
- `events` 仅承载「本轮搜索结果中能直接读出来的事件」，`parse_search_result` 不主动发起额外请求去补全事件（拉取逻辑属于其他子函数的职责）。
- 字段语义：
  - `id_hash`：长度恰好 12 的十六进制字符串，跨轮去重的唯一键，必填。
  - `ts`：游戏内时钟字符串，**保留原始风格**——足球用 `"21'"` / `"45+2'"` / `"HT"`；篮球用 `"Q3 4:32"` / `"OT 1:05"`；无法获取时填 `""`。必填字段，但允许 `""`。
  - `team`：`"home"` / `"away"` 之一；裁判/中立事件填 `""`。
  - `type`：枚举值之一；不在枚举内的杂项一律归到 `"other"`，不要新造字符串。
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
4. **歧义保护** —— `phase_keywords` 同时命中「Final」+「Live」时优先 `Final`；只有比分没有阶段词时填 `phase="进行中"` 并把 `Live` 加进 keywords，让 `classify_state` 判直播
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

    # 1a. 从 Stage A 截获的搜索结果 snippets / 链接里，按 §Stage A URL 优先级链
    #     选出该场比赛的 play-by-play URL（同一场比赛只挑一条，按表 1→5 顺序）。
    pbp_url = pick_playbyplay_url_from_stage_a(match)

    # 1b. 找不到 URL → graceful skip
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
    cap_b = 2 * config["max_events_per_msg"]   # 默认 12
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
  "type":    "goal|foul|sub|card|key_play|other",   // 6 个枚举值之一
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
    """
    # 1. 输入大小防御：超过 60KB 截断（防止 prompt 爆 LLM 上下文）
    if len(page_md) > 60_000:
        page_md = page_md[-60_000:]   # 取末尾，因为最新事件通常在页面底部

    # 2. 字面 prompt（HARD CONTRACT，4 家 Agent 不许改字符）
    prompt = """你是一个体育赛事文字直播解析器。下面是一段 markdown 格式的比赛文字直播页面。

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
""".replace("%SCHEMA%", str(schema or "")).replace("%MATCH_KEY%", match_key).replace("%PAGE_MD%", page_md)

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
    VALID_TYPES = {"goal", "foul", "sub", "card", "key_play", "other"}
    cleaned = []
    for ev in events[:30]:                       # 硬上限 30 条
        if not isinstance(ev, dict):
            continue
        ts   = str(ev.get("ts",   "")).strip()
        team = ev.get("team")
        if team not in ("home", "away", None):
            team = None
        ev_type = str(ev.get("type", "other")).strip().lower()
        if ev_type not in VALID_TYPES:
            ev_type = "other"
        text = str(ev.get("text", "")).strip()[:80]   # 80 字符硬截断

        if not ts or not text:                   # ts 或 text 为空 → 丢弃
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
  "seen_event_ids":  ["<id_hash>", "<id_hash>", ...]   # 已推送过的事件去重环形缓冲区
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

#### `render_live(fresh, last_pushed_score, score_snapshot)` —— 去重 + events 门控

`render_live` 在比分去重的基础上叠加「事件门控」——即使比分没变，只要本轮存在**新事件**（`fresh` 中每个 match 的 `m.new_events` 至少有一条；`new_events` 由主循环 §主流程伪代码 在调用本函数前预先计算并附到 match dict 上，见 §主循环 push gate），就**仍然推送一条「比分无变化但有新事件」的简化卡片**。

##### 2×2 状态表（branch matrix）

| 比分 (Score) | 新事件 (New Events) | 分支 | body 前缀 |
|--------------|---------------------|------|-----------|
| 已变化 (Changed) | 任意 (≥0) | regular push | (沿用原前缀，例如直接以 `msg.body` 起头，不加无变化前缀) |
| 未变化 (Unchanged) | ≥1 | events-only push | `📌 比分无变化 · 📋 N 条新事件` (N = 本轮新事件总数) |
| 未变化 (Unchanged) | 0 | **跳过 (SKIP)** | — (返回字符串字面量 `"SKIP"`，主循环检测到后 `continue` 不推送) |

> 注：本表只覆盖「正在直播」分支下的实时推送；未开始 / 已完结由 `render_preview` / `render_final` 处理，不进本函数。

##### 分支伪代码（4 个分支，if/else 决策树）

```text
def render_live(fresh, last_pushed_score, score_snapshot):
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

    # === Branch 1: 比分变化（无论事件多少）→ regular push ===
    if score_changed:
        # 不修改 msg.body 前缀；msg.events 已挂载，渲染器自决定是否展示
        return msg

    # === Branch 2: 比分未变化 + 有新事件 → events-only push ===
    if has_new_events:
        msg.body = "📌 比分无变化 · 📋 " + str(new_events_count) + " 条新事件\n" + msg.body
        return msg

    # === Branch 3: 比分未变化 + 无新事件 → SKIP（哨兵）===
    # 主循环看到 "SKIP" 字符串后必须 continue，不调用 sign_and_post
    return "SKIP"

    # === Branch 4 (隐含): score_changed 与 has_new_events 互斥分类已穷尽，无 fallthrough ===
```

##### `scores_equal(a, b)` 语义

`scores_equal(a, b)` 比较所有进行中比赛的「主队-客队-比分-节/半场」四元组集合是否完全相等。任何一项变化即视为有变化。**只对比赛集合做集合比较，不要对消息文本做字符串 diff**（描述性文字会自然变化）。

##### `"SKIP"` 哨兵契约

- 当 `render_live` 判定「比分未变 且 无新事件」时，**必须**返回字面量字符串 `"SKIP"`（而不是返回 None / 抛异常 / 返回空 msg）。
- 主循环（§主流程伪代码 第 2 步 RENDER 之后）**必须**在调用 `sign_and_post` 之前显式判定 `if msg == "SKIP": continue`，跳过 PERSIST 与 POST，直接进入下一个 sleep 切片。
- 该哨兵保证了「无变化无事件」时**不发空消息打扰群成员**，同时让 push gate 决策仍然集中在 `render_live`，而不是散落在主循环里。

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
| `msg.body` | string | 卡片正文（Markdown 或纯文本）。承载比分 + 节次/阶段摘要，**语义不变**——是字符串形式的「比分快照 + phase」概览。 |
| `msg.footer` | string | 页脚文本，如 "🤖 上班看球播报员 \| 下次更新: 3分钟后" |
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

> 实现注：`goal` 行展示时由渲染器根据 `match_type`（或 `match.home` / `match.away` 联赛上下文）二选一；其余 5 个 emoji 跨赛事一致。

**`render_feishu_card(msg)`** → 按 §飞书消息格式 组装为「N 元素」卡片（N ∈ {3, 4}）：

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
    bullet = "\n".join("- " + e["ts"] + " " + emoji_for(e) + " " + e["text"]  for e in msg.events)
    elements.append({ "tag": "div", "text": { "tag": "lark_md",
        "content": "📋 **实时事件**\n" + bullet
    } })                                                                              # Element 2: 事件（CONDITIONAL）
elements.append({ "tag": "hr" })                                                       # 分隔线
elements.append({ "tag": "note", "elements": [{
    "tag": "plain_text",
    "content": msg.footer    # e.g. "🤖 上班看球播报员 | 下次更新: 3分钟后" —— 与 §msg schema (msg.footer) byte-equal，与 §等价 JSON 模板示例一致
}] })                                                                                  # 注脚

# 等价 JSON 模板（仅 msg.events 非空时整 4 元素，msg.events 空时退化为 3 元素）：
{
  "elements": [
    // Element 1: 比分 + phase 摘要（始终存在）
    { "tag": "div", "text": { "tag": "lark_md", "content": msg.body } },

    // Element 2: 实时事件列表（CONDITIONAL —— 仅当 msg.events 非空时插入）
    // 当 len(msg.events) == 0 时整个 element 省略，卡片回退到 3 元素，向后兼容。
    { "tag": "div", "text": { "tag": "lark_md", "content":
        "📋 **实时事件**\n"
        + "\n".join("- " + e.ts + " " + emoji_for(e) + " " + e.text  for e in msg.events)
    } },

      // Element 3: 分隔线（始终存在）
      { "tag": "hr" },

      // Element 4: 页脚 note（始终存在）
      { "tag": "note", "elements": [{ "tag": "plain_text", "content": msg.footer }] }
    ]
  }
}
```

`emoji_for(e)` 严格遵循上方 §Event 类型 → emoji 映射 表：根据 `e.type` 取对应 emoji；`goal` 类型再按比赛是足球还是篮球二选一。

**编排规则**：

1. 当 `msg.events == []` → **省略 Element 2**，elements 数组只剩 3 项（div / hr / note），与历史卡片**完全一致（向后兼容）**。
2. 当 `msg.events` 非空 → 在 Element 1 之后、Element 3（`hr`）之前插入事件 div，elements 数组为 4 项。
3. 事件 div 的 `lark_md` 正文以 `📋 **实时事件**\n` 起头，随后每条事件占一行 bullet：`- {ts} {emoji} {text}`（`ts` 为空字符串时 bullet 仍保留两个空格防止排版崩裂，由渲染器实现兜底）。
4. 事件展示顺序与 `msg.events` 数组顺序一致——`render_live` 已保证按 match → event 抓取序铺平，渲染器不重排。

如果 `secret` 非空，由 `sign_and_post` 在该 JSON 顶层追加 `timestamp` 和 `sign` 字段。

**`render_dingtalk_md(msg)`** → 按 §钉钉消息格式 组装为：

```text
{
  "msgtype": "markdown",
  "markdown": {
    "title": msg.title,
    "text": "## " + msg.title + "\n\n"
            + msg.body
            + (when msg.events is non-empty:
                  "\n\n📋 **实时事件**\n"
                  + "\n".join("- " + e.ts + " " + emoji_for(e) + " " + e.text  for e in msg.events)
              else: "")
            + "\n\n---\n\n> " + msg.footer
  }
}
```

拼接顺序固定为 `title → body → events 段（可选）→ 分隔线 + footer`。`emoji_for(e)` 仍然引用上方 §Event 类型 → emoji 映射 表（与飞书卡片**同一份映射，不要复制粘贴定义**）。当 `msg.events == []` 时整段事件块省略，`text` 退化为「title + body + footer」三段，与历史 markdown 输出**完全一致**。

##### 钉钉 18000 字节 size guard（防超长截断）

钉钉 markdown `text` 字段有约 ~20KB 的硬上限（实测 server 侧裁剪点不稳定，约在 19~20KB 之间），超长会被服务端拒收并返回 `errcode != 0`。`render_dingtalk_md` 在返回前**必须**做尺寸守卫：

- **阈值**：渲染后 `len(rendered_text) <= 18000`（按 UTF-8 字节数；保守留 2KB 安全边际给签名 / URL 编码 / 钉钉服务端额外裁剪）。
- **超额时的截断算法**（迭代丢弃**最旧**事件）：

  ```text
  def render_dingtalk_md(msg):
      events_local = list(msg.events)   # 工作副本，不修改 msg.events 本身

      def assemble(events_to_render, truncated_count):
          ev_block = ""
          if len(events_to_render) > 0:
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

1. **首次启动**: Agent 必须给用户发一句提示——「您可以随时发送『停止 / 结束 / stop』终止播报。由于本 Skill 用前台 sleep 等待 3 分钟（内部按 60 秒切片），您发送停止后**最长等待一个 sleep 切片（约 60 秒）**才会真正退出，这是设计如此。」
2. **每轮 SLEEP 之前** Agent 检查一次最新对话历史里是否含「停止 / 结束 / stop」字样
3. **每个 60 秒 sleep 切片之间** Agent 也再检查一次（见 §chunked sleep），从而把停止响应延迟从 180 秒压到 ≤60 秒
4. **如果发现** → 立即发送最终赛果并退出，**不进入下一个切片或下一轮 sleep**
5. **如果当前已经在 sleep 切片中** → 用户的「停止」要等当前切片自然结束、控制权回到 Agent 之后才会被读到，这是预期行为

### 跨平台 sleep 实现

`sleep_in_foreground(N)` 是所有 4 家 Agent 必须支持的同步阻塞调用。**禁止**用任何异步定时器实现。Agent 应该按以下顺序选择：

> ⚠ **重要：下表只列出底层 sleep 命令本身（按 60 秒切片，单次调用一段）。3 分钟（180s）= 调用 3 次 60 秒。绝不直接用 `Start-Sleep -Seconds 180` / `sleep 180` 单次跑满 3 分钟（会超过 shell 工具默认 timeout 触发杀进程）。详见下方 §宿主 shell 工具 timeout 兜底（chunked sleep）。**

| 环境 | 单段命令（60s） | 通过哪个 shell 工具调用 |
|------|----------------|----------------------|
| **Windows PowerShell 5.1** (opencode/Cursor on Windows 默认) | `Start-Sleep -Seconds 60` × 3 次 | `bash` / `Bash` / `shell` |
| **Windows + Git Bash / WSL** | `sleep 60` × 3 次 | `bash` / `Bash` / `shell` |
| **macOS / Linux** | `sleep 60` × 3 次 | `bash` / `Bash` / `shell` |
| **跨平台兜底（POSIX 不可用时）** | `python -c "import time; time.sleep(60)"` × 3 次 | shell 工具 |

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

**消息体示例 — 实时播报（含事件列表，4 元素布局）**:

当 `include_events=true` 且本轮成功抓到事件时，飞书卡片采用 **4 元素布局**（`div-score` / `div-events` / `hr` / `note`）。`div-events` 的 `lark_md` 正文以 `📋 **实时事件**\n` 起头，每条事件占一行 bullet。`emoji_for(e)` 严格遵循 §Event 类型 → emoji 映射 表（合法 type 仅有：`goal`/⚽ 或 🏀、`score`/🎯、`foul`/🟨、`card`/🟥、`sub`/🔄、`key_play`/⭐、`other`/•）。**绝不**虚构 `yellow_card` / `red_card` 等枚举外类型——红黄牌统一用 `card` + text 描述区分（如 `text="梅西吃到本场第二张黄牌被罚下"`）。

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
            "content": "🤖 上班看球播报员 | 下次更新: 3分钟后"
          }
        ]
      }
    ]
  }
}
```

> 当 `msg.events == []`（Stage B 未抓到任何事件）时，**省略 `div-events` 这一项**，整张卡片退化为前面的 3 元素布局，与历史输出完全一致。

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

**消息体示例 — 实时播报（含事件列表）**:

当 `include_events=true` 且本轮成功抓到事件时，钉钉 markdown `text` 字段会在 body（比分 / 阶段）后追加 **`📋 **实时事件**`** 段，每条事件占一行 bullet（最多 `max_events_per_msg` 条）。emoji 引用 §Event 类型 → emoji 映射 表，与飞书卡片同一份映射。

```json
{
  "msgtype": "markdown",
  "markdown": {
    "title": "⚽ 世界杯 实时播报",
    "text": "## ⚽ 世界杯 实时播报\n\n**阿根廷 vs 法国**\n\n- 🕐 下半场 73'\n- 🏟️ Lusail Stadium\n\n---\n\n### 📊 当前比分\n\n**阿根廷 2 - 1 法国**\n\n---\n\n📋 **实时事件**\n\n- 71' ⚽ 梅西禁区右肋抽射破门，阿根廷再下一城\n- 68' 🔄 法国换人：吉鲁下，科洛·穆阿尼上\n- 64' 🟨 德保罗战术犯规吃到本场首张黄牌\n- 59' ⭐ 姆巴佩单刀被大马丁神扑没收\n\n---\n\n> 🤖 上班看球播报员 | 下次更新: 3分钟后"
  }
}
```

> 当 `msg.events == []` 时，整段「📋 **实时事件**」省略，`text` 退化为「title + body + footer」三段，与上方第一个示例的结构一致。渲染前还要走 §钉钉 18000 字节 size guard：超过 18000 字节按 FIFO 丢队首事件，必要时附「已截断 N 条」标记。

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

---

## 边界情况 / FAQ（commentary 相关）

下列三种情况是 Stage A / Stage B / 推送决策必须**显式优雅处理**的边界，不是 bug，是设计如此：

**Q1: Stage A 嗅探不到 play-by-play URL 怎么办？**

A: play-by-play URL 找不到 → `events=[]` 静默回退；推送照旧（仅比分）。Agent **不要**为此再额外发起一次 web_search 强搜，也**不要**因此跳过该轮推送——比分本身就是有用信息。日志里记一行 `no play-by-play source for {home} vs {away}, skip Stage B` 即可，渲染器看到 `msg.events == []` 会自动走 3 元素飞书卡片 / 「title + body + footer」钉钉 markdown 退化路径。

**Q2: Stage B `webfetch` 在单步预算内超时 / 失败怎么办？**

A: webfetch 在 60s 切片预算内超时 → `events=[]` 优雅跳过当前 match；不阻塞主循环。同理覆盖：HTTP 5xx、页面被反爬挡掉、LLM 提取阶段返回空 / 不合法 schema、URL 临时挂掉等所有 Stage B 失败场景。**绝对不允许**因为 commentary 抓取失败就漏掉这一轮的比分推送，更不允许让单场比赛的失败拖垮整个轮询循环。本场比赛下一轮（3 分钟后）会重新尝试 Stage B。

**Q3: 比分没变，事件去重后也是空，要发吗？**

A: score 不变 AND 去重后 new_events 为空 → no push (BOTH conditions required to skip; either alone still triggers a push — see R1 decision)。换句话说：

- 比分变了 → 必须推（即使没新事件）
- 比分没变但有新事件 → 必须推（事件本身是新信息）
- **比分没变 AND 没有任何新事件**（这两个条件**同时成立**）→ 跳过本轮推送，避免刷屏

这是 R1 决策矩阵里的唯一 skip 分支。任一条件不满足都会触发推送。判断逻辑见 §主循环决策点 R1。

---
