# Integrating stable tab-routing into a NEW browser extension

> Goal: any new automation extension should natively support **locking a task to one tab by `tabId`**
> (same-window multi-tab parallelism), plus **self-reload** and **one-command-in-flight-per-tab**.
> Add the four touch-points below; copy the snippets.

## 0. Architecture (assumed)
```
runner (Python/any)
   │  HTTP
   ▼
local backend (command queue, persisted to disk)     ← /api/command
   │  long-poll
   ▼
extension service worker  ──CDP/scripting──► the tab
```
The extension long-polls the backend; the backend routes each command to the right **tab**.

## 1. Backend: carry `tab_id` on the command
When enqueuing a command, keep `tab_id` (and optionally `window_id`/`target_url`):
```python
command = {
    "id": cmd_id, "action": action, "params": body.get("params", {}),
    "window_id": body.get("window") or body.get("window_id"),
    "tab_id":    body.get("tab_id"),          # ← stable per-tab target
    "target_url": body.get("target_url"),
    "created_at": ..., "status": "pending",
}
```
Route results back by `command.id` (never a single "last result" slot). Persist the queue to disk (`queue file` + one JSON per command + TTL).

## 2. Extension: resolve the target tab by **tabId first**
In the command handler, make `tab_id` the **highest-priority** branch — and **fail loudly** on a miss
(never fall back to "active tab", never `chrome.tabs.create` a hidden tab):
```js
let targetTabId = null;
if (cmd.tab_id != null && cmd.tab_id !== '') {
  try { const t = await chrome.tabs.get(Number(cmd.tab_id)); if (t && t.id != null) targetTabId = t.id; } catch (e) {}
  if (!targetTabId) { await ack(cmd.id, { ok:false, error:'no_tab_by_id', tab_id: cmd.tab_id }); return; }
} else if (cmd.window_id != null && cmd.window_id !== '') {
  /* optional: window-level fallback (active tab) */
} /* else: global command */

// per-tab single in-flight: if this tab already has an unfinished command, DON'T dispatch
if (targetTabId && routing.get(targetTabId)?.status === 'busy') { /* leave for next poll */ return; }
routing.set(targetTabId, { status: 'busy' });
try { /* CDP / chrome.scripting on targetTabId */ } finally { routing.get(targetTabId).status = 'idle'; }
```
Why: matching by URL is flaky (URLs change after navigation) and, on a miss, spawns a *hidden* tab
that silently "runs" while the user's tab looks frozen.

## 3. Extension: self-reload over the command channel
Handle a `RELOAD_EXTENSION` action so the agent can refresh the extension without a human click:
```js
if (cmd.action === 'RELOAD_EXTENSION') { await ack(cmd.id, { ok:true }); chrome.runtime.reload(); return; }
```
> Note: the reload command must actually be **claimed by a poll** — deliver it like any other
> command (with a `window_id`/`tab_id` if your queue lanes by tab). If it never gets claimed,
> only a manual reload / Chrome restart will apply code changes.

## 4. Runner: reference tabs by id, and require readiness
- List tabs, take their **ids**: `GET /api/... `→ `#819949142`, `#819949213`.
- Send every navigate/read command with `tab_id=<that id>`.
- **Validate results**: require `readyState === 'complete'` **and** non-empty content of a minimum length;
  retry with bounded backoff; write exhausted failures to a **dead-letter** log.
- **Wait for readiness** (poll `url + readyState`) instead of a fixed sleep — SPA pages are often
  not ready in 9s, and reading early returns empty.

## 5. Minimal checklist for a new extension
1. Backend command carries `tab_id`; results routed by `command.id`.
2. Command queue persisted to disk + TTL; not a single global slot.
3. Extension resolves target tab by **`tab_id`** first; loud failure on miss; **no hidden-tab fallback**.
4. **One in-flight command per tab** (busy → leave for next poll).
5. `RELOAD_EXTENSION` handler + `chrome.runtime.reload()`.
6. Auth: loopback-only + shared token; token file `chmod 600`.
7. Runner: bounded retry + result validation + dead-letter + readiness wait.
8. Observability: queue depth, per-tab in-flight, empty-result rate, dead-letter count, stall alarm.

> One line: **a new extension should identify its work by a stable `tabId`, never by "the active tab" or a URL.**
