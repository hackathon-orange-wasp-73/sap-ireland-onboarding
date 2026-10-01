# header-watch

An autonomous agent that watches only the **header/navigation bar** of the SAP
Dublin onboarding site for structural or visual changes across deploys — not a
whole-page diff, and not a deterministic pixel-diff script.

It is adapted from the [GitHub Copilot SDK
workshop](https://github.com/jeffrey-groneberg/ghcp-sdk-workshop) (`watch.py` /
`workshop_support.py`): the model is given a plain-English goal and a narrow
set of tools, and it decides what "changed" means, instead of a `while` loop
or a pixel-diff algorithm.

## How it works

Each run:

1. Navigates to the configured URL with the official `@playwright/mcp` server.
2. Uses `browser_snapshot` to locate the header/nav landmark and its exact
   element `target` ref.
3. Takes an **element-scoped** screenshot of only that header/nav element
   (`browser_take_screenshot` with `element` + `target`, never `fullPage`)
   into `after.png`.
4. Inspects `after.png` (and the previous run's `before.png`, if any) with the
   built-in `view` tool — the model sees the actual pixels.
5. Calls the custom `finish_run` tool with `status` (`baseline` / `unchanged`
   / `changed`), a `summary`, and a list of concrete `changes`.

A `ToolGuard` only allows: navigating to the assigned URL, screenshotting the
header element into the assigned path, and viewing the two assigned images.
It refuses to finalize until both images have actually been read, and it
rejects `fullPage` screenshots (this tool only ever compares the header, not
the full page).

Each run produces a portable folder:

```text
before.png     # previous run's header screenshot (absent on the first/baseline run)
after.png      # this run's header screenshot
report.md      # only written when status == "changed"
outcome.json   # the staged finish_run result
snapshot.json  # this run's identity/hash record
```

`latest.json` (one level up, shared across runs for the same URL/config) only
advances on a **complete** run — an interrupted or incomplete run does not
move the baseline forward.

## Local usage

From this directory, with Python 3.11+ and Node 22+:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
npm install              # no package-lock.json is committed; see note below
npm run browser:install  # playwright install --with-deps chromium
python -m copilot download-runtime

copilot login --device-code   # sign in with an account that has Copilot CLI access
python header_watch.py --list-models
python header_watch.py                                  # watches the default onboarding site URL
python header_watch.py https://example.com/              # or watch any other URL
python header_watch.py --model claude-haiku-4.5 --timeout 240
```

### CLI flags

| Flag | Default | Purpose |
| --- | --- | --- |
| `url` (positional, optional) | `https://hackathon-orange-wasp-73.github.io/sap-ireland-onboarding/` | The page to watch. |
| `--model` | `claude-haiku-4.5` | A PNG-capable model that supports two-image comparison. |
| `--list-models` | — | List compatible models without running the agent. |
| `--output` | `.header-watch` | Local artifacts root. |
| `--timeout` | `180` | Overall run limit in seconds. |

### Running against a local demo instead of the public Pages URL

The repo root has its own `package.json` with a `demo` script (matching the
[workshop's](https://github.com/jeffrey-groneberg/ghcp-sdk-workshop) pattern)
that serves the site's static files on a fixed local port:

```bash
# Terminal 1, from the repo root:
npm run demo
# -> serves http://127.0.0.1:8000/ (Python's http.server, bound to 127.0.0.1)

# Terminal 2, from tools/header-watch/:
python header_watch.py http://127.0.0.1:8000/
```

This lets you iterate on header/nav markup locally and re-run the watcher
without waiting on a Pages deploy, and without depending on the public URL
being reachable. Stop the server with Ctrl-C in terminal 1 when done.

Run the same command again to compare against the previous run; each
successful run becomes the next baseline.

> **No `package-lock.json` is committed.** The workshop's identity key reads
> the resolved Playwright version from a lockfile; this tool instead reads
> the pinned `@playwright/mcp` version string directly from `package.json`,
> so `npm install` (not `npm ci`) is used both locally and in CI.

## CI: `.github/workflows/header-watch.yml`

Runs daily (06:00 UTC) and on-demand via `workflow_dispatch` (with optional
`url`/`model` inputs). Each run:

1. Installs Node 22, Python 3.12, the pinned npm/pip deps, and Chromium.
2. Runs the watcher against the Pages URL (or the dispatched `url` input).
3. Uploads the run's output folder (`report.md` + screenshots + JSON) as a
   build artifact, named `header-watch-run-<run id>`, retained 90 days.
4. **Fails the job** if the agent reports `status: changed` — a failed
   required/scheduled run is the simplest way to surface a flagged header
   change, without needing issue-tracking logic in the workflow itself. Open
   the run's artifact to read `report.md` for what changed.

### Required secret: `COPILOT_SDK_TOKEN`

The workflow needs an authenticated Copilot SDK session, which cannot use
interactive device-code login in CI. The workflow already references
`secrets.COPILOT_SDK_TOKEN` as a placeholder — it just needs the secret's
**value** provisioned once:

1. Pick (or create) a GitHub account that has GitHub Copilot access enabled
   (an individual Copilot subscription, or org/enterprise-assigned seat) and
   that can run the Copilot CLI (`copilot login --device-code` succeeds for
   that account locally).
2. Create a **fine-grained personal access token** for that account:
   [github.com/settings/personal-access-tokens/new](https://github.com/settings/personal-access-tokens/new).
   No special repository permissions are required beyond the defaults — this
   token is only used to authenticate the Copilot SDK/CLI session, not to
   access repository contents. Give it a descriptive name (e.g.
   `header-watch-copilot-sdk`) and a reasonable expiry, then rotate it before
   it expires.
3. In this repository, go to **Settings → Secrets and variables → Actions →
   New repository secret**, name it `COPILOT_SDK_TOKEN`, and paste the PAT
   value.
4. Re-run the workflow (`Actions → Header watch → Run workflow`) to confirm
   authentication succeeds.

Without this secret, the workflow fails at the "Run header watch" step with
an authentication error — everything else (Node/Python/Chromium install,
artifact upload) runs independently of it.

### GitHub Pages visibility

The repository is public, and its GitHub Pages site is served at the standard
project-Pages URL: `https://hackathon-orange-wasp-73.github.io/sap-ireland-onboarding/`,
which is what this tool watches by default. (An earlier private-repo state
served the site from a different, auth-gated subdomain
(`legendary-tribble-ny53n3v.pages.github.io`); that URL now 404s.)
