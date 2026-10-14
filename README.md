# Same-Window Multi-Tab Parallel Automation (via stable `tab_id`)

## Playbook（本仓库的方法论文档）
- **[README](./README.md)** — 同窗多标签并行：用稳定 `tabId` 锁定标签（含实战踩坑）
- **[extension-integration.md](./extension-integration.md)** — **新扩展接入指南**：`tabId` 定位 + 自刷新 + per-tab in-flight 的 4 处改动骨架
- **[agent-browser-cognition.md](./agent-browser-cognition.md)** — 浏览器操作 **46 条**认知：请求前决策 / 速率 / 熔断冷却 / 证据判定 / 会话保护
- **[agent-web-testing-cognition.md](./agent-web-testing-cognition.md)** — 网页**测试与验证**认知：侦察先行 / 断言即证据 / 重放省模型 / 上下文经济性
- **[runner-ownership-and-self-healing.md](./runner-ownership-and-self-healing.md)** — **一个目标一个 runner**：按 id 认领标签 + 按 `windowId` 过滤 / 锚点不能被刚读到的 URL 覆盖 / 双守护互杀 / 自愈必须能停


> How to let **multiple browser-automation tasks each lock onto one tab**
> inside a **single Chrome window** (split view) and run in parallel —
> instead of only ever driving the focused/active tab.

## The problem
With two target tabs open in one window, automation kept driving **only the tab that was focused**;
the other tab never moved. Sometimes a **hidden background tab** appeared out of nowhere.

## Root cause
How the extension resolves the "target tab" decides everything:

| Strategy | Why it fails |
|---|---|
| Only a `window` id | Falls back to the window's **active tab** → only the focused tab moves |
| Match by **URL substring** (`target_url`) | URLs **change after navigation** and look alike across tabs → flaky; worse, on a miss the extension **creates a new inactive tab** and drives *that*, so your real tab looks frozen |
| *(missing)* by **tab id** | This capability didn't exist — yet it's the only stable handle |

**Key insight: the only stable handle is the tab id (`tabId`). It does not change when the URL changes.**

## The fix — add a `tab_id` channel (4 touch points)
Flow: `runner --tab-id` → `pageagent push(tab_id=…)` → backend command field `tab_id` → extension uses `cmd.tab_id` directly.

| # | Layer | Change |
|---|---|---|
| 1 | Backend (`server.py`) | command dict gains `'tab_id': body.get('tab_id')` |
| 2 | Extension (`background.js`) | **highest-priority** branch: `if (cmd.tab_id) targetTabId = (await chrome.tabs.get(Number(cmd.tab_id))).id`; if it can't resolve, **ack `no_tab_by_id`** (never fall back to URL match / never create a tab) |
| 3 | Client (`pageagent.py`) | `push(action, params, window, wait=8, target=None, tab_id=None)` → `body["tab_id"] = tab_id` |
| 4 | Runner | add `--tab-id`, pass `tab_id=` on every push |

> Reload the extension in `chrome://extensions` after (2); restart the backend after (1).

## Usage
```bash
# 1) list tabs in the window and grab their tabIds
pageagent.py tabs --win <window_id>
#   ◀ win=... #819949142  https://example.com/a
#   ◀ win=... #819949213  https://example.com/b

# 2) two runners, each locked to one tab, in parallel
runner.py --win <window_id> --tab-id 819949142 ...
runner.py --win <window_id> --tab-id 819949213 ...
```

**Smoke test (bypass URL, hit the tab id directly):**
```python
push("EXECUTE_JS", {"code": "location.href"}, <window_id>, 15, tab_id=819949142)  # returns that tab's own URL
```

## Pitfalls (design-level)
- ❌ Distinguishing same-site tabs by `target_url` → flaky **and** spawns hidden tabs.
- ❌ Passing only `window` → active-tab fallback.
- ✅ Trust only `tab_id`; fail loudly when unresolved — never guess, never open a new tab.

