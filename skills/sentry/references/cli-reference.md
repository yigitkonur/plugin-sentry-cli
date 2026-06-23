# Sentry CLI reference

The `sentry` binary (verified `v0.37.0`). Distinct from legacy `sentry-cli` (which is for
sourcemaps/releases/CI uploads). All flags and JSON shapes below were confirmed against a
live org — if something here disagrees with `sentry <cmd> --help`, trust `--help` and update
this file.

## Table of contents
- [Auth & global flags](#auth--global-flags)
- [Targets & selectors](#targets--selectors)
- [Search query syntax](#search-query-syntax)
- [issue list](#issue-list)
- [issue view](#issue-view) — embedded-event JSON shape + recipes
- [issue events / event view](#issue-events--event-view)
- [issue explain / plan (Seer AI)](#issue-explain--plan-seer-ai)
- [issue mutations](#issue-mutations)
- [log](#log)
- [explore (aggregates)](#explore-aggregates)
- [trace / span](#trace--span)
- [replay](#replay)
- [api / schema](#api--schema)
- [Other commands](#other-commands)

## Auth & global flags

- Token via env: `SENTRY_AUTH_TOKEN` + `SENTRY_FORCE_ENV_TOKEN=1`. Personal tokens start
  `sntryu_` (reach every org/project the user can see); org tokens start `sntrys_` (scoped
  to one org, and may lack read scopes — prefer a personal token).
- `sentry auth status` shows the active token; `sentry auth login --token <t>` persists one;
  `sentry auth login --read-only` requests only read scopes (good for CI/agents).
- `sentry auth whoami` returns **404 for personal (`sntryu_`) tokens** — harmless, identity
  endpoint isn't supported for that token type; everything else works.
- Global flags on (almost) every command: `--json`, `--fields a,b,c` (select/limit JSON
  fields, dot-notation ok), `--fresh`/`-f` (bypass cache), `-h`.
- Output is **cached** (you'll see "cached · just now · use -f to refresh"). Add `--fresh`
  when you need up-to-the-second data (e.g. confirming a fix stopped errors).

## Targets & selectors

Target argument forms (apply to list/view/log/trace/explore/replay):
- `<org>/<project>` — explicit org + project
- `<org>/` — ALL projects in org (**trailing slash required**)
- `<project>` — searches for a project by that name across orgs
- omitted — auto-detect from local DSN/config

Issue selectors (for `issue view`/`explain`/`plan`/`events`):
- `PROJ-1234` — short id (searches across orgs)
- `@latest` — most recent unresolved issue
- `@most_frequent` — highest event frequency
- `<org>/@latest` — selector scoped to an org
- numeric id (e.g. `128672846`) also works

## Search query syntax (`-q/--query`)

Verified rules:
- Terms are space-separated and **implicitly ANDed**. There is **no `OR` operator**.
- OR-within-one-key: `key:[val1,val2]` (e.g. `level:[error,fatal]`).
- Negation `!key:value`; comparisons `key:>N` / `key:<N`; wildcards `*term*`;
  quoted phrases `message:"exact phrase"`.
- Built-ins: `is:unresolved`, `is:resolved`, `is:ignored`, `assigned:me`, `has:user`.
- Dates: `age:-24h` (seen in last 24h), `firstSeen:-24h` (first appeared in last 24h —
  i.e. NEW issues), `firstSeen:+7d` (older than 7d), `lastSeen:-1h`.
- Custom tags are queryable: `release:<version>`, `environment:production`, `server_name:...`,
  or any custom tag the app sets.
- Official docs: https://docs.sentry.io/concepts/search/

## issue list

`sentry issue list [target] [flags]`  (alias `sentry issues`, `sentry issue ls`)

Flags: `-q/--query`, `-n/--limit` (max 1000; in multi-project mode the limit is split across
projects), `-s/--sort` = `date|new|freq|user` (default `date`), `-t/--period` (default
**90d** — almost always narrow it), `-c/--cursor` (`next`/`prev`/`first` to paginate),
`--compact/--no-compact`, `-f/--fresh`, `--json`, `--fields`.

`--fields` (and JSON keys): `id, shortId, title, culprit, count, userCount, firstSeen,
lastSeen, level, status, substatus, permalink, project, metadata, assignedTo, priority,
platform, isUnhandled, seerFixabilityScore`.

⚠️ **`count` and `userCount` are `null` in `issue list --json`.** They only populate in
`issue view`. To rank by volume, use `-s freq` (server-side sort) and read titles; to get an
actual number, `issue view` that issue.

JSON shape: `{ "data": [ {issue}, … ], "hasMore": bool, "hasPrev": bool }` — iterate `.data[]`,
NOT `.[]`.

Triage recipe:
```bash
sentry issue list <org>/<proj> -q 'is:unresolved' -s freq -t 24h -n 15 \
  --json --fields shortId,title,level,priority,seerFixabilityScore \
| jq -r '.data[] | "\(.shortId)  [\(.priority)]  seer=\(.seerFixabilityScore // 0)  \(.title)"'
```

## issue view

`sentry issue view <issue> [--web] [--spans N] [--fresh] [--json]`

`--web` opens it in a browser. **The latest event is auto-embedded** under `.event` — this is
why one `issue view` call is usually all you need.

Top-level JSON keys: `id, shortId, title, culprit, count, userCount, firstSeen, lastSeen,
level, status, substatus, priority, isUnhandled, permalink, project, org, metadata,
assignedTo, seerFixabilityScore, replayIds, trace, event`.

`.event` keys: `eventID, dateCreated, dateReceived, message, title, culprit, platform,
release, dist, tags, contexts, user, entries, errors, fingerprints, sdk, size, …`.

`.event.entries[]` is an array of typed blocks. The two that matter:
- `type == "exception"` → `.data.values[]` each has `{type, value, mechanism:{type,handled},
  stacktrace:{frames:[…]}}`. Frames have `filename, absPath, function, lineNo, colNo,
  inApp(bool), context(surrounding source lines)`.
- `type == "breadcrumbs"` → `.data.values[]` each `{category, type, level, message, data,
  timestamp}` in chronological order (last = most recent before crash).

`.event.tags` → `[{key, value}]` (release, environment, runtime, os, server_name, handled,
level, plus any custom app tags). `.trace` → `{traceId, spans:[…]}`.

Most useful single extraction (everything at once):
```bash
sentry issue view <ID> --json | jq '{
  title, count, userCount, lastSeen,
  release: .event.release.version,
  exception: [.event.entries[]|select(.type=="exception")|.data.values[]|{type,value,handled:.mechanism.handled}],
  appFrames: [.event.entries[]|select(.type=="exception")|.data.values[].stacktrace.frames[]?|select(.inApp)|"\(.filename):\(.lineNo) \(.function)"],
  tags: [.event.tags[]?|"\(.key)=\(.value)"],
  traceId: .trace.traceId
}'
```

## issue events / event view

- `sentry issue events <issue> [--json]` → `{data:[{eventID,dateCreated,…}]}` — list of
  individual occurrences of one issue (use to gauge recency/cadence or grab another event id).
- `sentry event view <eventID> <org>/<project> [--json]` → one full event (same shape as the
  embedded `.event`). Needs the project target alongside the event id.
- `sentry event list <issue>` — alias-ish of issue events. `sentry event send` — send a
  synthetic event (testing ingestion / DSN wiring).

## issue explain / plan (Seer AI)

- `sentry issue explain <issue> [--force] [--json]` → **root-cause analysis**. JSON is an
  array: `[{id, description, relevant_repos:[…]}]`. Takes **minutes** ("Starting root cause
  analysis, it can take several minutes…"). Run backgrounded with `timeout 600 … &`. `--force`
  re-runs even if a cached analysis exists.
- `sentry issue plan <issue> [--force] [--json]` → Seer's step-by-step **solution plan**.

These call Sentry's Seer AI server-side. Worth it for gnarly/unfamiliar issues; skip for
obvious ones where the stack already tells you the fix.

## issue mutations

Change shared state — confirm with the user before running:
- `sentry issue resolve <issue> [--in <release>]` — resolve, optionally "in next release".
- `sentry issue unresolve <issue>` — reopen.
- `sentry issue archive <issue> [--until <condition>]` — ignore/mute.
- `sentry issue merge <issue> <issue> …` — merge groups into one canonical.

## log

Sentry **Logs** product (structured logs, separate from events). Not every project ships them.

`sentry log list [target] [flags]`: `-q/--query` (`level:error`, `project:[a,b]`, etc.),
`-n/--limit` (default 100, max 1000), `-t/--period`, `-s/--sort` = `newest|oldest`,
`-f/--follow` (**live stream**, optional poll-interval seconds), `--json`.

JSON row keys: `message, severity, timestamp, timestamp_precise, trace, sentry.item_id`.
The `trace` field correlates a log line to a distributed trace → pivot from an issue's
`.trace.traceId` to its logs with `-q "trace:<id>"`.

```bash
sentry log list <org>/<proj> -q 'level:error' -t 24h --json \
| jq -r '.data[]|"\(.timestamp) \(.severity) \(.message)"'
sentry log list <org>/<proj> -q level:error -f      # tail -f equivalent
```
`sentry log view <id>` for one entry's full detail.

## explore (aggregates)

`sentry explore [target] [flags]` — aggregate/group queries over a dataset. The power tool for
"group errors by tag" and "find slow endpoints".

Flags: `-d/--dataset` = `errors|spans|metrics|logs|replays` (default `errors`),
`-F/--field` (repeatable — plain fields like `title`, `release`, `transaction`, or aggregates
like `count()`, `p50(...)`, `p95(span.duration)`), `-s/--sort` (e.g. `-count()`,
`-p95(span.duration)`), `-q/--query`, `-t/--period` (default 24h), `-n/--limit` (1–1000),
`-c/--cursor`, `-m/--metric` + `--agg` (for the metrics dataset), `-e/--environment` (replays).

⚠️ **`--sort` is only honored on the `spans` dataset.** On `errors` it's ignored (with a warn).

```bash
# group unresolved errors by release (errors dataset)
sentry explore <org>/<proj> -d errors -F release -F 'count()' -q 'is:unresolved' -t 7d
# slowest transactions (spans dataset — sort works here). is_transaction:true is REQUIRED to
# measure transaction (end-to-end) p95; without it you get per-span p95, which is much lower.
sentry explore <org>/<proj> -d spans -q 'is_transaction:true' -F transaction -F 'p95(span.duration)' -F 'count()' -s '-p95(span.duration)' -t 7d -n 10
# drill into one transaction: group by span.op, then by span.description for the slow op
sentry explore <org>/<proj> -d spans -q 'transaction:/your/route span.op:http.client' -F 'span.description' -F 'p95(span.duration)' -F 'count()' -s '-p95(span.duration)' -t 7d
```

## trace / span

- `sentry trace list [target] [-q -s -t -n --json]` — recent distributed traces.
- `sentry trace view <traceId>` — one trace's span tree.
- `sentry trace logs <org>/<project>/<traceId>` (or `<org>/<project> <traceId>`) — logs for a
  trace. (Arg order matters: target before/with the 32-char hex trace id.)
- `sentry span list <org>/<project>` or `<org>/<project>/<traceId>` — spans in a project or
  within a trace; `sentry span view <spanId>`.

## replay

`sentry replay list [target] [-t -n --json]` / `sentry replay view <id>`. Session Replays for
frontend projects. JSON includes `id, duration, count_errors, error_ids, count_segments,
count_rage_clicks, count_dead_clicks, count_urls, browser, os, device, environment, user,
releases, started_at, finished_at`. `error_ids` pivots replay → the errors seen in that
session — great for "show me a user session where this error happened".

## api / schema

- `sentry api '<path>?<querystring>'` — raw authenticated request, returns raw JSON. Use for
  anything the typed commands don't expose. E.g.
  `sentry api 'organizations/<org>/issues/?query=is:unresolved&statsPeriod=24h&limit=5'`,
  `sentry api 'projects/<org>/<proj>/'`.
- `sentry schema` — browse the Sentry API schema to discover endpoints/params.

## Other commands

Available but rarely needed for debugging: `alert` (alert rules), `dashboard`, `monitor`
(cron monitors), `release` (releases/deploys), `repo`, `code-mappings` (stacktrace↔repo
linking), `team`, `project`, `org`, `sourcemap`, `proguard`, `dart-symbol-map`, `trial`,
`init`/`local` (project setup). Run `sentry <cmd> --help` when one becomes relevant.
