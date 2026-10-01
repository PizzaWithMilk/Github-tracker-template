# Repo Monitor

[📖 Visual Tutorial](Tutorial.md)

## What this actually does

This is one ready-to-use GitHub Actions workflow. It checks one or more GitHub repositories for **new releases** and/or **new commits**, then posts Discord notifications when something changes.

How it behaves — which repos, releases vs commits, which Discord channel, who gets pinged, stats dashboard, incident log, and formatting options — is controlled by **`config/repos.yml`**. You do not edit the workflow file to change those settings.

One workflow can watch any mix of repos at once: some to a normal text channel, some to a Forum channel, some releases-only, some commits-only, each configured independently.

### How runs are triggered

| Event | What happens |
|--------|----------------|
| **`workflow_dispatch`** (manual **Run workflow**, or an external cron that calls it) | **Live monitor** — checks GitHub, posts Discord notifications, updates state and the optional stats dashboard. |
| **`push` / `pull_request`** (except changes that only touch `.github/monitor-state/`) | **Validation only** — parses `config/repos.yml` and the workflow. Does **not** query monitored repos for updates as a full monitor cycle; does **not** post normal update notifications. Config errors can still update the stats dashboard / incident log if those are enabled. |

This copy of the workflow does **not** ship with a built-in `schedule:` cron. Live checks are started with **Run workflow** or an external scheduler that triggers `workflow_dispatch`. See [External cron with cron-job.org](#external-cron-with-cron-joborg) below. (You can also add a `schedule:` block under `on:` if you prefer GitHub’s own cron.)

State commits under `.github/monitor-state/` are ignored by push/PR triggers so the monitor cannot re-trigger itself in a loop.

## A few words explained

- **Repo (repository):** a project on GitHub.
- **Workflow:** the automation file under `.github/workflows/`.
- **Config file:** `config/repos.yml` — lists repos, webhooks, pings, and options.
- **Webhook:** a Discord channel URL. Anyone with the URL can post to that channel; keep it private if the hosting repo is public.
- **State directory:** `.github/monitor-state/` by default — bookmarks, failure tracking, stats, and delivery resume files. You normally never edit these by hand.
- **Stats dashboard:** optional single Discord embed (edited in place) summarizing health, this run, lifetime totals, and per-repo activity.
- **Incident log:** optional separate channel where each new error/rate-limit/fatal posts a **new** message (with ping). Not edited in place.

## Text channels, Forum channels, releases, and commits

Per repo in `config/repos.yml`:

- `type:` — `normal` (regular text channel) or `forum` (Forum channel).
- `monitor:` — `commits` or `releases` (default **`releases`**).

**Regular text channel:** continuous chat stream → `type: normal`.  
**Forum channel:** grid of posts with **New Post** → `type: forum`.

`type` must match the kind of channel the effective webhook was created on.

## Quick start

1. Put the workflow under `.github/workflows/` (any `.yml` name is fine). Keep `name:` and concurrency groups unique if you run multiple copies in one repo.
2. Create `config/repos.yml` with global ping/webhook defaults and a `repos:` list.
3. Leave `CONFIG_FILE`, `STATE_DIR`, and `NOTIFY_ON_INITIAL_RUN` unless you have a reason to change them.
4. Commit both files. Use **Actions → Run workflow** to do a live check, or set up an [external cron](#external-cron-with-cron-joborg).

No GitHub secret is required for Discord webhooks — they live in `repos.yml` as plain text (see [Security](#security--permissions-notes)). A **personal access token** is only needed if you use an external cron service to trigger the workflow.

---

## Full setup

### Step 1: Install the workflow

The workflow must live in a GitHub repo under `.github/workflows/`. It does **not** have to be the same repo you monitor.

**On GitHub.com:** **Add file → Create new file**, path like `.github/workflows/repo-monitor.yml`, paste the workflow, commit.

**Locally:** `mkdir -p .github/workflows`, save the file, commit, and push.

### Step 2: Create `config/repos.yml`

```yaml
# copy paste all of this if you want.
ping_mode: user
ping_id: "YOUR_DISCORD_USER_ID_HERE"
webhook: "YOUR_DISCORD_WEBHOOK_URL_HERE"

# Optional: stats dashboard (single embed, edited in place)
stats_enabled: true
stats_webhook: "YOUR_STATS_CHANNEL_WEBHOOK"

# Optional but recommended: error log channel (new message per incident, with ping)
incident_webhook: "YOUR_INCIDENT_CHANNEL_WEBHOOK"
# incident_ping_mode / incident_ping_id default to global ping_mode / ping_id

repos:
  - repo: OWNER/REPO
    type: normal
```

Minimum per repo: `repo:` first (`OWNER/REPO`), and `type:` (`normal` or `forum`).

### Step 3: Workflow env settings

Under `jobs.monitor.env`: recommended to leave default.

| Setting | Meaning |
|--------|---------|
| `CONFIG_FILE` | Path to config (default `config/repos.yml`). |
| `STATE_DIR` | Bookmark/state folder (default `.github/monitor-state`). |
| `NOTIFY_ON_INITIAL_RUN` | `"true"` (default) or `"false"` — whether the first seen release/commit for a repo posts to Discord or only seeds state. |

### Step 4: Discord webhooks

1. Channel settings → **Integrations → Webhooks → New Webhook** → **Copy Webhook URL**.
2. For Forum repos, create the webhook **on the Forum channel itself**.
3. Paste into top-level `webhook:` and/or per-repo `webhook:`.
4. Optionally set `stats_webhook` and `incident_webhook` to other channels.

`GITHUB_TOKEN` is provided automatically by GitHub Actions.

### Step 5: Run and verify

**Actions →** this workflow → **Run workflow**. Open the log:

- Healthy idle: `No new release.` / `No new commit.`
- Activity: lines about new releases/commits and Discord posts
- Validation-only push/PR: `mode: VALIDATION ONLY` and a success notice if config is valid

Green check = success. Red X = failure (see [Troubleshooting](#troubleshooting)).

---

## External cron with cron-job.org

GitHub’s built-in `schedule:` can be delayed or disabled after long inactivity. An external cron (such as [cron-job.org](https://cron-job.org)) calls GitHub’s API on a timer and starts a **live** monitor run via `workflow_dispatch`.

### 1. Create a GitHub token

You need a token that can trigger workflows on the **hosting** repo (the one that contains the workflow file).

**Fine-grained personal access token (recommended):**

1. GitHub → **Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token**.
2. Resource owner: your user or org that owns the hosting repo.
3. Repository access: **Only select repositories** → pick the hosting repo.
4. Permissions → **Repository permissions**:
   - **Actions:** Read and write  
   - **Contents:** Read-only is enough for dispatch; Read and write is fine if you already use it elsewhere  
5. Generate and **copy the token once** (you will not see it again).

**Classic PAT (alternative):** scope at least `repo` (private hosting repo) or `public_repo` (public hosting repo). Workflow dispatch also needs the Actions permission that classic `repo` includes.

Treat this token like a password. Do **not** put it in `repos.yml` or commit it to the repo.

### 2. Find the dispatch URL

```text
https://api.github.com/repos/OWNER/HOSTING-REPO/actions/workflows/WORKFLOW_FILENAME/dispatches
```

Replace:

| Part | Example |
|------|---------|
| `OWNER` | Your GitHub user or org |
| `HOSTING-REPO` | Repo that contains `.github/workflows/...` |
| `WORKFLOW_FILENAME` | Exact file name only, e.g. `repo-monitor.yml` (not the full path) |

Example:

```text
https://api.github.com/repos/myuser/my-monitor-host/actions/workflows/repo-monitor.yml/dispatches
```

You can also use the numeric workflow id from  
`https://api.github.com/repos/OWNER/HOSTING-REPO/actions/workflows`  
instead of the filename.

### 3. Create the job on cron-job.org

1. Sign up / log in at [cron-job.org](https://cron-job.org).
2. **Cronjobs → Create cronjob**.
3. **Title:** e.g. `Repo Monitor`.
4. **URL:** the dispatch URL from step 2.
5. **Schedule:** e.g. every 5 minutes (`*/5 * * * *`) or whatever interval you want. Match this roughly to what you want the stats “Next check” feel to be; the workflow itself uses a 5-minute interval only as a display hint for the dashboard.
6. **Request method:** **POST**.
7. **Request headers** (add all three):

   | Header | Value |
   |--------|--------|
   | `Authorization` | `Bearer YOUR_GITHUB_TOKEN` |
   | `Accept` | `application/vnd.github+json` |
   | `Content-Type` | `application/json` |
   | `X-GitHub-Api-Version` | `2022-11-28` (optional but recommended) |

8. **Request body** (raw JSON):

   ```json
   {"ref":"main"}
   ```

   Use your default branch name if it is not `main` (e.g. `"master"`).

9. Save and enable the cronjob.

### 4. Test it

- Use cron-job.org’s **“Run now”** (or equivalent) once.
- On GitHub: **Actions** → your workflow → a new run should appear with event **`workflow_dispatch`**.
- Open the log and confirm `mode: LIVE MONITOR` (not validation-only).

If you get **401/403**, the token is missing, expired, or lacks Actions write on that repo.  
If you get **404**, check `OWNER`, `HOSTING-REPO`, and the workflow **filename**.  
If you get **422**, the `ref` branch name is wrong or the workflow file is not on that branch.

### 5. Security tips

- Prefer a **fine-grained** token limited to the single hosting repository.
- Store the token only in cron-job.org’s job settings (HTTPS), not in git.
- If the token leaks, revoke it on GitHub immediately and create a new one.
- cron-job.org will send your token on every tick — use a dedicated token you can rotate without affecting other tools.

### Alternative: GitHub `schedule:` instead (not recommended)

If you would rather not use an external service, add under `on:` in the workflow file:

```yaml
on:
  schedule:
    - cron: "*/5 * * * *"   # every 5 minutes (GitHub may delay under load)
  workflow_dispatch:
  push:
    paths-ignore:
      - ".github/monitor-state/**"
  pull_request:
    paths-ignore:
      - ".github/monitor-state/**"
```

Scheduled runs still need recent activity on the hosting repo or GitHub may disable them after ~60 days of no commits.

---

## Config validation

Push and pull_request runs (other than pure state commits) use the **same job** in **validation-only** mode:

- Parses and validates `config/repos.yml`
- Does **not** run the full live monitor cycle (no normal update posts, no routine state bookmarks for “what’s new”)
- Does **not** cancel in-progress validation runs (`cancel-in-progress: false`)
- Live monitor uses concurrency group `repo-monitor`; validation uses `repo-monitor-validation`

If config is invalid, the job fails and the log shows `::error::` with the problem (often including a line number).

**Note:** If stats/incidents are enabled, a **fatal config error** can still update the stats embed and post an incident message, then persist state so lifetime error counts are not lost on the next live run.

---

## Configuring `repos.yml`

### Global settings

| Setting | Default | Meaning |
|--------|---------|---------|
| `ping_mode` | `user` | `user` \| `role` \| `everyone` \| `none` |
| `ping_id` | *(none)* | Required when `ping_mode` is `user` or `role` |
| `webhook` | *(none)* | Default Discord webhook; required unless every repo sets its own |
| `github_api_base_url` | `https://api.github.com` | GitHub Enterprise Server API base if needed |
| `commit_accent_color` | `5865F2` | Accent for commit notifications |
| `release_accent_color` | `FEE75C` | Accent for release notifications |
| `stats_enabled` | `false` | Enable the stats dashboard embed |
| `stats_webhook` | *(none)* | Webhook for the stats message (required if `stats_enabled: true`) |
| `incident_webhook` | *(none)* | If set, posts a new incident message per error/rate-limit/fatal |
| `incident_ping_mode` | *(global `ping_mode`)* | Ping mode for incident messages |
| `incident_ping_id` | *(global `ping_id`)* | Ping id for incident messages |

Colors: `#RRGGBB`, `RRGGBB`, `0xRRGGBB`, or decimal.

**Deprecated (ignored safely):** `inactivity_accent_color`.

**Not supported:** `digest:` (unknown key → validation failure).

### Per-repo settings

| Field | Default | Meaning |
|--------|---------|---------|
| `repo` | *required* | `OWNER/REPO` (must be first in the block) |
| `type` | *required* | `normal` or `forum` |
| `monitor` | `releases` | `commits` or `releases` |
| `enabled` | `true` | `false` parks the repo (skipped; state kept). Missing field = enabled |
| `ping` | inherits | Overrides `ping_mode` |
| `ping_id` | inherits | Overrides `ping_id` |
| `webhook` | inherits | Per-repo channel |
| `github_api_base_url` | inherits | Per-repo API base |
| `accent_color` | inherits | Per-repo accent |
| `include_release_assets` | `true` | List release assets with **name + human-readable size** when available |
| `short_summary` | `false` | Shorten long release notes |
| `summary_lines` | `8` | Lines kept when `short_summary` is true |
| `tags` | *(none)* | Forum tag IDs (max 5 per post) |

Display name in Discord is the repo name after `/` (e.g. `winutil`). On the **stats** dashboard, short names are used unless two monitored repos share the same name (then `owner/repo` is shown).

### Example

```yaml
ping_mode: user
ping_id: "YOUR_DISCORD_USER_ID_HERE"
webhook: "https://discord.com/api/webhooks/DEFAULT"

stats_enabled: true
stats_webhook: "https://discord.com/api/webhooks/STATS"

incident_webhook: "https://discord.com/api/webhooks/INCIDENTS"

repos:
  - repo: ChrisTitusTech/winutil
    type: normal

  - repo: torvalds/linux
    type: normal
    monitor: releases

  - repo: someuser/some-other-repo
    type: normal
    monitor: commits

  - repo: someorg/chatty-repo
    type: forum
    ping: role
    ping_id: "123456789012345678"
    short_summary: true
    summary_lines: 6
    tags:
      - "111111111111111111"

  - repo: owner/parked-for-now
    type: normal
    enabled: false
```

Notes:

- Pings are **per notification**, not once per whole run.
- Each repo (and commit vs release stream) has independent bookmark state.
- `enabled: false` skips API calls and Discord for that repo; state is left as-is.
- One broken repo does not stop the others; the overall job still fails so you notice.
- Invalid `repos.yml` fails the whole run before monitoring.

---

## Stats dashboard

When `stats_enabled: true` and `stats_webhook` is set:

- One Discord **embed** is created and then **edited in place** each live run (and on some failure paths).
- Layout (single embed):
  - **Repo Monitor** — status, last/next check, repos checked, GitHub API budget when known
  - **This Run** — commits/releases/notifications this run
  - **Lifetime** — runs, commits, releases, notifications, errors (errors only increase)
  - **Repository Activity** — inline fields per repo (short name unless collision)
- **Incidents are not shown in this embed** — use `incident_webhook` for the error log.
- No ping on stats updates (`allowed_mentions` empty).
- Lifetime **Errors** is a durable high-water mark (git state + `_lifetime.json` + Actions cache + in-state high-water). Successful runs must not lower it.
- **Next check** countdown is not reset by error-driven stats edits mid-cycle.

Last release/commit lines use markdown links to GitHub when URLs are known.

---

## Incident log

When `incident_webhook` is set, each new problem posts a **new** message (never edits an old one), with ping:

| Kind | Typical title | When |
|------|----------------|------|
| Error | Repo Monitor · Error | Per-repo monitor failure |
| Rate limit | Repo Monitor · Rate Limit | Primary GitHub API budget exhausted |
| Fatal | Repo Monitor · Fatal Error | Config/startup/unhandled failure |

Optional fields in the body: repo, when, monitor kind, HTTP status, reset time, scope.

This is separate from **failure alerts** on the repo’s own webhook (Components-style “Repo Monitor Alert” after permanent errors or repeated transient failures).

---

## Edited releases

For **release** monitoring, the monitor fingerprints the **newest** release (body + assets metadata).

If that same tag’s body or assets change later:

- The previous Discord notification is **edited** when possible
- Title uses `(edited)` and a note that the release was edited on GitHub
- Ping is sent again

Only the **latest** release per repo is tracked for edits. Older tags are not kept for edit detection after a newer release appears.

---

## Routing to different Discord channels

```yaml
webhook: "https://discord.com/api/webhooks/DEFAULT"

repos:
  - repo: owner/project-one
    type: normal

  - repo: owner/project-two
    type: forum
    webhook: "https://discord.com/api/webhooks/FORUM_CHANNEL"

  - repo: owner/project-three
    type: normal
    webhook: "https://discord.com/api/webhooks/OTHER_SERVER"
```

`type` must match each webhook’s channel type.

---

## Choosing who gets pinged

| `ping_mode` | Behavior |
|-------------|----------|
| `user` | Mentions the user id in `ping_id` |
| `role` | Mentions the role id in `ping_id` |
| `everyone` | `@everyone` |
| `none` | No mention |

Per-repo `ping` / `ping_id` override globals. Incident messages use `incident_ping_*` (or the global ping settings).

---

## First run behavior

`NOTIFY_ON_INITIAL_RUN`:

- `"true"` (default) — first discovery of a release/commit stream can notify Discord.
- `"false"` — first discovery only writes bookmarks; later changes notify.

---

## Release notes options

- **`include_release_assets: true`** (default) — lists assets with download links and human-readable sizes when GitHub provides size.
- **`short_summary: true`** — truncates long notes to `summary_lines` (default 8).

Issue/PR references like `#123` and full GitHub URLs in notes are turned into clickable Discord links. Enterprise `github_api_base_url` links point at that host.

---

## Forum channel behavior

- Each new release or commit becomes its **own** Forum post (thread).
- Titles look like `{Project} Updated! — {tag or version}` (max 100 characters).
- Long content continues as replies in the same thread.
- Optional `tags:` list (numeric Forum tag IDs, max 5).
- Interrupted multi-part deliveries can resume via state files; mismatched content or a deleted thread starts clean.

---

## Reliability

- Network errors: retried with backoff.
- HTTP 429 / secondary limits: retried using `Retry-After` when present.
- Primary GitHub rate limit (`remaining: 0`): wait briefly if reset is soon; otherwise stop remaining repos for that run.
- 5xx / 408: limited retries.
- Discord sends spaced by ~0.5s.
- Permanent GitHub **401 / 403 / 404** and Discord **401 / 403 / 404**: not endlessly retried; permanent failure alerts once per issue.
- Transient failures: alert on the **repo webhook** after **3** consecutive failed runs for that repo/kind.
- Primary rate-limit exhaustion: separate alert after **3** consecutive affected runs.
- Monitor concurrency group does not cancel in-progress live runs.

---

## Troubleshooting

**Validation didn’t run when I edited config.**  
Push/PR should run for any change except pure `.github/monitor-state/**` updates. Confirm the workflow on the default branch includes those triggers, and that Actions is enabled. Logs should show `mode: VALIDATION ONLY`.

**Config file not found.**  
`CONFIG_FILE` must match the real path (default `config/repos.yml`).

**Immediate failure mentioning invalid type / monitor / ping / webhook.**  
Fix `repos.yml` (message usually includes a line number).

**Stats Errors went up on a config error, then dropped after a good run.**  
Older builds only persisted state on `workflow_dispatch`. Current builds save cache + git state on validation failures too, and treat lifetime Errors as a high-water mark. After updating the workflow, a good run should keep the higher count.

**Repo Monitor failure alert on the repo channel.**  
Permanent access errors alert once; transient failures after 3 consecutive fails. A successful run clears the matching failure state.

**One repo fails, others post.**  
Expected. Job is still red overall; log shows `Error while monitoring OWNER/REPO: ...`.

**Nothing posted, no error.**  
Usually “nothing new.” Check log for `No new release.` / `No new commit.`, spelling of `repo:`, `monitor:`, and `enabled:`.

**Forum post fails mentioning `thread_name`.**  
Webhook was almost certainly created on the wrong channel type.

**Save monitor state / git push fails.**  
Branch protection may block `github-actions[bot]`. Allow the bot or host the workflow in an unprotected repo.

**Re-test a notification.**  
Edit or clear the relevant bookmark under `STATE_DIR` (names like `Owner__Repo_release.txt` / `_commit.txt`).

---

## Security & permissions

- Workflow permission: **contents: write** (checkout + push state). Not issues/PRs/other repos.
- Token only covers the **hosting** repo unless you add a PAT for other private repos.
- **Webhook URLs, ping ids, and repo lists are plain text in `repos.yml`.** Prefer a private hosting repo if that matters.
- Prefer not committing real webhooks to public forks.

---

## FAQ

**Private repos?**  
Same repo as the workflow: usually fine with `GITHUB_TOKEN`. A *different* private repo needs a PAT with read access, stored as a secret and wired into the script.

**Multiple channels?**  
Per-repo `webhook:` overrides. Optional separate `stats_webhook` and `incident_webhook`.

**Multiple workflow copies?**  
Unique `name`, concurrency groups, and preferably distinct `STATE_DIR` / config files.

**Draft / prerelease releases?**  
Not notified — follows GitHub’s “latest release” rules (no drafts/prereleases).

**Every commit?**  
No. Bot/merge/empty-user-facing commits are filtered; default branch only.

**Rate limits?**  
Designed for normal authenticated use; primary limit is handled explicitly. Stats embed can show remaining/limit/reset when headers are observed.

**Built-in schedule?**  
This workflow file is driven by `workflow_dispatch` (+ push/PR validation). Use [cron-job.org](#external-cron-with-cron-joborg) (or similar) for a reliable external timer, or add a GitHub `schedule:` cron under `on:` if you prefer.

---

## Changelog highlights (vs older README)

- Single job: live monitor **or** validation-only (not a separate `validate` job).
- Push/PR triggers use `paths-ignore` for monitor state (not a narrow `paths` allowlist).
- Optional **stats dashboard** (embed) and **incident log** channel.
- Per-repo **`enabled`** (default true).
- **Edited-release** detection for the newest tag only.
- Release **asset sizes** in notifications when available.
- Lifetime **Errors** high-water persistence across git/cache/state.
- `inactivity_accent_color` deprecated; no built-in 59-day inactivity Discord reminder in this workflow revision.
- No `digest` mode.
