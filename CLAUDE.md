# Operational notes for the "Monitor awesome-mcp-servers PRs" scheduled task

This fork (`dockndevai/awesome-mcp-servers`) is the push target for 8 PRs open
against upstream `punkpeye/awesome-mcp-servers`, tracked by the scheduled task
`trig_018jnbgaHWPGHqF4A4q7ypTh` ("Monitor awesome-mcp-servers PRs (auto-heal
conflicts)"), which fires into this session every 8 hours.

## Known environment constraint

This session has `dockndevai/awesome-mcp-servers` attached (needed to push
conflict-healing commits). `punkpeye/awesome-mcp-servers` (same leaf repo
name) **cannot** also be attached here — the environment checks repos out by
name, so a second repo named `awesome-mcp-servers` collides with this one.
As a result `gh`, the GitHub MCP tools, and even anonymous `curl` to
`api.github.com` for `punkpeye/awesome-mcp-servers` all return 403 "not
configured for this session" from this session, no matter what's tried.
`gh pr view` also doesn't work regardless (GraphQL is blocked for these
sessions) — use `gh api repos/<owner>/<repo>/pulls/<n>` (REST) instead.

## The workaround (set up 2026-10-24, reader recreated 2026-11-10)

A second, dedicated session holds `punkpeye/awesome-mcp-servers` as its
attached repo and can query PR status/diffs via the GitHub REST API:

- Reader session: `session_011UANak9dBp3D9jvWATUp2k`
- Poke trigger (no schedule of its own, wakes the reader session on demand):
  `trig_01MJLugzQoHgH1ekoym3MimY`

(The original pair — `session_01ETsWUTK22Qsa6dD3sY8u4d` /
`trig_016tskYDtSZ9TvfLWHDgbZeZ` — went unresponsive: frozen
`connection_status: disconnected` since 2026-10-24 despite multiple pokes
across two later runs. Recreated fresh per the recovery steps below rather
than keep poking a stuck session. The old pair is left alone, harmless
clutter, not deleted.)

To get fresh PR data from this session: call
`mcp__Claude_Code_Remote__fire_trigger` with
`trigger_id: "trig_01MJLugzQoHgH1ekoym3MimY"` and a `text` param describing
exactly what's needed (e.g. "check status of PRs 13310,13896,...", or "run
`gh pr diff <n> -R punkpeye/awesome-mcp-servers | grep '^+- \[dockndevai/'`
and reply with the line"), and tell it to reply via `SendMessage` to this
session (whatever name `ListAgents` shows for it) with the results, taking no
other action. **The reply arrives later as a separate `<cross-session-message>`
turn — it does not come back synchronously.** It's fine to end a run's turn
without the reply yet; the reply will wake this session again to finish the
report/healing.

If `session_011UANak9dBp3D9jvWATUp2k` or `trig_01MJLugzQoHgH1ekoym3MimY` is
ever gone (or goes unresponsive again — see the 2026-11-10 pitfall below),
recreate: `create_session` with
`source_url: "https://github.com/punkpeye/awesome-mcp-servers"`, then
`create_trigger` with `persistent_session_id` set to the new session id and
no `cron_expression`/`run_once_at` (poke-only). Update this file with the new
IDs (and push the update) once recreated.

**Do not create a new one-off "PR status checker" session every run** —
reuse the reader session above. (Many redundant ones from before this fix
exist in the account's session list; they're harmless, just clutter — no
need to clean them up unless asked.)

### Pitfall observed 2026-11-01: this session's own name is not stable

This session's display name (what `ListAgents` shows for it, e.g.
`awesome-mcp-servers-78`) **changes across runs** — a prior run's
`fire_trigger` text baked in a name (`awesome-mcp-servers-2c`) that was
already stale by the time the reader session tried to `SendMessage` back,
so the report silently failed to deliver and sat unread in the reader
session for over a week. When composing the `fire_trigger` text, tell the
reader session explicitly to call `ListAgents` itself, right before sending,
and use whatever name that call returns — never hardcode this session's
current name into the instruction text, since it may already be stale by
the time the reader session acts on it.

Note: `trig_018jnbgaHWPGHqF4A4q7ypTh` itself was created via the web UI
(`http_api`), not by an agent, so it can only be edited by the user at
https://claude.ai/code/routines/trig_018jnbgaHWPGHqF4A4q7ypTh — an agent
session can disable it (`enabled=false`) but not rewrite its prompt. That's
why this workaround lives here instead of in the task's own prompt text.

### Pitfall observed 2026-11-10: `trig_016tskYDtSZ9TvfLWHDgbZeZ`'s own stored prompt is stale, and can't be fixed from here

