# Knowing your org, slugs, and what data each project has

The skill is generic; the specifics of *your* Sentry live in two places you control. Read
this when you don't yet know the org slug, which Sentry project maps to the repo you're in,
or whether a project even ships logs/traces/replays.

## Option A — a local settings file (recommended, persists)

Drop a `.claude/sentry-cli.local.md` in your repo (or workspace root). The skill reads it
first, so it never has to re-discover your slugs. It's a plugin-settings file: YAML
frontmatter + a free-form body. **Keep it out of git** (the plugin's `.gitignore` already
ignores `.claude/*.local.md`) — slugs aren't secret, but the file is per-user/per-machine.

```markdown
---
org: your-org-slug
# folder (repo dir name) → Sentry project slug. List only the ones that differ or that you
# touch often; for the rest, folder name usually equals the slug.
projects:
  api-service: api-service
  web-frontend: web-frontend
  legacy-app: old-internal-slug      # example: folder name != Sentry slug
# data each project actually has (so you don't waste a call): issues|logs|traces|replays
has_logs: ["api-service", "web-frontend"]
has_replays: ["web-frontend"]
profiling: false                      # is profiling enabled on the plan? usually no
region: us                            # us | de (eu). The CLI routes transparently; informational.
---

# Notes

- Busiest error projects, known noisy issues, anything worth remembering between sessions.
- e.g. "api-service: idempotency FK storms come from the worker, not the API."
```

Defaults when this file is absent: assume folder-name == slug, discover everything live.

## Option B — discover live (no setup)

```bash
sentry org list                                   # your org slug(s)
sentry project list <org>/ --json | jq -r '.data[] | "\(.slug)\t\(.platform)"'
```

Build a quick observability matrix so you know which data types are worth querying per
project before spending calls (counts capped by `--limit`, so "100" means "plenty"):

```bash
for s in $(sentry project list <org>/ --json | jq -r '.data[].slug'); do
  iss=$(sentry issue list <org>/$s -q is:unresolved -t 14d -n 100 --json | jq '.data|length')
  log=$(sentry log  list <org>/$s -t 14d -n 100 --json | jq '.data|length')
  tr=$(sentry trace list <org>/$s -t 14d -n 100 --json | jq '.data|length')
  rp=$(sentry replay list <org>/$s -t 14d -n 100 --json | jq '.data|length')
  printf '%-28s iss=%s log=%s trace=%s replay=%s\n' "$s" "$iss" "$log" "$tr" "$rp"
done
```

(zsh doesn't word-split unquoted vars — the `$(...)` here is fine; if you store slugs in a
plain string, iterate with `${(f)slugs}` or a real array.)

## Gotchas worth recording in your local file

- **Folder name ≠ Sentry slug** for some repos — record those explicitly; guessing wastes calls.
- **Not every project ships logs/traces/replays.** If `log list` returns "No logs found",
  fall back to the event's embedded tags/breadcrumbs (`issue view`), not logs.
- **Region (us vs de/eu):** the CLI + a personal token route to the right region
  automatically — no `--url`/region flag needed. Note it only as a reminder.
- **Profiling** is usually not on the plan; treat performance as spans/traces (`explore -d spans`).
