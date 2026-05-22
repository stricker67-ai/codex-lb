# Task: fix slow / wedged sticky-session reattach on `/v1/responses`

## Symptom

OpenAI-format clients (notably **openclaw**) calling `POST /v1/responses` with a
previously-seen `prompt_cache_key` (their `durable_session_id` mapped through
`affinity.py`) repeatedly hit a path where codex-lb takes **120–125 seconds** to
return — long enough that any sane client times out first.

Concretely, once a `prompt_cache_key` has been used and the upstream ChatGPT
thread it's affined to has gone stale (account quota event, rotation, restart,
or anything else that drops the server-side thread), subsequent reattach calls
on that same key hang. Fresh keys serve in ~2 seconds against the *same* pool.

The wedge is sticky: every retry on the same `prompt_cache_key` hits the same
wedged state, so the client appears permanently stuck. The workaround on the
client side is to throw away the session and start fresh — which kills the
prompt-cache benefit codex-lb exists to provide.

## Repro

```bash
# 1) Pick any responses-API call with a stable prompt_cache_key.
# 2) Let the affined upstream thread go stale (account rotate / quota hit /
#    container restart of codex-lb upstream side).
# 3) Re-issue the call:
curl -m 200 -X POST http://<codex-lb>/v1/responses \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"model":"gpt-5.5", "input":"...", "prompt_cache_key":"<stale-key>"}'
# → ~122 seconds, eventually returns 200, but well past every client's timeout.
```

A fresh `prompt_cache_key` against the same pool returns in ~2 s — proving the
pool itself is healthy. The latency is internal to codex-lb's reattach path.

Server logs around the wedge show repeated bridge events on the same affined
account before the eventual return:

```
http_bridge_event event=retry_precreated bridge_kind=prompt_cache
  bridge_key=sha256:46e71d1c264b
  account_id=2bdd7ee1-…_b814984b model=gpt-5.5 pending=1
http_bridge_event event=reconnect bridge_kind=prompt_cache
  bridge_key=sha256:46e71d1c264b
  detail=request_stage=reattach, previous_account=2bdd7ee1-…,
         selected_account_id=2bdd7ee1-…  (same account)
  durable_session_id=ee46dd21-db34-4cfa-9acf-c0687fd9ee3d
  pending=None
```

i.e. codex-lb keeps re-selecting the same affined account, retries the upstream
stream, and only after the full upstream timeout falls through.

## Hypothesis

Reattach to a `prompt_cache_key`'s affined upstream waits for the upstream
stream's full idle timeout (looks like ~120 s) before deciding the bridge is
dead. There appears to be no fail-fast on a likely-stale bridge:

- No fast-path probe ("is this thread still resumable?") before the heavy
  reattach attempt.
- No per-reattach attempt budget — the only budget is the upstream's own
  idle/read timeout.
- No automatic eviction of the affinity entry once a reattach has exceeded a
  short reattach budget — so every subsequent call hits the same wedge.

The right behavior, ideally:

1. Reattach attempt has its own short budget (e.g. 5–10 s, configurable).
2. If the reattach budget expires, drop the affinity entry, pick a fresh
   account, serve the request, log the eviction. The client gets a
   ~5–10 s response instead of ~122 s.
3. Optional: a lightweight probe before the reattach to short-circuit obvious
   staleness (skip if the affined account hasn't seen activity in N seconds).

## Where to look

I haven't read every line, but starting points based on a quick scout:

- `app/modules/proxy/affinity.py` — derives and resolves `prompt_cache_key` for
  the request; this is the entry point.
- `app/modules/proxy/sticky_repository.py` — `purge_prompt_cache_before(cutoff)`
  suggests there's already a TTL/eviction concept; the reattach path may need a
  budget-based eviction sibling.
- `app/modules/sticky_sessions/service.py` — `_count_stale_prompt_cache_entries`
  knows what "stale" means; that signal could be applied at request time.
- `app/core/openai/v1_requests.py` and `app/core/openai/requests.py` — the
  request handlers that emit the `http_bridge_event` lines above; the timeout
  budget for the upstream reattach call lives in this region.
- `app/core/config/settings.py` already has a
  `proxy_downstream_websocket_idle_timeout_seconds: 120.0` knob — a sibling
  `proxy_upstream_reattach_budget_seconds` (default ~10) would fit cleanly.

## Acceptance criteria

1. A stale-bridge reattach **must not exceed a configurable short budget**
   (default 5–10 s) before either succeeding or evicting the affinity and
   retrying with a fresh upstream. End-to-end client wall time on a wedged
   `prompt_cache_key` should be ≤ 15 s, not ~122 s.
2. **No regression on the healthy path**: when a bridge reattach genuinely
   succeeds quickly (the common case), prompt-cache hits must still happen —
   we are *not* asking you to disable prompt caching. Verify via an existing
   prompt-cache-hit test or add a small integration test that asserts a
   second call with the same `prompt_cache_key` lands on the same account and
   has cached prefix tokens > 0.
3. Add a metric / log line on eviction so the wedge is observable
   (`bridge_event=evicted_after_reattach_budget` or similar).
4. The new budget must be a setting in `app/core/config/settings.py` with a
   sane default and an env var override. Pattern after the existing
   `proxy_downstream_websocket_idle_timeout_seconds` field.
5. Tests: unit cover for the budget path, plus an integration test that
   simulates a stale upstream and asserts fast-fallback behavior.

## Out of scope

- Don't change the `prompt_cache_key` derivation logic in `affinity.py`.
- Don't change client-facing API shape; this is purely a server-side latency /
  resilience fix.
- Don't touch dashboard UI in this PR.

## Context (FYI, not action items)

- Bug surfaced in the wild from openclaw (a downstream client of this broker)
  whose default LLM idle timeout is 120 s. We worked around it client-side by
  bumping openclaw's `timeoutSeconds` for the codex-lb provider to 300 s,
  which masks the symptom but doesn't fix the bridge wedge — every wedged
  default session still spends 120+ s on its first reply. The proper fix is
  here.
- Repo topic already includes `openclaw`, so this is in-scope for the project.

## Deliverable

A PR against `stricker67-ai/codex-lb` (the fork). I'll review and forward
upstream to Soju06/codex-lb if it's clean.
