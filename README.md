# Repo Monitor Templates

GitHub Actions workflow templates that watch a GitHub repo of your choosing and post a Discord notification when something new happens. There are four templates — pick based on what you want to track and which kind of Discord channel you're posting into:

|  | Notifies on | Regular text channel | Forum channel |
|---|---|---|---|
| **Releases** | New GitHub Releases (version tags, release notes) | `repo-release-monitor-template.yml` | `Repo_Release_Tracker_Template_Forum_Channel.yml` |
| **Commits** | New commits pushed to the default branch | `repo-commit-monitor-template.yml` | `Repo_Commit_Tracker_Template_Forum_Channel.yml` |

Everything in this README applies to all four unless a section says otherwise. The [Forum channel behavior](#forum-channel-behavior) section covers what's different about the forum versions. You can run any number of these together on the same repo — e.g. releases into a forum channel and commits into a regular one.

## Quick start

1. Change the `name:` field and the `concurrency: group:` value at the top of the file so it doesn't collide with any other monitor workflow in the same repo.
2. Fill in the `env:` block under the `monitor` job (`REPO`, `PROJECT_NAME`, `STATE_FILE`, `PING_MODE`, `PING_ID`, and `THREAD_STATE_FILE` for forum-channel templates).
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
   | `REPO` | The `owner/repo` you want to watch, e.g. `ChrisTitusTech/winutil` |
   | `PROJECT_NAME` | Display name used in the Discord message title, e.g. "winutil" |
   | `STATE_FILE` | Filename used to remember the last commit/release seen. Only needs changing if you add more than one monitor to the same repo — give each one a different filename. Can include a subfolder (e.g. `state/last_release.txt`) — the folder is created automatically if it doesn't exist yet. |
   | `THREAD_STATE_FILE` | *Forum-channel templates only.* Filename used to remember an in-progress forum post if a run fails partway through, so the next run resumes posting into that same thread instead of creating a duplicate one. Same rules as `STATE_FILE` — give each monitor its own unique filename. |
   | `PING_MODE` | Who gets pinged — see below |
   | `PING_ID` | A Discord user or role ID — see below |
   | `NOTIFY_ON_INITIAL_RUN` | `"true"` (default) or `"false"` — see [First run behavior](#first-run-behavior) below |

4. **Add the webhook secret.** In the repo you copied the file into: **Settings → Secrets and variables → Actions → New repository secret**, name it `DISCORD_WEBHOOK`, and paste in your Discord webhook URL (Discord channel → Edit Channel → Integrations → Webhooks).

   > **Using a forum-channel template?** Create the webhook on the **Forum channel itself** — right-click the forum channel (or its Channel Settings) → Integrations → Webhooks — not on a text channel or a specific thread inside it. A forum channel's webhook behaves differently from a text channel's, so pointing the wrong template at the wrong webhook will fail on the first post (see [Troubleshooting](#troubleshooting)).

   Treat this URL like a password — anyone who has it can post to your channel. It only needs to live in the repo secret; it's never printed to logs or committed anywhere.

   `GITHUB_TOKEN` needs no setup — GitHub provides it automatically, and the workflow uses it just to get a higher GitHub API rate limit and to commit the state file back to the repo.

5. **Commit and push.** The workflow runs automatically every 5 minutes. To test it immediately without waiting, go to the **Actions** tab → select the workflow → **Run workflow**. Watch the run's logs — the script prints what it found (`No new release.`, `New releases to report: N`, etc.) so you can confirm it's working before you walk away from it.

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

Only the *first* message of a batch is pinged (see [Catching up on multiple items](#catching-up-on-multiple-items) below) — if five releases land at once, you get pinged once, not five times.

You don't need to touch any code for any of this — `PING_MODE` and `PING_ID` are the only two things to set.

## How it behaves

### Polling and state

Both templates poll on a `*/5 * * * *` cron schedule (every 5 minutes — GitHub Actions' minimum granularity for scheduled workflows) and can also be run on demand via **Actions → Run workflow**. Each run:

1. Fetches recent releases/commits from the GitHub API.
2. Compares the newest one against `STATE_FILE`, which is committed back into the repo the workflow lives in.
3. If nothing's changed, it exits quietly (this is the normal outcome for almost every run).
4. If something's new, it posts to Discord and updates `STATE_FILE`.

`cancel-in-progress: true` means if a run is somehow still going when the next scheduled run starts (it shouldn't be — runs normally finish in a few seconds), the older one is cancelled rather than letting two runs race each other.

### Catching up on multiple items

If the monitor was offline for a while (Actions outage, repo paused, etc.), it doesn't just report the single newest item — it catches up on everything missed since the last check, oldest first, up to 10 items per run (`MAX_RELEASES_PER_RUN` / `MAX_COMMITS_PER_RUN` in the script). If more than 10 landed, the oldest ones beyond that cap are skipped with a note (`_N earlier release(s) in this batch were not posted individually._`) rather than flooding the channel — you'll still see the current latest one.

If the last-seen item has fallen out of GitHub's recent history entirely (e.g. very heavy release/commit activity, or a force-push that rewrote history), it can't reconstruct the full gap — it reports just the latest item with a note (`_Could not find the last-seen release in recent history..._`) instead of guessing.

Only the *first* item in a batch gets pinged and carries the note; the rest post silently to avoid spamming the channel.

**State is saved incrementally, after each item in a batch — not just once at the end.** If Discord or the GitHub API has a hiccup partway through a batch of, say, 5 releases, the ones that already posted successfully are remembered, so the next run resumes from where it left off instead of reposting them.

### Forum channel behavior

*(Applies only to the two forum-channel templates.)* Everything above works the same way, with one structural difference: **each new release or commit becomes its own forum post (thread)**, instead of a message dropped into a shared channel.

- The thread's title is `{PROJECT_NAME} Updated! — {tag}` for releases, or `{PROJECT_NAME} Updated! — Commit {short SHA}` for commits — truncated to Discord's 100-character forum title limit if it runs long.
- If one release/commit's notes need more than one message, the "(continued)" follow-ups are posted as replies inside that same thread, not as new posts.
- In a batch of several new releases/commits, each one gets its *own* thread — 3 new releases means 3 new forum posts, not one post with 3 messages. The ping still only fires once, on the first (oldest) thread in the batch.
- **Resume-safe by design:** before sending each continuation message, the thread ID and next part number are saved to `THREAD_STATE_FILE`. If a run fails partway through posting a long item (e.g. Discord errors out on message 2 of 3), the next run reads that file and resumes posting into the *same* thread starting from the part that failed — rather than creating a duplicate post. The progress file is cleared automatically once an item finishes posting.

### First run behavior

The very first time the monitor runs (no `STATE_FILE` exists yet), there's nothing to compare against — so by default it treats the current latest release/commit as "new" and posts a notification for it, ping included. This is useful as a quick confirmation the monitor is wired up correctly, but if you're adding this to a repo that already has a long release history, it means you'll be pinged once about something that isn't actually new.

Set `NOTIFY_ON_INITIAL_RUN: "false"` to skip that — the first run will silently record the current latest release/commit as the starting point, and you'll only be notified about things that land *after* that.

### Message length and splitting

The ping, title, and author/link info are sent together in one message along with as much of the release notes/commit message as fits in a single Discord message. If the content is longer than that, it spills into "(continued)" follow-up messages — always breaking on whole lines, so a bullet point, a link, or a heading is never cut in the middle. Extremely long unbroken lines (e.g. one giant paragraph with no line breaks) fall back to breaking on whitespace, but a single word is never split.

### Reliability

- Network hiccups (timeouts, DNS failures) are retried a few times with a short backoff before the run gives up.
- If GitHub or Discord responds with a rate-limit error (HTTP 429) — which can happen if a batch produces many messages in a row — the script waits (honoring the amount of time the server asks for) and retries automatically, up to 5 times, rather than failing the run.
- A small delay is kept between consecutive Discord messages specifically to avoid tripping that rate limit in the first place.
- Any other error from GitHub or Discord (bad webhook URL, repo not found, etc.) is *not* retried — it fails the run immediately so you see it in the Actions tab rather than it quietly retrying forever.

### Links in release notes / commit messages

Bare `#123` references *and* full bare PR/issue URLs are both converted into real clickable links before sending. Some projects mix both styles within one release (part hand-written, part auto-generated) — GitHub renders either form as a link on its own pages, but Discord does neither on its own, so both get normalized. Text that's already a markdown link (`[#123](...)`) is left alone rather than double-linked.

## Troubleshooting

**Nothing posted, and no error in the Actions tab.** Most likely there's genuinely nothing new — check the run's log for `No new release.` / `No new commit.`. If you expected something to post, confirm `REPO` is spelled correctly (`owner/repo`, case matters) and that the release/commit actually exists on the repo's default branch (the commit monitor only watches the default branch).

**The workflow never runs on schedule.** GitHub disables scheduled workflows on repos with no activity for 60 days — push a commit or re-enable it manually from the Actions tab. Scheduled runs can also be delayed by several minutes during periods of high GitHub load; this is a GitHub-side limitation, not something this workflow can control.

**Discord returns an error / nothing shows up in the channel.** Double check the `DISCORD_WEBHOOK` secret is set on the *same repo* the workflow file lives in, and that the webhook hasn't been deleted or regenerated on the Discord side (regenerating a webhook changes its URL). The run's log will print the exact HTTP error Discord returned.

**Forum-channel template fails, or Discord's error mentions `thread_name`.** The `DISCORD_WEBHOOK` almost certainly belongs to the wrong channel type — forum-channel templates need a webhook created *on a Forum channel*, and will fail immediately if pointed at a regular text channel's webhook (and a regular-channel template will likewise fail if pointed at a forum channel's webhook). Recreate the webhook on the correct channel type and update the secret.

**`git push` fails at the "Save monitor state" step.** This usually means branch protection rules on the default branch are blocking direct pushes, even from GitHub Actions. Either relax the protection rule for the `github-actions[bot]` actor, or add a small config repo that has no protection just for this workflow's state file.

**You want to re-test a notification you already saw.** Delete or edit `STATE_FILE` in the repo (set it to an older tag/SHA, or delete it entirely) and trigger a manual run — the monitor will treat it as new again.

## Security & permissions notes

- The workflow requests only `contents: write` — just enough to read the repo it's running in and commit the state file back. It does not request access to issues, pull requests, Actions settings, or anything else.
- That permission is scoped to the repo the *workflow* lives in, not the repo being *monitored*. `GITHUB_TOKEN` only grants access within its own repo, so monitoring a different, private repo than the one hosting the workflow won't work with the default token — see the FAQ below.
- `PING_ID` and `REPO`/`PROJECT_NAME` are plain config, not secrets — they're visible to anyone who can read the workflow file. Only `DISCORD_WEBHOOK` needs to be a secret.

## FAQ

**Can I monitor a private repo?** Only if it's the *same* repo the workflow file lives in — the automatic `GITHUB_TOKEN` only has access to its own repo. To monitor a different private repo, you'd need to supply a personal access token with read access to that repo (as an additional secret) and use it in place of `GITHUB_TOKEN` in the script's API calls. Public repos work regardless of where the workflow lives.

**Can I run more than one of these templates on the same repo?** Yes, any combination of the four — just make sure each one has a unique `name`, `concurrency: group`, `STATE_FILE`, and (for forum-channel templates) `THREAD_STATE_FILE`, as described in step 2 and the config table above.

**Can I point a forum-channel template at a regular text channel, or vice versa?** No — a Discord webhook is tied to one specific channel and channel type. Use the template that matches your channel (see the table at the top of this README), and see [Forum channel behavior](#forum-channel-behavior) for what's different about the forum versions.

**Can I change the polling frequency?** Yes, edit the `cron:` line under `on: schedule:`. 5 minutes is GitHub Actions' minimum; you can go less frequent (e.g. `*/15 * * * *` or `0 * * * *` for hourly) if you don't need near-real-time notifications.

**How do I pause or stop a monitor?** Go to **Actions** → select the workflow → the **⋯** menu → **Disable workflow**. Re-enable it the same way. Deleting the workflow file removes it entirely.

**Does this see prereleases/draft releases?** No — by default the release monitor skips draft releases and prereleases, matching GitHub's own definition of "latest release." If you want to be notified about prereleases too, that's a small code change inside the `get_recent_releases` function (drop the `prerelease` filter) — not exposed as a config variable here.

**What about GitHub API rate limits?** Unauthenticated GitHub API access is capped at 60 requests/hour; this template authenticates with `GITHUB_TOKEN` automatically so that isn't a concern unless you're running a very large number of monitors from one repo.
