# Repo Monitor Templates

Two GitHub Actions workflow templates that watch a GitHub repo of your choosing and post a Discord notification when something new happens. Pick the one that fits what you want to track:

- **`repo-release-monitor-template.yml`** — notifies on new GitHub Releases (version tags, release notes).
- **`repo-commit-monitor-template.yml`** — notifies on new commits (any commit pushed to the default branch).

You can use either one, or both, on the same repo.

## Setup

1. **Click Add file** → Create new file. In the filename box, enter: `.github/workflows/` then name the tracker whatever you want.

2. **Copy the file** into `.github/workflows/` in any repo you control — it does **not** have to be the repo you're monitoring. This repo just needs to be somewhere GitHub Actions can run and commit a small state file.

3. **Rename it to something unique.** At the top of the file, change:
   - `name:` — the workflow's display name in the Actions tab.
   - `concurrency: group:` — must be unique per monitor. If you add a second monitor to the same repo and forget to change this, the two will cancel each other out.

4. **Fill in the config block** under `jobs: monitor: env:`:

   | Variable | Meaning |
   |---|---|
   | `REPO` | The `owner/repo` you want to watch, e.g. `ChrisTitusTech/winutil` |
   | `PROJECT_NAME` | Display name used in the Discord message title, e.g. "winutil" |
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

## Notes

- By default, the release monitor skips draft releases and prereleases, matching GitHub's own definition of "latest release." If you want to be notified about prereleases too, that's a small code change inside the `get_recent_releases` function (drop the `prerelease` filter) — not exposed as a config variable here.
- Unauthenticated GitHub API access is capped at 60 requests/hour; this template authenticates with `GITHUB_TOKEN` automatically so that isn't a concern unless you're running a very large number of monitors from one repo.
