# Repo Monitor Templates

Two GitHub Actions workflow templates that watch a GitHub repo of your choosing and post a Discord notification when something new happens. Pick the one that fits what you want to track:

- **`repo-release-monitor-template.yml`** — notifies on new GitHub Releases (version tags, release notes).
- **`repo-commit-monitor-template.yml`** — notifies on new commits (any commit pushed to the default branch).

You can use either one, or both, on the same repo.

## Quick start

1. Change the `name:` field and the `concurrency: group:` value at the top of the file so it doesn't collide with any other monitor workflow in the same repo.
2. Fill in the `env:` block under the `monitor` job (`REPO`, `PROJECT_NAME`, `STATE_FILE`, `PING_MODE`, `PING_ID`).
3. Add a `DISCORD_WEBHOOK` repository secret (repo **Settings → Secrets and variables → Actions → New repository secret**).
4. Commit the file to `.github/workflows/` in any repo you control — it doesn't have to be the repo you're monitoring.

The rest of this README walks through each of those in more detail.

## Setup

1. **Copy the file** into `.github/workflows/` in any repo you control — it does **not** have to be the repo you're monitoring. This repo just needs to be somewhere GitHub Actions can run and commit a small state file.

   If that folder doesn't exist yet in your repo, you'll need to create it. Two ways:

   - **On GitHub.com:** open your repo → **Add file → Create new file**. In the filename box, type the full path including the folders, e.g. `.github/workflows/release-monitor.yml` — GitHub creates `.github/` and `workflows/` automatically as soon as you type the `/` characters, you don't create folders separately. Paste in the template's contents, then commit.
   - **On your own machine:** inside a local clone of the repo, run `mkdir -p .github/workflows`, save the template file into that folder (keep the `.yml` extension), then `git add .github/workflows/<filename>.yml`, `git commit -m "Add repo monitor"`, and `git push`.

   Either way, the exact filename inside `workflows/` doesn't matter (e.g. `release-monitor.yml`, `otd-monitor.yml`) — only that it ends in `.yml` and lives in that folder.

2. **Rename it to something unique.** At the top of the file, change:
   - `name:` — the workflow's display name in the Actions tab.
   - `concurrency: group:` — must be unique per monitor. If you add a second monitor to the same repo and forget to change this, the two will cancel each other out.

3. **Fill in the config block** under `jobs: monitor: env:`:

   | Variable | Meaning |
   |---|---|
   | `REPO` | The `owner/repo` you want to watch, e.g. `OpenTabletDriver/OpenTabletDriver` |
   | `PROJECT_NAME` | Display name used in the Discord message title, e.g. "OpenTabletDriver" |
   | `STATE_FILE` | Filename used to remember the last commit/release seen. Only needs changing if you add more than one monitor to the same repo — give each one a different filename. |
   | `PING_MODE` | Who gets pinged — see below |
   | `PING_ID` | A Discord user or role ID — see below |

4. **Add the webhook secret.** In the repo you copied the file into: **Settings → Secrets and variables → Actions → New repository secret**, name it `DISCORD_WEBHOOK`, and paste in your Discord webhook URL (Discord channel → Edit Channel → Integrations → Webhooks).

   `GITHUB_TOKEN` needs no setup — GitHub provides it automatically, and the workflow uses it just to get a higher GitHub API rate limit.

5. **Commit and push.** The workflow runs automatically every 5 minutes. To test it immediately without waiting, go to the **Actions** tab → select the workflow → **Run workflow**.

## Choosing who gets pinged

Set `PING_MODE` to one of:

| `PING_MODE` | What it does | Also set `PING_ID` to... |
|---|---|---|
| `user` (default) | Pings one specific person | Their Discord **user ID** |
| `role` | Pings everyone with a specific role | The **role ID** |
| `everyone` | Pings `@everyone` in the channel | *(not needed)* |
| `none` | Posts with no ping at all | *(not needed)* |

**Getting an ID:** In Discord, turn on Developer Mode (**User Settings → Advanced → Developer Mode**), then right-click a person or a role and choose **Copy User ID** / **Copy Role ID**.

**About `everyone`:** this will notify every member with access to that channel, not just active ones — it's meant for a channel where people already expect update pings (like a dedicated `#releases` channel), not a general chat. Discord also requires this to be explicitly allowed rather than just typing "@everyone" in a message, which is why `PING_MODE=everyone` exists as its own option rather than just being part of `role`/`user`.

You don't need to touch any code for any of this — `PING_MODE` and `PING_ID` are the only two things to set.

## How it behaves

Both templates share the same design as the monitors this was built from:

- Polls every 5 minutes, and catches up on everything missed since the last check (not just the single newest item) — up to 10 items per run, with a note if more landed than that.
- The ping, title, and author/link info are sent together in one message along with as much of the notes/commit message as fits. Long ones spill into "(continued)" follow-up messages, always breaking on whole lines (bullet points, links, etc. are never cut mid-word).
- Retries network calls a few times before giving up, and won't let two runs overlap.
- Bare `#123` references *and* full bare PR/issue URLs are both converted into real clickable links before sending. Some projects mix both styles within one release (part hand-written, part auto-generated) — GitHub renders either form as a link on its own pages, but Discord does neither on its own, so both get normalized.

## Notes

- By default, the release monitor skips draft releases and prereleases, matching GitHub's own definition of "latest release." If you want to be notified about prereleases too, that's a small code change inside the `get_recent_releases` function (drop the `prerelease` filter) — not exposed as a config variable here.
- Unauthenticated GitHub API access is capped at 60 requests/hour; this template authenticates with `GITHUB_TOKEN` automatically so that isn't a concern unless you're running a very large number of monitors from one repo.
