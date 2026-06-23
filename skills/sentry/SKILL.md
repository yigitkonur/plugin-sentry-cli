---
name: sentry
description: >-
  Investigate production errors, crashes, stacktraces, logs, performance, and session
  replays through the Sentry CLI (the new `sentry` binary — NOT legacy `sentry-cli`).
  Reach for this whenever you need ground truth about what is failing in production:
  triaging or debugging a real error/crash/exception, pulling a stacktrace or root cause
  for a reported bug, checking what broke after a deploy, inspecting production logs,
  finding slow endpoints/transactions, watching a live error stream, confirming a fix
  actually stopped an error, or whenever the user mentions Sentry, an issue short-id
  (e.g. PROJ-1234, API-7), a sentry.io URL, "check prod errors", "what's erroring",
  "why is X crashing/timing out in prod", "is the deploy healthy". Strongly prefer this
  over guessing about runtime failures you cannot reproduce locally — Sentry holds the
  real stacktrace, tags, breadcrumbs, frequency, and AI root-cause analysis.
---

# Sentry CLI

Ground truth about production failures, via the `sentry` CLI. Every flag and JSON shape here
was verified hands-on against a live org — they are real, not guessed.

## Why a funnel, not a dump

Sentry payloads are huge — a single event embeds a full stacktrace, every breadcrumb, all
tags, and trace spans. Dumping one raw event can cost thousands of tokens and bury the one
line that matters. So the entire workflow is a **funnel: stop at the shallowest rung that
answers the question, and always strip with `--json | jq` instead of printing raw.**

```
Rung 0  triage      issue list   → which issues exist, ranked          (cheap)
Rung 1  diagnose    issue view   → ONE issue + embedded latest event   (usually enough)
Rung 2  corroborate issue events / explore / logs / replay            (only if rung 1 is thin)
Rung 3  AI analysis issue explain / issue plan  (Seer)                 (slow, minutes)
```

Most debugging ends at **Rung 1**: `issue view <ID> --json` already embeds the latest event,
so one call gives you the exception, the in-app stack frames, the tags (release, env, server,
custom IDs), the breadcrumbs, and the trace context. You rarely need to go deeper.

**Hard rules that keep this token-efficient:**
- Never print a raw event/issue JSON. Pipe through `jq` and select the fields below.
- Scope every query: `<org>/<project>` + `--query` + `--period` + `--limit`. Defaults are wide (issue list defaults to 90d).
- `--sort freq` ranks by volume server-side. (`count`/`userCount` come back **null** in `issue list --json`; the real numbers only populate in `issue view`. So: list to find, view to quantify.)
- `issue explain` (Seer) takes minutes — run it backgrounded with a long timeout, do other work, read it later. Never block on it.

## Setup