Calling `get_trigger` on the poke trigger shows its `derived_state.prompt`
still has old, wrong content baked in from a previous run: a stale date
claim ("last check was Oct 15") and a hardcoded dead session name
(`"awesome-mcp-servers-2c"`) to `SendMessage` back to. `update_trigger` on
it is **refused** with "a routine's instructions (its prompt) can be
changed only from the conversation or thread the routine posts into, and
this is not that one" — so this session cannot fix that stored prompt; it
can only be fixed from inside `session_01ETsWUTK22Qsa6dD3sY8u4d`'s own
conversation. The per-poke `fire_trigger` `text` param is appended *after*
that stale stored prompt, not instead of it, so always include in the poke
text an explicit override telling the reader session to follow the
appended text over the base prompt, and to call `ListAgents` itself rather
than trust any name (old or new) in either.

Separately, on this run two `fire_trigger` pokes (~48 minutes apart) to
`trig_016tskYDtSZ9TvfLWHDgbZeZ` produced no `SendMessage` reply at all.
`get_session` on `session_01ETsWUTK22Qsa6dD3sY8u4d` showed `updated_at` and
`post_turn_summary` still frozen at 2026-10-24 (`connection_status:
disconnected`), and `get_trigger`'s `last_run` only reflects the trigger's
**creation-time** run from 2026-10-24 — it does not appear to update for
manual `fire_trigger` pokes, so it's not a reliable way to confirm a poke
landed. Net effect: this run could not get a live re-check of PR status.
The last confirmed data point (from that 2026-10-24 session state) is that
all 8 PRs were MERGED as of then — and GitHub PR merges are irreversible,
so that's still true today unless it was itself wrong/stale. If this
relay keeps going unanswered on a future run, consider recreating the
reader session fresh (see "The workaround" above) rather than continuing
to poke a possibly-stuck one, and update the IDs here.

### Resolution 2026-11-10 (same run): recreated the reader session

The old reader session/trigger pair never answered a third poke either
(3 total unanswered pokes across this and the prior run), so this run
recreated the pair fresh per the recovery steps above:
`session_011UANak9dBp3D9jvWATUp2k` / `trig_01MJLugzQoHgH1ekoym3MimY`
(IDs above already updated to match). Fired an initial check to the new
reader session; its reply arrives later as a `<cross-session-message>`
turn, same async pattern as before. This run's final report is based on
the last confirmed merged status (2026-10-24) pending that fresh
confirmation — treat a future run's fresh reply as the authoritative
update if it contradicts anything here.

### Confirmed 2026-11-10 (later run, same day): all 8 PRs closed/merged — and a better way to read the reader's result than waiting on SendMessage

A later run the same day called `get_session` on `session_011UANak9dBp3D9jvWATUp2k`
directly (no poke needed yet) and found it had already run at
2026-11-10T08:29–08:31 UTC with `post_turn_summary.status_detail`:
*"all 8 PRs checked: closed and merged; no sibling to relay to"* — i.e. the
reader did the check but, same as the stuck-relay pattern above, found no
monitoring session reachable via `ListAgents` at that moment, so its
`SendMessage` reply never went anywhere. **`get_session`'s
`post_turn_summary` field on the reader session is a direct, reliable way
to read its last result without depending on the `SendMessage` relay
working at all** — prefer checking that first before firing a fresh poke.

This run then fired one more poke anyway (`trig_01MJLugzQoHgH1ekoym3MimY`)
for an even fresher double-check, with explicit instructions to call
`ListAgents` right before sending and to override any stale stored
prompt — that reply, if it lands as a `<cross-session-message>`, is the
latest authoritative data; otherwise `get_session` on the reader (as
above) remains the fallback. Status as of this confirmation: **all 8 PRs
(13310, 13896, 13897, 13900, 13901, 13903, 13904, 14166) are CLOSED and
MERGED.**

### Confirmed again 2026-11-11: still all resolved; this session's own `gh` is broken too (irrelevant — nothing to heal)

This run's `gh auth status` showed `GH_TOKEN` itself is invalid in this
session (not just the punkpeye-repo-name-collision issue above — `gh api
repos/punkpeye/awesome-mcp-servers/pulls/13310` also 403'd with the usual
"not configured for this session"). Didn't chase a fix since it's moot:
`get_session` on the reader (`session_011UANak9dBp3D9jvWATUp2k`) still
showed the same terminal `post_turn_summary` from 2026-11-10
("all 8 PRs checked: closed and merged"), and PR merge/close is
irreversible, so no fresh data could contradict it anyway. Fired one more
poke to `trig_01MJLugzQoHgH1ekoym3MimY` for the record (reply, if any,
arrives later as `<cross-session-message>` — doesn't block this run's
report since the state is terminal). Since Step 2 (auto-heal) only
applies to OPEN+CONFLICTING PRs and none are OPEN, the broken `gh` auth
in this session didn't block this run's conclusion. If a future run
needs this session's own `gh` working again (e.g. a 9th PR gets added to
monitor), check `gh auth status` first — it was failing outright here,
not just on the punkpeye repo.
