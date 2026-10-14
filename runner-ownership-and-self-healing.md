# One Runner per Target — tab ownership, stable anchoring, and self-healing that can stop

> Companion note to the `tab_id` playbook. The README covers **how to address a tab**;
> this one covers **who owns it** and **what the watchdog around it may do**.
> All four rules below came from watching a long-running runner produce nothing for hours
> while every component involved reported itself healthy.

## TL;DR

| # | Rule | Failure it prevents |
|---|---|---|
| 1 | Claim the tab by id, and filter the tab list by `windowId` | Adopting a same-site tab from *another* window; the tab you watch looks dead |
| 2 | Never re-anchor with the URL you just read | Anchor goes stale on the next navigation → silent empty reads forever |
| 3 | One owner per target: one watcher, one runner | Two watchdogs kill each other's runner; both look healthy, the target never moves |
| 4 | Self-healing must be able to stop | A structurally un-healable case retries forever, spamming alerts and logs |

---

## 1. Claim the tab by id — and filter the tab list by window

**Symptom.** The runner is obviously working — it is producing output — yet the tab the
operator is staring at never moves. It looks like the automation died.

**Root cause.** The command that lists tabs returns tabs from **every window of the
profile**, not just the window you asked about. If you select "the tab whose URL contains
the site" without also matching `windowId`, you adopt the first same-site tab you happen
to see — quite possibly in a *different* window. The runner is then busy somewhere else,
and the tab the operator is watching has no owner at all.

```text
tabs  = list_tabs(window)                            # actually returns ALL windows' tabs
mine  = [t for t in tabs if t.windowId == window]    # <-- do not skip this line
target = first(t for t in mine if SITE in t.url)
```

The extension side of the same API often *does* filter by window internally (its own
bookkeeping needs it), so it is easy to assume the filter is free. If your runner
re-implements the scan, the filter is on you.

## 2. Never re-anchor with the URL you just read

**Symptom.** The first page after start succeeds. Then **every** page after it fails the
"am I on the target page?" check (empty body / not-on-target), and nothing is persisted.
Hours of requests, zero output.

**Root cause.** A runner that holds "my tab" as a URL substring, and refreshes that
substring from the URL it just read, is guaranteed to lose its own anchor: the tab is
navigated to the *next* target immediately afterwards, so the stored URL no longer matches
any tab. If the resolver's fallback is "match by URL", every later command routes to
nothing — or to whatever tab happens to be active.

**Rule.** A stable id, once claimed, *is* the anchor. Do not let URL-derived state
overwrite it. Keep URL matching strictly as a **bootstrap** for when no id is available,
and never let it become the steady-state handle.

Note the ordering that makes this survivable: refuse to re-anchor only while an id is
present, so the URL path still works for setups that genuinely have no ids.

## 3. One owner per target

**Symptom.** Nothing progresses for hours, yet **both** supervisors log "healthy":
one keeps starting a runner, the other keeps "restarting the stuck one".

**Root cause.** Two self-healing loops over the same target. Watcher A's recovery action is
"kill stale runners, start a fresh one"; watcher B's process *is* that fresh one. A kills B
every cycle. Each loop's own view is internally consistent — A sees stagnation, B sees
"started, then exited 20s later (environment problem)" — so neither emits an error worth
reading, and the target is starved the entire time.

**Rule.** Exactly **one** component may start/kill the runner for a given target. If you
need more than one, give them explicitly non-overlapping scopes (different targets), or
gate them behind a single switch so that at most one is armed at a time.

A file-based switch is the cheapest version that survives reboots without a config rewrite:

```text
if exists("SINGLE_OWNER"):  idle forever (do not kill, do not start)
else:                       normal supervision
```

## 4. Self-healing must be able to stop

**Symptom.** A desktop notification every minute for hours; logs growing without bound;
the recovery action attempted forever even though it can never succeed.

**Root cause.** The recovery action assumes a precondition that no longer holds. In our
case the target tab itself was gone, so "restart the runner against the existing tabs"
could never work — the loop kept restarting *and* re-alerting, converting one failure into
permanent noise.

**Rule.** Count **consecutive** failures and degrade:

1. first N cycles — heal and alert as usual;
2. after N — alert **once**, then back off (e.g. one attempt per 30 minutes) and log sparsely;
3. the moment the target makes progress again — reset the counter.

Healing that is structurally impossible must be **reported, not retried**. Pair it with a
user-visible pause switch (a marker file is enough) before leaving anything unattended.

---

## Checklist

- [ ] Runner claims its tab by **id** — not by URL, not by "the active tab".
- [ ] Tab scan filters by `windowId`.
- [ ] A claimed id is never overwritten by a post-navigation URL.
- [ ] Exactly one component starts/kills the runner per target.
- [ ] Consecutive-failure counter with a single alert, then backoff.
- [ ] The counter resets on **real progress** (new output), not on "process is alive".
- [ ] A pause switch exists for unattended runs.

## Why these are easy to miss

Each rule fails *quietly*: the process is up, the logs have lines, the watchdog is doing
something. The only signal is the thing that matters — **new output** — and none of the
health checks above look at it. If you take one habit from this note: make "target produced
something new in the last X minutes" the primary liveness signal, and treat everything
else as decoration.