Install the CLI — `brew install getsentry/tools/sentry` or `curl https://cli.sentry.dev/install -fsS | bash`
(docs: https://cli.sentry.dev/). This is the modern `sentry` binary, distinct from legacy
`sentry-cli` (which is for sourcemaps/release uploads in CI).

Authenticate once, either way:
- `sentry auth login` — OAuth device flow (add `--read-only` to request read-only scopes, good for agents/CI).
- or set `SENTRY_AUTH_TOKEN` to a **personal token** (`sntryu_…`, reaches every org/project you can see) plus `SENTRY_FORCE_ENV_TOKEN=1`. Org tokens (`sntrys_…`) are scoped to one org and may lack read scopes — prefer a personal token.

Verify with `sentry org list`. If you keep a project map in `.claude/sentry-cli.local.md`
(see `references/project-map.md`), read it first so you already know the org slug and which
project maps to the repo you're in — otherwise discover slugs with `sentry project list <org>/`.

Target syntax matters:
- `<org>/<project>` — one project
- `<org>/` — **all** projects in the org (the trailing slash is required)
- `<name>` (no slash) — treated as a project-name *search*, not a list

Full command + flag + JSON-shape + jq-recipe manual: **`references/cli-reference.md`**.

## When to use Sentry (decision tree)

- **"X is broken / erroring / crashing in prod", a bug report, an alert** → triage (Rung 0) to find the issue, then `issue view` (Rung 1) for the stacktrace + root cause.
- **A specific issue short-id or sentry.io URL is mentioned** → go straight to `issue view <ID>` (Rung 1).
- **"Is the deploy/release healthy?" / post-deploy check** → `issue list -q 'is:unresolved firstSeen:-24h' -s new` to find recently-introduced regressions; cross-check `release:<version>`.
- **"It's slow / timing out / which endpoint is slow"** → performance via `explore -d spans` (p95/p99 by transaction). See Performance below.
- **"What's happening right now / live errors"** → `log list <proj> -q level:error -f` to stream, or `issue list -s new -t 1h`.
- **Verifying your own fix** → after deploying, confirm the issue stops firing: check `lastSeen` and event count stop advancing, or `is:unresolved` no longer returns it.

**When NOT to use it:** a bug you can already reproduce locally (just debug it directly — Sentry adds latency, not insight); pre-deploy/local-only code that never shipped; design/architecture questions. Sentry is for *production reality you can't otherwise observe*.

## Rung 0 — Triage (find & rank)

```bash
sentry issue list <org>/<project> -q 'is:unresolved' -s freq -t 24h -n 15 \
  --json --fields shortId,title,level,priority,seerFixabilityScore \
| jq -r '.data[] | "\(.shortId)  [\(.priority)]  seer=\(.seerFixabilityScore // 0)  \(.title)"'
```

- `seerFixabilityScore` (0–1) is Sentry's estimate of how cleanly *you* can fix it — a great
  prioritization signal. High score + high frequency = fix this first.
- Across the whole org: use `<org>/` as the target.
- Query syntax is **implicit-AND, NO `OR`**. Use `key:[a,b]` for OR-within-a-key, `!key:val`
  to negate, `key:>N`, `*wild*`, `is:unresolved|resolved`, `assigned:me`, `level:error`,
  `firstSeen:-24h` (new in last 24h), `age:-1h`. Full syntax in `references/cli-reference.md`.

## Rung 1 — Diagnose (the one call that usually solves it)

```bash
sentry issue view <ID> --json > /tmp/iss.json     # <ID> = PROJ-1234, @latest, @most_frequent
```

Then pull only what you need (these recipes are verified against the real event shape):

```bash
# headline: what, how often, where, which release
jq -r '"\(.title)\nculprit: \(.culprit)\ncount: \(.count)  users: \(.userCount)  lastSeen: \(.lastSeen)\nrelease: \(.event.release.version // "-")"' /tmp/iss.json

# exception type/value + handled-or-not
jq -r '.event.entries[]|select(.type=="exception")|.data.values[]|"[\(.type)] \(.value)  (handled=\(.mechanism.handled))"' /tmp/iss.json

# the stack — APP frames only (skip dependency noise)
jq -r '.event.entries[]|select(.type=="exception")|.data.values[].stacktrace.frames[]?|select(.inApp==true)|"  \(.filename):\(.lineNo) in \(.function)"' /tmp/iss.json

# tags — release, environment, server, runtime, AND custom ids your app sets
jq -r '.event.tags[]? | "\(.key)=\(.value)"' /tmp/iss.json

# last breadcrumbs — what happened right before the error
jq -r '.event.entries[]|select(.type=="breadcrumbs")|.data.values[-6:][]|"[\(.category // .type)] \(.message // (.data|tostring))"' /tmp/iss.json
```

The tags are often the real lead: a failing id, a database error code, a retry attempt number,
the exact server name. The app stack frame gives you `file:line` to open. That combination is
usually enough to write the fix. (One-shot extractor that pulls all of the above at once is in
`references/cli-reference.md`.)

## Rung 2 — Corroborate (only when Rung 1 is ambiguous)

- **Is it still happening / how often?** `sentry issue events <ID> --json | jq '.data[0:5][]|{eventID,dateCreated}'` — individual occurrences with timestamps.
- **Which release/host/tag dominates?** `explore` aggregates, e.g. group errors by a tag:
  `sentry explore <org>/<proj> -d errors -F 'release' -F 'count()' -q 'is:unresolved' -t 7d`
- **Correlated logs:** `sentry log list <org>/<proj> -q "trace:<traceId>"` (traceId from `.trace.traceId` in the issue view). Logs frequently reveal a deeper cause than the exception title (e.g. an HTTP-400 whose logs show the real missing-resource reason).
- **Frontend issue?** `sentry replay list <org>/<proj>` — replays carry `error_ids` to pivot error↔session, plus rage/dead-click counts.

## Rung 3 — Seer AI (root cause + fix plan)

```bash
# Root cause — returns a JSON array of {id, description, relevant_repos}. SLOW (minutes).
timeout 600 sentry issue explain <ID> --json > /tmp/seer.json &   # backgrounded
# … do other work …  then:
jq -r '.[] | "• \(.description)  [\(.relevant_repos|join(", "))]"' /tmp/seer.json

sentry issue plan <ID> --json    # Seer's step-by-step solution plan
```

Seer is genuinely good — on a real FK-violation issue it returned *"a race condition where a
row's parent is deleted before the dependent row is created."* Use it to confirm or challenge
your own hypothesis, not to replace reading the stack.

## Logs

Many projects ship structured logs (some don't — check before relying on it).

```bash
sentry log list <org>/<project> -q 'level:error' -t 24h -n 20          # filter
sentry log list <org>/<project> -q level:error -f                      # -f = live stream
sentry log list <org>/<project> --json | jq -r '.data[]|"\(.timestamp) \(.severity) \(.message)"'
```

Logs carry a `trace` field, so you can pivot from an issue's `.trace.traceId` to its logs.

## Performance

"Performance" = **spans/traces** (profiling is a separate, plan-gated product; the CLI has no
`profile` command — don't assume flame graphs are available). Find slow endpoints with
`explore` on the `spans` dataset (the only dataset where `--sort` is honored):

```bash
# slowest TRANSACTIONS — is_transaction:true is essential: without it you measure every span
# named /foo (much lower) instead of the transaction's end-to-end duration.
sentry explore <org>/<project> -d spans -q 'is_transaction:true' \
  -F 'transaction' -F 'p95(span.duration)' -F 'p50(span.duration)' -F 'count()' \
  -s '-p95(span.duration)' -t 7d -n 10
```

Then narrow inside the worst transaction to find *what* is slow:
```bash
sentry explore <org>/<proj> -d spans -q 'transaction:/your/route' \
  -F 'span.op' -F 'p95(span.duration)' -F 'count()' -s '-p95(span.duration)' -t 7d
sentry explore <org>/<proj> -d spans -q 'transaction:/your/route span.op:http.client' \
  -F 'span.description' -F 'p95(span.duration)' -F 'count()' -s '-p95(span.duration)' -t 7d
```

Then `sentry trace view <id>` / `sentry span list` to confirm serial-vs-parallel on one slow
trace. Details in `references/cli-reference.md`.

## Mutations — pause and confirm first

`issue resolve [--in <release>] <ID>`, `issue archive [--until <cond>] <ID>`, `issue merge`,
`issue unresolve` change shared production state. These are outward-facing actions: don't run
them as part of investigation. Surface what you'd change and get explicit go-ahead first.

## Raw API escape hatch

Anything the subcommands don't cover:

```bash
sentry api 'organizations/<org>/issues/?query=is:unresolved&statsPeriod=24h&limit=5'
sentry schema        # browse available API endpoints
```

## Deepening a decision token-efficiently

Once you've decided an issue matters, deepen by pulling **targeted facets one at a time**
(a specific tag distribution, the breadcrumb trail, the correlated logs for its traceId, the
slow span), never by re-dumping the whole event. Each `jq` selection above is one such facet.
Decide → pull the single facet that tests your hypothesis → conclude. That is the whole game.
