# plugin-sentry-cli

> **superseded by [plugin-sentry](https://github.com/yigitkonur/plugin-sentry).** same idea, bigger scope: the triage ladder from this repo plus audit, init, apply and verify, on the current `sentry` v4 cli, with every claim traced to the official docs. install: `claude plugin marketplace add yigitkonur/plugin-sentry` then `claude plugin install sentry@plugin-sentry`. this repo is archived and will not change.

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

then make sure the cli itself is installed:

```shell
brew install getsentry/tools/sentry        # or: curl https://cli.sentry.dev/install -fsS | bash
```

then set up auth (see next section) and you're done — ask the agent "why is the api throwing 500s in prod" and the skill kicks in.

## auth — set one permanent token (the whole point)

here's the thing that motivated this plugin. the sentry **mcp** server authenticates over an oauth device flow, and those sessions expire and quietly log you out — so mid-task your agent starts hitting `not authenticated` and you're back to clicking through a browser. annoying.

the **cli** reads a token straight from an env var. set a personal token once in your shell rc and you're authed basically forever — no device-code dance, no re-login, no surprise logouts. that's the upgrade.

**1. make a personal token.** sentry.io → settings → account → user auth tokens → *create new token*. it starts with `sntryu_`. minimum scopes for read-only debugging:

```
org:read
project:read
event:read
```

add `event:write` only if you want the agent to resolve/archive/merge issues from the cli (the skill treats those as confirm-first anyway).

**2. drop it in your shell rc** (`~/.zshrc`, `~/.bashrc`, `~/.config/fish/config.fish` …):

```shell
export SENTRY_AUTH_TOKEN="sntryu_your_token_here"
export SENTRY_FORCE_ENV_TOKEN=1
```

`SENTRY_FORCE_ENV_TOKEN=1` tells the cli to always prefer this env token over any stored oauth login — so it never silently falls back to an expired session. that's what makes it stick.

**3. reload + verify:**

```shell
source ~/.zshrc        # or just open a new terminal
sentry org list        # should print your org(s) — you're in
```

that's it. the token lives in your environment, the skill picks it up automatically, and the agent never has to think about auth again.

> heads up: `sentry auth whoami` returns a 404 for `sntryu_` personal tokens — that's harmless, the identity endpoint just isn't wired for that token type. everything else works. (org tokens, `sntrys_`, are scoped to one org and may lack read scopes — personal token is the move.)
>
> self-hosted or eu? saas region routing is automatic; for a self-hosted instance use `sentry auth login --url https://sentry.your-company.com`.

oauth still works too — `sentry auth login` — if you'd rather. you just lose the "never log out" benefit.

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
