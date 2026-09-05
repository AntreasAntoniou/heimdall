# Heimdall

**A multi-driver browser test runner for agentic development — author one JSON test plan, run it through real Chrome, headless Playwright/CDP, or an isolated Docker + Xvfb sandbox, in parallel, with unified evidence.**

[![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE)

```bash
npm install -g git+https://github.com/AntreasAntoniou/heimdall.git
heimdall init        # write a sample plan
heimdall run heimdall.plan.json
```

---

## Why Heimdall exists

When an agent (Claude Code, or any coding agent) drives a browser to test a web app, it hits a hard wall: **focus**. A browser extension driving the *real* logged-in Chrome can only have one foreground tab with OS input focus at a time. Click here, type there — every action contends for the same physical focus. You cannot run twenty UI cases at once, because they all fight over the same window. Fidelity is perfect (it's your actual session), but parallelism is impossible and the run is slow and brittle.

The fix is to stop pretending there's only one way to drive a browser. Heimdall splits browser testing into **three drivers**, each a different point on the **fidelity-vs-parallelism** tradeoff:

| Driver | How it drives the browser | Fidelity tier | Parallel? | Needs |
| --- | --- | --- | --- | --- |
| `extension` | The agent's own browser tools drive your **real, logged-in Chrome** (highest fidelity). Heimdall **cannot** drive this itself, so it marks these cases `blocked` to keep reports honest. | `high` | No (single OS focus) | An agent with browser tools |
| `cdp` *(default)* | Playwright over the Chrome DevTools Protocol. Input is dispatched **per-target**, so many `BrowserContext`s run genuinely in parallel with no shared OS focus. | `medium` | **Yes** | Playwright Chromium |
| `container` | The `cdp` path **inside a disposable Docker + Xvfb sandbox** (Linux Chrome). Each case gets its own throwaway container → its own virtual display → its own OS focus, so you can run isolated *headed* sessions in parallel. The safe home for destructive or untrusted systems. | `medium-linux` | **Yes** | Docker |

The same plan runs across all three. `cdp` is the default and the verified-working lane: parallel, fast, headless-or-headed, with HAR, screenshots, and console capture per case. `extension` is the fidelity ceiling (your real session, your real cookies) but cannot be self-driven — Heimdall blocks those cases and tells you to run them via an agent's browser tools. `container` trades a little fidelity (Linux Chrome via Docker's VM) for isolation strong enough to point at risky systems.

Heimdall is **honest about what it did not run.** A case it can't execute faithfully is `blocked`, never silently passed.

---

## Install

Requires **Node.js >= 20**.

```bash
# CLI, directly from the public source release
npm install -g git+https://github.com/AntreasAntoniou/heimdall.git

# Agent Skill (Claude Code, Codex, and compatible clients)
npx skills add AntreasAntoniou/heimdall
```

The package is not currently published to the npm registry. The GitHub install above is
the supported installation path for this release.

The `cdp` (default) driver needs Playwright's Chromium:

```bash
npx playwright install chromium
```

Check your toolchain at any time:

```bash
heimdall doctor
```

`doctor` verifies Node >= 20, Playwright Chromium (the `cdp` lane — the floor), Docker (only needed for `container`), and whether the container image is built. It exits non-zero only if the `cdp` lane isn't ready.

---

## Quickstart

```bash
# 1. Scaffold a sample plan (refuses to overwrite an existing file)
heimdall init                       # writes ./heimdall.plan.json
heimdall init -o smoke.plan.json    # custom path

# 2. Edit baseUrl + cases, then run it
heimdall run heimdall.plan.json
```

A run prints a human-readable report to stderr and writes full evidence to disk:

```
Heimdall — example-plan  (1843ms, v0.1.0)
────────────────────────────────────────────────────────────
PASS  home-loads cdp [medium]
PASS  health-endpoint cdp [medium]
────────────────────────────────────────────────────────────
2 pass  0 fail  0 error  0 blocked  0 skipped   of 2
report: heimdall-runs/latest/report.json
```

Everything lands under the output dir (default `heimdall-runs/latest`):

```
heimdall-runs/latest/
  report.json                       # the full machine-readable RunReport
  cases/<case-id>/
    final.png                       # final-state screenshot (always captured)
    network.har                     # per-case HAR (cdp lane; headers omitted)
```

The process **exits non-zero if any case failed or errored** — and also if cases existed but *nothing actually ran* (everything blocked/skipped, or a `--filter` that matched no cases), so a CI gate can never pass green on a non-run. Blocked/skipped alongside at least one pass do not fail the run.

---

## The plan format

A plan is a single JSON file validated by a Zod schema (`src/schema.ts` is the source of truth). Emit the JSON Schema for your editor or other tools with `heimdall schema`.

### Plan

| Field | Type | Default | Notes |
| --- | --- | --- | --- |
| `name` | string | `"heimdall-plan"` | Shown in the report |
| `baseUrl` | string | — | Base for relative step/fetch URLs |
| `defaultDriver` | `extension` \| `cdp` \| `container` | `cdp` | Per-case `driver` overrides this |
| `storageState` | string | — | Path to a Playwright storageState for injected auth |
| `redaction` | `{ headers?: string[], patterns?: string[] }` | — | Extra response-**header** names and regex **patterns** scrubbed from the report on top of the per-run `${env.X}` secrets (see [Config & secrets](#config--secrets)) |
| `setup` | Step[] | — | Steps run **once** before any case (throwaway context) to seed shared state; if any fails, every runnable case is `blocked` (honesty preserved). **Trusted, ungated fixture** — see note below |
| `teardown` | Step[] | — | Best-effort steps run **once** after all cases to clean up shared state; failures are logged, never fatal. **Trusted, ungated fixture** — see note below |
| `cases` | TestCase[] | — | At least one required |

> **Plan hooks run UNGATED.** Unlike cases, `plan.setup`/`plan.teardown` are **not** subject to the `risk`/`--allow-risk` gate (a hook has no per-step risk field). They are treated as trusted, author-controlled fixtures and run unconditionally whenever any case is runnable. Keep destructive operations out of plan hooks unless you intend them to run without the gate; put risk-gated destructive work in a case (with `risk: "destructive"`) instead.

### TestCase

| Field | Type | Default | Notes |
| --- | --- | --- | --- |
| `id` | string | — | Required, non-empty; used for evidence dirs + filtering |
| `title` | string | — | Optional human label |
| `dimension` | string | — | Which expert/lens authored this case |
| `driver` | `extension` \| `cdp` \| `container` | inherits plan | Per-case override |
| `baseUrl` | string | — | Per-case override of the plan `baseUrl` |
| `setup` | Step[] | — | Steps run on the case's own page **before** `steps` (share its `${vars}`); a failing setup step fails the case and skips its steps/oracles |
| `steps` | Step[] | `[]` | Actions to perform before oracles run |
| `teardown` | Step[] | — | Best-effort cleanup steps run **after** oracles regardless of pass/fail (share the case's page + captured `${vars}`); a teardown failure is noted in `notes`, never flips the verdict |
| `oracle` | Oracle[] | — | **At least one required** — "otherwise it is a click, not a test" |
| `risk` | `read-only` \| `writes` \| `destructive` \| `paid` \| `prod` | `read-only` | `destructive`/`paid`/`prod` are gated (see Risk gating) |
| `priority` | `p0` \| `p1` \| `p2` \| `p3` | `p2` | |
| `tags` | string[] | `[]` | Used by `--filter` |

### Steps

A step's shape is selected by its `action`:

| Action | Fields |
| --- | --- |
| `goto` | `url` (absolute or resolved against baseUrl); `waitUntil?` (`load` \| `domcontentloaded` \| `networkidle` \| `commit`) |
| `click` | `selector` |
| `fill` | `selector`, `value` |
| `select` | `selector`, `value` (string or string[] of option values) |
| `hover` | `selector` |
| `check` | `selector` |
| `uncheck` | `selector` |
| `setViewport` | `width`, `height` |
| `press` | `key` (e.g. `"Enter"`, `"Control+A"`); `selector?` (else dispatched to the keyboard) |
| `waitFor` | `selector?`, `state?` (`visible` \| `hidden` \| `attached` \| `detached`), `timeoutMs?` (no selector ⇒ waits for `networkidle`) |
| `waitForResponse` | `urlContains`, `timeoutMs?`, `as?` (capture a response — already-arrived or future — for oracles incl. `responseTime`); `capture?` |
| `wait` | `ms` |
| `fetch` | `url`, `method?`, `headers?`, `body?`, `as?` (name to store the response under for oracles); `capture?`. Runs in-page with `credentials: "include"`. |
| `request` | `url`, `method?`, `headers?`, `body?`, `as?`, `capture?`; `redirect?` (`follow` \| `manual`, default `follow`) — a browser-context API request via Playwright's `APIRequestContext`. **No page/`goto` needed and not subject to page CORS.** With `redirect: "manual"` a 3xx is returned with **readable** headers (`location` / `hx-redirect` / …) instead of being auto-followed — this is **`request`-step only**, since an in-page `fetch` can't read an `opaqueredirect`. Captures status, headers, body + parsed JSON like `fetch`. |
| `pollUntil` | `url`, `method?`, `headers?`, `body?`, `oracle` (array of response-based oracles), `intervalMs?` (default `1000`), `timeoutMs?` (default `30000`), `as?` — issue the request via the browser-context `APIRequestContext` and re-evaluate `oracle` each poll until **all** pass (then stored as last + `as`), or **throw** when `timeoutMs` elapses (so the case fails honestly). |
| `sse` | `url`, `events?` (resolve after collecting N events), `timeoutMs?` (default `10000`), `closeAfterMs?` (hard stop), `as?` — open an in-page `EventSource` (same about:blank origin guard as `fetch`), collect message events `{event,data,id}` until `events` is reached or `timeoutMs`/`closeAfterMs` elapses (or the server closes the stream), and store an aggregate: the events array under `json`, the newline-joined `data` under `bodyText`, status `200` if any events arrived else `0`. Read it with the `eventCount` / `bodyContains` / `jsonPath` oracles. |
| `load` | `url`, `method?`, `headers?`, `body?`, `times` (total iterations), `concurrency?` (max in-flight; default `min(times, 10)`), `as?` — fire the same request `times` times via the browser-context `APIRequestContext` (bounded by `concurrency`) and store an aggregate (`errorRate` plus latency percentiles `p50`/`p95`/`p99`/`max`). Read it with the `errorRate` / `percentile` oracles. |
| `route` | `urlContains` (substring matched against request URLs), `mode` (`block` \| `abort` \| `fulfill` \| `delay`), `status?`/`body?`/`headers?` (for `fulfill`), `delayMs?` (added latency for `delay`, also usable with `fulfill`) — install a network interception for the rest of the case: `block`/`abort` fail matching requests, `fulfill` returns a synthetic status/body/headers, `delay` slows them. Fault injection / stubbing so failure-recovery is a case, not hand-driving. **Intercepts PAGE-originated requests only** (`goto` + in-page `fetch`/`eval`); it does **not** affect the `request`/`load`/`pollUntil` steps (those use Playwright's APIRequestContext, which bypasses `page.route`) — drive those flows through an in-page `fetch` step if you need them faulted. |
| `race` | `steps` (array of **network** steps — only `request`/`fetch`/`load` are allowed, enforced at parse time) — run the nested steps concurrently and wait for **all** of them to settle, capturing every response for cross-assertion (two writers racing one resource, optimistic-lock contention). Unlike `load` (which fires N *identical* requests), `race` fires N *distinct* steps at once. |
| `eval` | `expression` (JS evaluated in the page, for side effects) |
| `screenshot` | `name?` |

#### Variable capture + `${...}` templating

`fetch`, `request`, and `waitForResponse` accept an optional **`capture`** map that pulls values out of the step's response into a per-case variable bag:

```jsonc
"capture": {
  "runId": { "header": "x-run-id" },     // from a response header (case-insensitive)
  "userId": { "jsonPath": "data.user.id" } // or a JSON path into the parsed body
}
```

Captured values are then substitutable as `${name}` in later steps' `url`, `body`, `headers` values (`goto`/`fetch`/`request`/`pollUntil`/`load`/`sse`) and in `waitForResponse`'s `urlContains`. A token beginning `env.` (e.g. `${env.API_TOKEN}`) resolves from `process.env` at runtime instead of the capture bag — this is how a plan injects secrets via the environment rather than inlining them (see [Config & secrets](#config--secrets)). Unknown `${tokens}` are left verbatim. Together with `request` `redirect: "manual"` and `pollUntil`, this expresses a full async lifecycle — **POST → capture an id from a redirect header → poll until ready → assert** — entirely within a plan:

```jsonc
{ "action": "request", "url": "/start", "method": "POST", "redirect": "manual",
  "capture": { "runId": { "header": "x-run-id" } }, "as": "start" },
{ "action": "pollUntil", "url": "/poll?id=${runId}", "timeoutMs": 5000, "as": "p",
  "oracle": [{ "assert": "jsonPath", "path": "ready", "equals": true }] }
```

### Oracles

The pass/fail layer. A case passes only if **every** oracle passes.

| Assert | Fields | Checks |
| --- | --- | --- |
| `status` | `equals`, `of?` (named fetch response; defaults to last) | HTTP status equals |
| `statusIn` | `values` (non-empty int[]), `of?` | Captured response status is one of `values` |
| `statusRange` | `min`, `max`, `of?` | `min <= status <= max` (inclusive; e.g. `200..499` for "not 5xx") |
| `responseOk` | `of?` | Response was 2xx |
| `jsonPath` | `path` (e.g. `"data.user.id"` or `"$.ok"`), `equals?`, `exists?`, `of?` | JSON body at path equals / exists |
| `visible` | `selector` | Element is visible |
| `hidden` | `selector` | Element is not visible |
| `textContains` | `selector`, `value` | Element text contains substring |
| `urlContains` | `value` | Current URL contains substring |
| `titleContains` | `value` | Page `<title>` contains substring |
| `count` | `selector`, `equals` | Number of matching elements equals |
| `attribute` | `selector`, `name`, `equals?`, `contains?`, `exists?`, `matches?` | Element attribute equals / contains / matches `RegExp`; `exists:true` asserts presence (any value), `exists:false` asserts absence |
| `responseTime` | `maxMs`, `of?` | Captured response's duration is within budget |
| `header` | `name` (case-insensitive), `equals?`, `contains?`, `of?` | Captured response header equals / contains (fails clearly if absent) |
| `jsonType` | `path`, `type` (`string` \| `number` \| `boolean` \| `array` \| `object` \| `null`), `of?` | JSON value at path has the given type |
| `nonEmpty` | `path?`, `of?` | JSON value at path is a non-empty string/array; with no `path`, the response body is non-empty |
| `bodyContains` | `value`, `of?` | Response body text includes `value` |
| `jsonMatch` | `path`, `pattern`, `of?` | String at path matches `RegExp(pattern)` |
| `titleMatches` | `pattern` | Page `<title>` matches `RegExp(pattern)` |
| `urlMatches` | `pattern` | Current URL matches `RegExp(pattern)` |
| `eventCount` | `min?`, `equals?`, `of?` | Number of events collected by an `sse` step is at least `min` and/or exactly `equals` |
| `errorRate` | `max` (0..1), `of?` | A `load` step aggregate's error rate (non-2xx or thrown) is `<= max` |
| `percentile` | `p` (`50` \| `95` \| `99` \| `"max"`), `maxMs`, `of?` | A `load` step aggregate's chosen latency percentile is `<= maxMs` |
| `noConsoleErrors` | — | No `console.error` / page errors during the case |
| `evalTruthy` | `expression` | JS evaluated in the page is truthy |
| `a11y` | `maxImpact?` (`minor` \| `moderate` \| `serious` \| `critical`, default `serious`), `include?`/`exclude?` (CSS selectors) | Runs an axe-core accessibility audit over the page and fails on any violation **at or above** `maxImpact`; the violation list is captured as evidence. Self-drivable on `cdp`/`container`. |
| `screenshotMatches` | `baseline` (path to a baseline PNG), `maxDiffRatio?` (0..1, default `0.01`), `selector?` (clip to an element instead of full page) | Pixel-compares a screenshot of the page (or `selector`) against a committed golden via pixelmatch and fails when the fraction of differing pixels exceeds `maxDiffRatio`; the diff image is written as evidence. |

### A concrete plan

```json
{
  "name": "example-plan",
  "baseUrl": "http://localhost:3000",
  "defaultDriver": "cdp",
  "storageState": "auth/session.storageState.json",
  "cases": [
    {
      "id": "home-loads",
      "title": "Home page renders without console errors",
      "dimension": "functional",
      "steps": [{ "action": "goto", "url": "/" }],
      "oracle": [
        { "assert": "noConsoleErrors" },
        { "assert": "visible", "selector": "body" }
      ],
      "risk": "read-only",
      "priority": "p0",
      "tags": ["smoke"]
    },
    {
      "id": "search-flow",
      "title": "Searching navigates to results",
      "steps": [
        { "action": "goto", "url": "/" },
        { "action": "fill", "selector": "#q", "value": "heimdall" },
        { "action": "press", "key": "Enter" },
        { "action": "waitFor", "selector": "[data-testid=results]", "state": "visible" },
        { "action": "screenshot", "name": "results" }
      ],
      "oracle": [
        { "assert": "urlContains", "value": "/search" },
        { "assert": "textContains", "selector": "[data-testid=results]", "value": "heimdall" }
      ],
      "risk": "read-only",
      "priority": "p1",
      "tags": ["search"]
    },
    {
      "id": "health-endpoint",
      "title": "Health endpoint returns 200 and ok:true",
      "dimension": "api",
      "steps": [
        { "action": "fetch", "url": "/api/health", "method": "GET", "as": "health" }
      ],
      "oracle": [
        { "assert": "status", "equals": 200, "of": "health" },
        { "assert": "responseOk", "of": "health" },
        { "assert": "jsonPath", "path": "ok", "equals": true, "of": "health" }
      ],
      "risk": "read-only",
      "priority": "p1",
      "tags": ["api", "smoke"]
    }
  ]
}
```

---

## CLI commands

### `heimdall run <plan>`

Run a plan through its drivers (`cdp`/`container`) and report.

| Flag | Default | Meaning |
| --- | --- | --- |
| `-d, --driver <driver>` | — | Force a driver for every case: `cdp` \| `container` |
| `-o, --out <dir>` | `heimdall-runs/latest` | Output dir for evidence + report |
| `-b, --base-url <url>` | — | Base URL for relative step/fetch URLs (overrides plan) |
| `-s, --storage-state <file>` | — | Playwright storageState for injected auth |
| `-c, --concurrency <n>` | `4` | Max parallel cases **per driver** |
| `-r, --retries <n>` | `0` | Retry a failed/errored case up to n times (helps with flake) |
| `--timeout <ms>` | — | Overall wall-clock budget per case (per-case `timeoutMs` wins) |
| `--allow-risk` | `false` | Permit cases marked `destructive`/`paid`/`prod` to run |
| `--headed` | `false` | Visible browser window (`cdp` only) |
| `--insecure` | `false` | Disable TLS validation (ignored when `--storage-state` is set) |
| `--trace [mode]` | `off` | Record a Playwright `trace.zip` per case: `off`/`on`/`on-failure` (bare `--trace` ⇒ `on-failure`) |
| `--video [mode]` | `off` | Record a video per case: `off`/`on`/`on-failure` (bare `--video` ⇒ `on-failure`) |
| `--html [file]` | — | Also write a self-contained HTML report (default `<out>/report.html`) |
| `--junit <file>` | — | Also write a JUnit XML report for CI |
| `--group-by <key>` | — | Group report rows (terminal + HTML) and the JUnit `classname` by case `tag` or `dimension`; also surfaces a prominent "Blocked (N)" panel |
| `--diff <prevReport.json>` | — | Compare against a previous run's `report.json` and print a regression diff (newly-failing / newly-passing / still-failing / added / removed) after the run |
| `-C, --config <file>` | `heimdall.config.json` | Load run defaults + an `env` seed from a config file before parsing the plan (see [Config & secrets](#config--secrets)) |
| `-f, --filter <idOrTag>` | — | Only run cases matching an id substring or exact tag (repeatable) |
| `--merge-results <file>` | — | Fold an externally-produced `Result[]` (or a `{ results: [...] }` report) into the run — e.g. `extension` cases the agent drove in real Chrome. A result whose `id` matches a (typically `blocked`) case replaces it **verbatim**; unknown ids are appended. The merge never fabricates a passing verdict — see [`heimdall extensions`](#heimdall-extensions-plan) for the two-lane workflow |
| `--json` | `false` | Print the full JSON report to stdout |
| `-v, --verbose` | `false` | Verbose debug logging |

```bash
heimdall run plan.json
heimdall run plan.json -d container --allow-risk        # isolated, risky SUT
heimdall run plan.json -f smoke -f search -c 8          # filter + more parallelism
heimdall run plan.json --retries 2 --timeout 30000      # tolerate flake, cap each case
heimdall run plan.json --html --junit results.xml       # shareable + CI reports
heimdall run plan.json --group-by dimension             # group rows by authoring lens
heimdall run plan.json --diff prev/report.json          # regression diff vs last run
heimdall run plan.json --config heimdall.config.json    # defaults + env-seeded secrets
heimdall run plan.json --trace on-failure --video on-failure  # capture failures for replay
heimdall run plan.json --headed                         # watch the cdp browser
heimdall run plan.json --json > report.json             # machine-readable to stdout
```

A captured trace is replayable with `npx playwright show-trace <out>/cases/<id>/trace.zip`.

### `heimdall doctor`

Check the toolchain: Node, Playwright Chromium, Docker, and the container image. Exits non-zero only if the `cdp` lane isn't ready.

### `heimdall gc`

Prune orphaned Playwright browser revisions from the cache (Playwright hoards every revision it ever downloaded). Keeps the newest revision per browser family.

| Flag | Default | Meaning |
| --- | --- | --- |
| `-y, --yes` | `false` | Actually delete (default is a dry run) |

### `heimdall init`

Write a sample plan you can edit. Refuses to overwrite an existing file.

| Flag | Default | Meaning |
| --- | --- | --- |
| `-o, --out <file>` | `heimdall.plan.json` | Where to write the sample plan |

### `heimdall schema`

Emit the JSON Schema for a plan (for editors / other tools).

| Flag | Default | Meaning |
| --- | --- | --- |
| `-o, --out <file>` | stdout | Write to a file instead of stdout |

### `heimdall extensions <plan>`

Emit a manifest of the plan's `driver: extension` cases for an agent to drive in real Chrome. Heimdall cannot self-drive your logged-in browser, so it `block`s those cases; this command extracts them as an actionable manifest — each case's `steps` + `oracle` plus a human `tapHint` ("Drive X in real Chrome starting at …, then verify N step(s) + M oracle(s).").

| Flag | Default | Meaning |
| --- | --- | --- |
| `-o, --out <file>` | stdout | Write the manifest to a file instead of stdout |

The two-lane workflow closes with `--merge-results`: run the self-drivable lanes, drive the extension cases via the agent's real-Chrome tools, write their outcomes as a `Result[]`, then fold them back into a single, honest, whole-plan report:

```bash
heimdall extensions plan.json -o ext-manifest.json   # what the agent must drive
# … agent drives those cases in real Chrome, writes ext-results.json …
heimdall run plan.json --merge-results ext-results.json   # one unified RunReport
```

### `heimdall build-image`

Build the Docker image used by the `container` driver (`heimdall:local`). It also auto-builds on first container run.

| Flag | Default | Meaning |
| --- | --- | --- |
| `-t, --tag <tag>` | `heimdall:local` | Image tag |

### `heimdall auth save`

Open a browser, log in manually, and save the session to a file. Your password/MFA never touch Heimdall.

| Flag | Default | Meaning |
| --- | --- | --- |
| `-u, --url <url>` | *(required)* | Login URL to open |
| `-o, --out <file>` | `auth/session.storageState.json` | Where to save the storageState |

### `heimdall mcp`

Run Heimdall as an [MCP](https://modelcontextprotocol.io) server over stdio, so Claude Code (or any MCP client) calls it as a **native tool** instead of shelling out.

```bash
# register once with Claude Code
claude mcp add heimdall -- heimdall mcp
```

It exposes three tools:

| Tool | Input | Returns |
| --- | --- | --- |
| `run` | `plan` (object) + optional `baseUrl`/`driver`/`concurrency`/`retries`/`timeoutMs`/`allowRisk`/`storageState`/`trace`/`video`/`filter`/`outDir` | the full `RunReport` (summary + per-case results + evidence paths) |
| `doctor` | — | toolchain checks + `cdpReady` |
| `schema` | — | the JSON Schema for a plan |

The agent builds a plan, calls `run`, and reads the structured report back — no CLI parsing, no temp files. An invalid plan comes back as an MCP error with the validation message.

---

## Risk gating

Every case carries a `risk`. Cases marked `destructive`, `paid`, or `prod` are **blocked unless you pass `--allow-risk`** (and a human has confirmed it). `read-only` and `writes` run freely. Blocked cases appear in the report as `blocked` with the reason — they are never silently skipped or passed.

`extension` cases are always blocked too: Heimdall cannot self-drive your real Chrome, so it records them as `blocked` with a pointer to run them through an agent's browser tools.

---

## Auth via `storageState`

Brittle, scripted logins are a classic test-suite tax. Heimdall sidesteps them with Playwright's **storageState** — a snapshot of cookies + localStorage you capture once and inject into every run.

```bash
# 1. Capture a session interactively (real headed browser; you sign in by hand)
heimdall auth save -u https://app.example.com/login -o auth/session.storageState.json

# 2. Inject it into a run
heimdall run plan.json --storage-state auth/session.storageState.json
#    …or set "storageState" in the plan, or per the cdp/container context.
```

The `cdp` driver loads the storageState into every `BrowserContext`; the `container` driver copies it into the sandbox and reuses it inside. **The session file holds live cookies/tokens — keep it gitignored and never share it.**

---

## Config & secrets

An optional `heimdall.config.json` (loaded from the cwd, or an explicit `--config <file>`) supplies run-level defaults and — crucially — an `env` map that **seeds `process.env` for this run** so a plan can reference secrets via `${env.NAME}` templating instead of inlining them:

```jsonc
{
  "baseUrl": "https://staging.example.com",
  "defaultDriver": "cdp",
  "concurrency": 6,
  "retries": 1,
  "storageState": "auth/session.storageState.json",
  "env": {
    "API_BASE": "https://api.staging.example.com"
  }
}
```

- **Defaults precedence is CLI > plan > config.** A config value only fills a hole the CLI flag (and, for `baseUrl`/`defaultDriver`/`storageState`, the plan) left open.
- **`env` never clobbers a real environment variable** — an entry is applied only when the key is unset, so a CI-provided `API_TOKEN` always wins over the config file. Put non-secret defaults here; keep actual secrets in the real environment.
- In a plan, `${env.API_TOKEN}` resolves from `process.env` at runtime (unknown env ⇒ empty string), so tokens live in the environment, never in the committed plan JSON. The `container` driver forwards the env vars a case references into the sandbox via `docker run -e NAME` (the value is pulled from the host environment at run time — the on-disk sub-plan keeps the `${env.NAME}` token, never the resolved secret), so `${env.*}` resolves identically on the `cdp` and `container` lanes.

**Secret hygiene — what is and isn't scrubbed.** Env-resolved secret values are redacted from everything Heimdall serializes: `report.json` (including a `${env.TOKEN}` spliced into a request URL, and any error message that embeds it) and the terminal/HTML/JUnit reports. They are **not** scrubbed from the binary Playwright **evidence files** — the per-case `network.har` (which retains request headers and URLs even though response bodies are omitted) and `trace.zip` can contain the secret on the wire, exactly as an injected `storageState` holds live cookies. **Treat a run's output directory as secret-bearing** — keep it gitignored and don't share it.

**Extra redaction config.** Beyond the per-run `${env.X}` secrets, a plan can declare a `redaction` block to scrub SUT-returned PII or sensitive headers from the report:

```jsonc
{
  "name": "my-plan",
  "redaction": {
    "headers": ["authorization", "set-cookie"],   // response header values blanked (case-insensitive)
    "patterns": ["[0-9]{16}", "\\b\\w+@\\w+\\.\\w+\\b"]  // RegExp sources; any match in report strings is blanked
  },
  "cases": [ /* … */ ]
}
```

The spec is folded with the env secrets and applied to the serialized report (`report.json` + terminal/HTML/JUnit) **and the on-disk `network.har`** (scrubbed in the driver via `scrubHar` — both the `cdp` lane and the `container` lane, which threads the spec into its inner run). The binary `trace.zip`/video are **not** scrubbed — treat the run's output dir as sensitive. Embedders can supply the same spec out-of-band via `runPlan(plan, { redaction })`, which is merged with `plan.redaction`. A malformed regex pattern is ignored (fail-safe) rather than aborting the run.

A malformed config (bad JSON or an unknown key) fails loudly; a missing config is simply ignored.

---

## How it pairs with agent test workflows

Heimdall is designed to be embedded, not just run by hand. The public API (`src/index.ts`) exports the schemas, the runner, the oracle evaluator, and the reporter:

```ts
import { runPlan, parsePlan, evaluateOracles, formatReport, exitCodeFor } from "@antreas/heimdall";

const plan = parsePlan(JSON.parse(rawPlanJson));
const report = await runPlan(plan, {
  outDir: "heimdall-runs/latest",
  allowRisk: false,
  headed: false,
  concurrency: 4,
});
process.exit(exitCodeFor(report));
```

An agent workflow can author a thorough plan and then execute it through the most faithful available browser lane. The division of labour is clean:

- The skill authors plans against the schema (`heimdall schema` gives it the contract), assigning each case a `dimension`, `risk`, `priority`, and the right `driver`.
- Heimdall runs the **self-drivable lanes** (`cdp`, `container`) in parallel and emits a structured `RunReport` with screenshots, HAR, console errors, and captured responses as evidence.
- Cases that demand the highest fidelity — your real logged-in session — are tagged `driver: extension`. Heimdall blocks those honestly, the calling workflow extracts them with [`heimdall extensions`](#heimdall-extensions-plan), drives them through real-Chrome browser tools, and folds the outcomes back in.

**Per-oracle results — auditable greens.** Each `Result` carries an optional `oracleResults[]` — one entry per oracle with its `kind` (the `assert` discriminant), `passed`, and a human-readable `detail`. A passing case no longer reports only `observed: "all N oracle(s) satisfied"`; the verifier can see *which* oracle fired and on what, detecting a hollow green from the report alone without a re-run. (Older reports omit the field, so it is optional.)

**Merging the two lanes into one report.** Embedders close the loop programmatically: the skill collects the externally-driven `extension` outcomes as a `Result[]` and passes them to `runPlan(plan, { externalResults })` (the CLI equivalent is `--merge-results`). The runner replaces each matching (blocked) id verbatim and appends any extra ids, then sorts back into plan order — so a live test ends with a single `RunReport`, and `exitCodeFor` gates on the merged truth:

```ts
const report = await runPlan(plan, {
  outDir: "heimdall-runs/latest",
  externalResults,   // Result[] the agent produced for the extension cases
  redaction: { headers: ["set-cookie"] },  // optional extra scrubbing, merged with plan.redaction
});
```

The result: one plan format, one evidence model, with each case routed to the driver that can run it most faithfully — and a report that never lies about what was actually exercised.

---

## License

MIT © Antreas Antoniou
