# plugin-sentry-cli

a claude code plugin that teaches your agent to debug production through the **new sentry cli** — the real `sentry` binary, not the crusty legacy `sentry-cli` everyone confuses it with.

the whole idea: when something's on fire in prod, your agent shouldn't guess. sentry already has the actual stacktrace, the tags, the breadcrumbs, how often it fires, and even an ai root-cause take. this plugin gives the agent a tight, token-efficient playbook for getting at all of that from the terminal.

## why it exists

agents are bad at sentry in two opposite ways. either they don't think to look (and hallucinate a fix for a bug they can't see), or they dump a giant raw event json into context and torch 4k tokens to read one stack frame. this plugin fixes both: it nudges the agent to reach for sentry on any "it's broken in prod" moment, and it hands over a **funnel** so it stops at the shallowest rung that answers the question and always strips output with `jq`.

```
rung 0  triage      issue list   → what's broken, ranked        (cheap)
rung 1  diagnose    issue view   → ONE issue + embedded event    (usually enough)
rung 2  corroborate events / explore / logs / replay            (only if rung 1 is thin)
rung 3  ai analysis issue explain / plan  (seer)                 (slow, background it)
```

most debugging ends at rung 1 — `issue view` already embeds the latest event, so a single call gets you the exception, the in-app frames, the tags, the breadcrumbs, and the trace. no need to go deeper most of the time.

## what's in the box

- a `sentry` skill that auto-triggers on prod errors, issue ids, sentry.io urls, "why is x crashing", "is the deploy healthy", slow-endpoint questions, etc.
- `references/cli-reference.md` — a hands-on, verified manual of every command worth knowing (`issue`, `event`, `log`, `explore`, `trace`, `span`, `replay`, `api`) with real flags, real json shapes, and copy-paste `jq` recipes.
- `references/project-map.md` — how to wire up *your* org + project slugs once so the agent doesn't rediscover them every time (see config below).

## install

```shell
/plugin marketplace add yigitkonur/plugin-sentry-cli
/plugin install sentry-cli@plugin-sentry-cli
```

then make sure the cli itself is around and logged in:

```shell
brew install getsentry/tools/sentry        # or: curl https://cli.sentry.dev/install -fsS | bash
sentry auth login                          # oauth, or set SENTRY_AUTH_TOKEN to a personal token
sentry org list                            # sanity check
```

that's it — next time you ask the agent something like "why is the api throwing 500s in prod", the skill kicks in.

## config (optional but nice)

slugs aren't secret, but they're yours, so they live in a local file the plugin reads and git ignores. drop a `.claude/sentry-cli.local.md` in your repo:

```markdown
---
org: your-org-slug
projects:
  api-service: api-service
  legacy-app: old-internal-slug   # when the folder name != the sentry slug
has_logs: ["api-service"]
profiling: false
---

# notes
- anything worth remembering between sessions
```

without it, the agent just discovers your org + slugs live via `sentry org list` / `sentry project list`. the file only saves it a couple of calls. full template + a one-liner to build an "what data does each project even have" matrix is in `references/project-map.md`.

## how this was built

not vibes — it was produced empirically. an agent drove the real `sentry` cli against a live org with a dozen production projects, ran every relevant command, and captured the *actual* flags and json shapes (so there are no invented `--flags` in here). then the skill was eval-tested: three real debugging scenarios (find-the-root-cause, find-the-slow-endpoint, pivot-into-logs) run twice each — once with the skill, once with a bare agent. both found the answer (capable models can read `--help`), but the skill roughly **halved wall-clock time** and kept the agent from wandering, plus it corrects a common wrong turn (assuming sentry "profiling" is available when most plans gate it — performance here means spans/traces). that one profiling caveat was the single assertion the unaided agent flunked.

## the one thing to remember

this is the **new** sentry cli (`cli.sentry.dev`, the `sentry` command). the old `sentry-cli` is a different tool for sourcemaps and release uploads in ci. if your muscle memory types `sentry-cli issues`, that's the wrong binary.

## license

mit. see [LICENSE](./LICENSE).