## Pitfalls from actually landing it (2026-10-13)
1. **Read timeouts (most subtle).** Two runners firing `EXECUTE_JS` (full-page DOM read) at the same window → the extension processes them **serially** → a 25s wait is not enough → empty result → the runner treats it as "not on target → don't persist" → **the tab looks frozen and output stops**. Reading the *same* tab by hand works fine (`readyState:complete`, `tables:N`) — proof it's a **concurrency read-timeout**, not the page. → Raise read wait to **45s**, view-switch to **30s**.
2. **Double-spawn race.** A supervisor loop that `pkill`s then polls twice in a row can **start two runners locking the same tab** → they navigate over each other → "read the wrong page". → `sleep 5` after starting a runner before the next poll.
3. **Ordered worklist looks like an infinite loop.** If the worklist keeps a **fixed order**, every restart re-does the *first few items* → "it's always the same N". → **reshuffle the list each round**.
4. **Un-satisfiable items hog forever.** Some pages genuinely lack the target section → the item can never succeed → stays in the list forever. → **retry cap** (drop after N rounds) via a `*-tries.json`; increment the counter **once per "start the runner"**, not once per poll (or it zeroes out instantly).
5. **Don't "kill all and restart" when count < expected.** That interrupts the healthy runner → restart thrash. → **patch per runner**: start only the missing one (check by `tab_id`).
6. **Keep the login-state probe honest.** During navigation the account menu/avatar is momentarily absent → false "logged out". → probe + **retry** (only trust repeated hits).

## Rate-limit caveat (when automating real sites)
- **Parallel ≠ faster.** For a site's anti-bot layer, the server only sees the **request timing from one account + one egress IP**; *windows/tabs are invisible to it*. Two tabs at 25s each ⇒ ~12.5s/page globally — i.e. the global request rate **doubles**.
- So orchestrate parallel tabs behind a **shared per-domain rate limiter** (single lock + min interval) to keep the global cadence within a safe band.
- On a block signal (e.g. HTTP 429 / Cloudflare 1015), **stop immediately** and back off.

## Reliability stack (what actually fixed the intermittent failures)
Layered by **runner → extension → backend**:

**Runner side**
1. **Bounded retry with backoff** on a failed read: 3 tries, 1.5/3/6s + jitter.
2. **Result validation** — success requires `readyState === 'complete'` **and** non-empty **and** text length > a threshold (e.g. 200). Failing validation = failure → retry, **never silently skip**.
3. **Classify empty vs timeout** — "acked but empty body" and "transport timeout" logged separately (otherwise root-causing stalls).
4. **Dead-letter log** — exhausted retries append to a `dead-letter.jsonl` (slug / err / read_url / ts); never drop silently.
5. **Slug normalization** — sites often normalize `-` ↔ `_`; compare after `re.sub(r"[-_]", "", s)`.
6. **Once-scraped = done** — already-fetched items (incl. those with no data) are not re-fetched, avoiding loops.
7. **Stall alarm** — output file not growing for > N minutes → alert (catches "running but producing nothing").

**Extension side**
8. **One in-flight command per tab** — if the tab already has an unfinished command, do **not** dispatch another (avoids CDP/debugger races → intermittent empty results).
9. **Per-tab debugger session**; parallelism only *across* different tabs.

**Backend / channel**
10. **`command_id` end-to-end**; results routed strictly back to the issuing runner (no single "last result" slot).
11. **Command queue persisted to disk** (`queue file` + one JSON per command + TTL) → survives process restart.
12. **Auth**: loopback-only port + shared token (`X-Auth-Token`) + token file `chmod 600`.
13. **Observability**: queue depth, per-tab in-flight, empty-result rate, dead-letter count, stall duration.

> One line: **"response received" ≠ "read succeeded"**; every failure needs a class, a retry, and a dead-letter — never a silent drop.

## TL;DR
> **Same-window multi-tab parallelism = add a `tab_id` channel and target tabs by stable `tabId` — never by URL or "the active tab".**
