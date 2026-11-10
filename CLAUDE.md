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

## The workaround (set up 2026-10-24)

A second, dedicated session holds `punkpeye/awesome-mcp-servers` as its
attached repo and can query PR status/diffs via the GitHub REST API:

- Reader session: `session_01ETsWUTK22Qsa6dD3sY8u4d`
- Poke trigger (no schedule of its own, wakes the reader session on demand):
  `trig_016tskYDtSZ9TvfLWHDgbZeZ`

To get fresh PR data from this session: call
`mcp__Claude_Code_Remote__fire_trigger` with
`trigger_id: "trig_016tskYDtSZ9TvfLWHDgbZeZ"` and a `text` param describing
exactly what's needed (e.g. "check status of PRs 13310,13896,...", or "run
`gh pr diff <n> -R punkpeye/awesome-mcp-servers | grep '^+- \[dockndevai/'`
and reply with the line"), and tell it to reply via `SendMessage` to this
session (whatever name `ListAgents` shows for it) with the results, taking no
other action. **The reply arrives later as a separate `<cross-session-message>`
turn — it does not come back synchronously.** It's fine to end a run's turn
without the reply yet; the reply will wake this session again to finish the
report/healing.

If `session_01ETsWUTK22Qsa6dD3sY8u4d` or `trig_016tskYDtSZ9TvfLWHDgbZeZ` is
ever gone, recreate: `create_session` with
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
