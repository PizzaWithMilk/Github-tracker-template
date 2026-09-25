# Repo Monitor

[📖 Visual Tutorial](Tutorial.md)

## What this actually does

This is one ready-to-use automation file for GitHub. Once you set it up, it quietly checks one or more GitHub repos (projects) every 5 minutes, and if something new shows up — a new release or a new commit — it automatically posts a message about it in Discord. You don't need to run anything on your own computer; GitHub runs it for you, on a timer, forever (or until you turn it off).

Everything about *how* it behaves — which repos it watches, whether it watches releases or commits or both, which Discord channel each one posts to, who gets pinged, and a handful of formatting options — is controlled entirely by one small config file, `config/repos.yml`, that sits alongside the workflow. You never edit the workflow file itself to change any of that. See [Configuring repos.yml](#configuring-reposyml) below.

One workflow file can watch **any mix** of repos at once — some posting to a regular text channel, others to a Forum channel, others tracking only releases, others only commits, all in the same run, each configured independently in `repos.yml`. It also watches out for a separate GitHub quirk that can silently turn the whole thing off on its own — see [Inactivity reminder](#inactivity-reminder) below.

## A few words explained, before we start

If you already know GitHub Actions and Discord webhooks well, skip this part. If you're newer to this, these quick definitions will make the rest of this README much easier to follow:

- **Repo (repository):** a project's folder of files on GitHub.
- **Workflow:** an automation recipe GitHub runs for you. It's a text file ending in `.yml`, and it lives in a special folder called `.github/workflows/` inside a repo. The file described in this README *is* the workflow.
- **Config file:** a separate small text file, `config/repos.yml`, that lists which repo(s) to watch, where to post about them, and who to ping. It's not a workflow itself — it's data the workflow reads. Explained in full in [Step 2](#step-2-create-your-reposyml-config-file).
- **Webhook:** a special web address (URL) that Discord gives you for one specific channel. Any program that sends a message to that address will have its message posted in that channel — no bot, no login, no password needed on Discord's side. Anyone who has the URL can post to your channel with it, so it's worth keeping private — see [Security & permissions notes](#security--permissions-notes) for exactly where it lives in this setup and what that means.
- **Secret (in GitHub):** a private value that you store in your repo's settings, which workflows can use without it ever being shown in logs or visible to anyone browsing the repo. This workflow doesn't require you to create any secrets — see the note in [Step 4](#step-4-add-your-discord-webhook).
- **State directory:** a small folder this workflow keeps inside your repo, purely to remember "the last thing I already told you about" — one small bookmark file per repo (and per update type) you're watching, named automatically. You never need to create or edit these files yourself.
- **Cron schedule:** a standard way of telling a computer "run this automatically, on repeat, at these times" — e.g. "every 5 minutes." You'll see a line like `*/5 * * * *` in the file; you don't need to understand that syntax to use the template as-is.
- **Commit / push:** "commit" means saving a change permanently in a repo's history; "push" means sending that saved change up to GitHub. When this README says the workflow "commits" or "pushes" something, it's the automation doing this on its own — you don't have to do anything for that part.
- **Forum channel vs. regular (text) channel:** explained in the next section.

## Text channels, Forum channels, releases, and commits — all in one file

Older versions of this project used four separate template files (one each for releases/commits × regular/Forum channels). That's no longer how this works. There's just **one workflow file**, and every one of those choices is now a setting *per repo* inside `config/repos.yml`:

- `type:` — `normal` (a regular text channel) or `forum` (a Forum channel). Decides how that repo's notifications get posted.
- `monitor:` — `commits`, `releases`, or `both`. Decides what kind of updates get tracked for that repo.

**Not sure which kind of Discord channel you have?** Look at the channel in Discord:

- If it looks like a normal chat — one continuous stream of messages — that's a **regular text channel**. Use `type: normal`.
- If it shows a grid/list of separate titled "posts," each one opening into its own conversation, with a **"New Post"** button — that's a **Forum channel**. Use `type: forum`.

Most Discord channels are regular text channels, so if you're not sure, that's the safer guess.

Because `type` and the destination webhook are both per-repo settings (see [Routing different repos to different channels](#routing-different-repos-to-different-discord-channels)), a single run of this workflow can post some repos' releases to a Forum channel in one server, someone else's commits in a regular channel in a different server, and so on — all from the one file, checked on the one schedule.

## Quick start (short version)

If you've done this kind of thing before, here's the condensed version — the full walkthrough with screen-by-screen detail is right after this:

1. Create the workflow under `.github/workflows/` and keep its `name:` and concurrency groups unique if you run more than one copy in the same repository.
2. Create a `config/repos.yml` file listing the repo(s) you want to watch, a Discord webhook URL, your default ping settings, and (optionally) per-repo overrides — see [Configuring repos.yml](#configuring-reposyml).
3. Keep the workflow's `CONFIG_FILE`, `STATE_DIR`, and `NOTIFY_ON_INITIAL_RUN` settings as needed. If you change the config path away from `config/repos.yml`, update the validation job's `CONFIG_FILE` too.
4. Commit the workflow and `repos.yml` to the repo.

No GitHub secret needs to be created — the Discord webhook URL goes directly into `repos.yml` (see [Step 4](#step-4-add-your-discord-webhook) for what that means for privacy).

## Full setup walkthrough

Take this step by step — none of it requires programming knowledge, just careful copy/paste.

### Step 1: Put the file where GitHub can run it

**This file needs to live inside a GitHub repo, in a specific folder, for GitHub to notice and run it.** It does **not** have to be the repo you're monitoring — you can create a small, separate repo just for this if you want, or add it to a repo you already have. All it needs is somewhere GitHub Actions can run and occasionally save a tiny state file.

If the `.github/workflows/` folder doesn't already exist in that repo, you'll need to create it. Two ways to do this:

- **Directly on GitHub.com (no software needed):** Open your repo in a browser → click **Add file → Create new file**. In the box where you'd normally type a filename, type the *entire path* including the folders, like this: `.github/workflows/repo-monitor.yml`. As soon as you type each `/`, GitHub automatically creates that folder for you — you don't need to create `.github` and `workflows` as separate steps. Paste in the full contents of the template file, then scroll down and click **Commit changes**.
- **On your own computer, using git:** inside a local copy (clone) of the repo, run `mkdir -p .github/workflows` to create the folder, save the template file inside it (keep the `.yml` file ending), then run `git add .github/workflows/<filename>.yml`, `git commit -m "Add repo monitor"`, and `git push` to send it up to GitHub.

The exact filename you save it as inside that folder doesn't matter — `repo-monitor.yml`, `discord-updates.yml`, anything — as long as it ends in `.yml` and sits inside `.github/workflows/`.

### Step 2: Create your repos.yml config file

This is where you tell the workflow which repo(s) to watch, where to post, and who to ping — it's a separate file from the workflow itself.

1. In the same repo you put the workflow file in, create a new file at `config/repos.yml` (same two methods as Step 1 — type the full path including the `config/` folder when creating it on GitHub.com, or `mkdir -p config` locally).
2. Fill it in following this shape:

```yaml
ping_mode: user                          # user | role | everyone | none
ping_id: "YOUR_DISCORD_USER_ID_HERE"     # required only when ping_mode is user or role
webhook: "YOUR_DISCORD_WEBHOOK_URL_HERE" # see Step 4 for how to get this

repos:
  - repo: OWNER/REPO
    type: normal                          # normal | forum - must match the kind of channel this repo posts to
```

`ping_mode`, `ping_id`, and `webhook` at the top are all *defaults* — used by every repo below that doesn't set its own override. See [Configuring repos.yml](#configuring-reposyml) for the full list of fields, and [Choosing who gets pinged](#choosing-who-gets-pinged) for what each `ping_mode` value does.

Every repo block needs, at minimum, `repo:` (written as `OWNER/REPO`, and it has to be the *first* field in the block) and `type:` (`normal` for a regular text channel, `forum` for a Forum channel). There's no field for a custom display name — Discord will show the repo's own name (the part after the last `/`), e.g. `owner/winutil` is shown as `winutil`.

3. If `CONFIG_FILE` in the workflow's `env:` block still says `config/repos.yml` (the default), you don't need to change anything else in the workflow file — just make sure `repos.yml` actually lives at that exact path in the repo.

### Step 3: Fill in the workflow's settings block

Further down in the file, under `jobs: monitor: env:`, you'll find a small block of settings:

| Setting | What it means |
|---|---|
| `CONFIG_FILE` | Path to `repos.yml`, relative to the repository root. The monitor job defaults to `config/repos.yml`. If you change it, change the validation job's `CONFIG_FILE` in the `validate` job to the same path so validation checks the same file. |
| `STATE_DIR` | The folder where the workflow keeps its bookmark, failure, and multi-part delivery resume files. The default, `.github/monitor-state`, is fine for most setups. |
| `NOTIFY_ON_INITIAL_RUN` | `"true"` (the default) or `"false"`. Controls what happens the first time this monitor checks a given repo for a given update type — see [First run behavior](#first-run-behavior) below. |

### Step 4: Add your Discord webhook

This is how the workflow is actually able to post into your Discord channel.

1. In Discord, go to the channel you want notifications posted into. Open its settings (for a regular text channel: the gear icon, or right-click → **Edit Channel**; for a forum channel: right-click the channel → **Edit Channel**).
2. Go to **Integrations → Webhooks → New Webhook** (or **Create Webhook**).
3. Give it any name you like, then click **Copy Webhook URL**.

   > **Using a Forum channel?** Make sure you create this webhook **on the Forum channel itself** — not on a regular text channel, and not inside one specific thread. A Forum channel's webhook works a little differently under the hood, so if you accidentally use the wrong kind of webhook with `type: forum` (or vice versa), it will fail the very first time it tries to post (see [Troubleshooting](#troubleshooting) if that happens).

4. Paste that URL into `config/repos.yml`, either as the top-level `webhook:` default, or inside a specific repo's block if you want that repo to post somewhere different — see [Routing different repos to different Discord channels](#routing-different-repos-to-different-discord-channels).

**You do not need to create any GitHub secret for this.** Unlike some earlier versions of this kind of tool, the webhook URL is a plain setting in `repos.yml`, right alongside everything else — not a hidden secret. That's simpler to set up, but it does mean the URL is stored as visible plain text in your repo. Read [Security & permissions notes](#security--permissions-notes) below before deciding where to put it, especially if the repo hosting the workflow is public.

You do **not** need to do anything for `GITHUB_TOKEN` either — GitHub creates and provides this automatically for every workflow run. This workflow uses it to talk to GitHub's API a bit more freely, and to save its own bookmark files back into the repo.

### Step 5: Turn it on and check it worked

Commit and push both files if you haven't already (or click **Commit changes** if you used the GitHub.com website). Once they're saved, the workflow will start running automatically every 5 minutes, forever, without you needing to do anything else.

To check it's working right away, without waiting:

1. Go to your repo's **Actions** tab.
2. Click on the workflow's name in the list on the left.
3. Click the **Run workflow** button, then confirm.
4. After a few seconds (a bit longer if you're watching several repos, or a rate limit briefly kicks in), click into that run and open up its steps to read the log. You should see printed lines like `No new release.` / `No new commit.` (meaning it checked and there was nothing new — this is a completely normal, healthy result) or `New releases to report for OWNER/REPO: 1` (meaning it found something and should have posted it to Discord). If you listed several repos in `repos.yml`, you'll see a block of lines like this for *each* repo, one after another, in the same run.

A run with a green checkmark ✅ means everything worked. A red ✕ means something failed — see [Troubleshooting](#troubleshooting).

## Config validation

The same workflow also contains a separate `validate` job so bad config changes can be caught before you rely on the monitor.

- A **push** or **pull request** that changes `config/repos.yml` or the workflow file runs the `validate` job.
- A **scheduled run** or **manual `Run workflow`** runs the `monitor` job instead.
- Validation does not contact Discord, does not query the monitored repositories, and does not modify monitor state.
- The validation job uses its own concurrency group (`repo-monitor-validation`) and can cancel an older validation run when a newer config change arrives.
- The monitor job uses a separate `repo-monitor` concurrency group and does **not** cancel an active monitor run.

When validation fails, read the `validate` job log. The parser reports the config problem and the line it found it on. Fix the config, commit the change, and GitHub will start a new validation run automatically.

## Configuring repos.yml

`config/repos.yml` has a small set of top-level (global) settings, followed by a `repos:` list where each block can override most of those settings just for that one repo.

### Global settings

| Setting | Default | What it does |
|---|---|---|
| `ping_mode` | `user` | Default ping behavior for every repo that doesn't override it — see [Choosing who gets pinged](#choosing-who-gets-pinged). |
| `ping_id` | *(none)* | Required only when `ping_mode` is `user` or `role`. |
| `webhook` | *(none — required unless every repo sets its own)* | The default Discord webhook URL. Any repo without its own `webhook:` uses this one. |
| `github_api_base_url` | `https://api.github.com` | Change this only if you're monitoring a repo hosted on a GitHub Enterprise Server instance rather than github.com — e.g. `https://github.example.com/api/v3`. |
| `commit_accent_color` | `5865F2` (Discord blurple) | Default accent color (the vertical stripe on the message) for commit notifications. |
| `release_accent_color` | `FEE75C` (gold) | Default accent color for release notifications. |
| `inactivity_accent_color` | `ED4242` (red) | Accent color for the [inactivity reminder](#inactivity-reminder) message. |

Colors can be written as `#RRGGBB`, `RRGGBB`, `0xRRGGBB`, or a plain decimal number — whichever's easiest for you.

### Per-repo settings

Every block under `repos:` supports these fields. Only `repo` and `type` are required; `monitor` defaults to `both`, and the other fields fall back to a sensible default or to the matching global setting.

| Field | Default | What it does |
|---|---|---|
| `repo` | *(required)* | `OWNER/REPO`. Must be the first field in the block. |
| `type` | *(required)* | `normal` or `forum` — which kind of Discord channel this repo's notifications go to. |
| `monitor` | `both` | `commits`, `releases`, or `both` — which kind(s) of updates to track for this repo. |
| `ping` | *(inherits `ping_mode`)* | Overrides `ping_mode` just for this repo. |
| `ping_id` | *(inherits `ping_id`)* | Overrides `ping_id` just for this repo. |
| `webhook` | *(inherits the global `webhook`)* | Overrides which Discord channel this repo posts to — see [Routing different repos to different Discord channels](#routing-different-repos-to-different-discord-channels). |
| `github_api_base_url` | *(inherits the global value)* | Per-repo override, for a repo hosted on a different GitHub instance than the rest of your list. |
| `accent_color` | *(inherits `commit_accent_color`/`release_accent_color`)* | Overrides the embed accent color just for this repo. |
| `include_release_assets` | `true` | Release tracking only. Lists the release's downloadable files as links under "Release Assets." |
| `short_summary` | `false` | Release tracking only. Shortens long release notes instead of posting them in full — see [Release notes options](#release-notes-options). |
| `summary_lines` | `8` | Release tracking only, used with `short_summary`. How many lines of the release notes to keep. |
| `tags` | *(none)* | Forum channels only. A list of Forum tag IDs to apply to this repo's posts — see [Forum channel behavior](#forum-channel-behavior). |

There is **no `digest` setting in this version**. Adding `digest:` to a repository block is treated as an unknown setting and will fail config validation.

### Putting it together

```yaml
ping_mode: user
ping_id: "YOUR_DISCORD_USER_ID_HERE"
webhook: "YOUR_DEFAULT_DISCORD_WEBHOOK_URL_HERE"

repos:
  - repo: ChrisTitusTech/winutil
    type: normal                # monitor not set - uses the default (commits + releases)

  - repo: torvalds/linux
    type: normal
    monitor: releases           # release monitoring only

  - repo: someuser/some-other-repo
    type: normal
    monitor: commits            # only commits are tracked for this repo

  - repo: someorg/chatty-repo
    type: forum
    ping: role
    ping_id: 123456789012345678
    short_summary: true
    summary_lines: 6
    tags:
      - 111111111111111111
```

A few more things worth knowing about how this works:

- **Pinging happens per repo, not once for the whole run.** If three of your listed repos each happen to have something new in the same 5-minute check, each one pings independently — using its own ping settings, whether that's the config's default or a per-repo override — so you'd get three pings, not one shared ping for the whole run. Within a single repo's own batch of catch-up items, only the first one still gets pinged; see [Catching up on multiple items](#catching-up-on-multiple-items).
- Each repo's bookmark file (inside `STATE_DIR`) is completely independent — one for its commit tracking, one for its release tracking, if `monitor: both` — so one repo catching up on a backlog has no effect on any other repo, or on the other update type for the same repo.
- If one repo in the list is broken somehow (renamed, deleted, a typo, or a repeated API error), it won't stop the *other* repos from being checked and posted about in that same run — but the overall run will still show as failed (a red ✕) in the Actions tab so you notice. Check the run's log for a line starting with `Error while monitoring` to see exactly which repo it was.
- One exception: if `repos.yml` itself is malformed — a missing `/` in a repo name, an invalid `type` or `ping` value, a `user`/`role` ping with no `ping_id` given, a repo with no webhook available to it, or a setting placed somewhere it isn't allowed, for example — the *entire* run fails immediately, before any repo gets checked at all. See [Troubleshooting](#troubleshooting).
- There's no field for a custom display name — Discord always shows the repo's own name (the part after the last `/`) as the project name in the message.

## Routing different repos to different Discord channels

Because `webhook:` can be set per repo, one workflow run can post different repos to entirely different channels — even different Discord servers — without needing separate workflow files or separate config files:

```yaml
webhook: "https://discord.com/api/webhooks/DEFAULT_CHANNEL_WEBHOOK"

repos:
  - repo: owner/project-one
    type: normal
    # no webhook set - uses the default channel above

  - repo: owner/project-two
    type: forum
    webhook: "https://discord.com/api/webhooks/A_DIFFERENT_FORUM_CHANNEL_WEBHOOK"

  - repo: owner/project-three
    type: normal
    webhook: "https://discord.com/api/webhooks/A_THIRD_SERVER_ENTIRELY"
```

A repo's `type` (`normal`/`forum`) simply has to match whichever channel its own effective webhook (its own `webhook:`, or the global default if it doesn't set one) actually points to — mixing them up will fail the first time that repo tries to post (see [Troubleshooting](#troubleshooting)).

If you'd rather keep things fully separate for other reasons (independent scheduling, independent state, etc.), you can still run more than one copy of this workflow file, each with its own `CONFIG_FILE`, `STATE_DIR`, `name`, and `concurrency: group:` — but for simply posting to more than one channel, one workflow with per-repo `webhook:` overrides is usually all you need.

## Choosing who gets pinged

"Pinged" means Discord sends someone a notification, the same as if you typed `@username` in a message. `ping_mode` in `config/repos.yml` controls whether that happens, and for whom.

Set `ping_mode` (at the top of `config/repos.yml`, or per-repo via `ping:` — see [Configuring repos.yml](#configuring-reposyml)) to one of these four values:

| `ping_mode` | What it does | Also set `ping_id` to... |
|---|---|---|
| `user` (default) | Pings one specific person | Their Discord **user ID** (a long number — see below for how to get it) |
| `role` | Pings everyone who has a specific role | The **role ID** |
| `everyone` | Pings `@everyone` in the channel (notifies everyone who can see it) | *(not needed — leave it out)* |
| `none` | Posts the update with no ping at all — just a normal message | *(not needed — leave it out)* |

**How to find a Discord user ID or role ID:** In Discord, go to **User Settings → Advanced**, and turn on **Developer Mode**. Once that's on, you can right-click any person's name or any role and a new option appears: **Copy User ID** or **Copy Role ID**. Paste that number into `ping_id` (quoting it, like `ping_id: "123456789012345678"`, is a safe habit for the top-level default, though not required).

**A note on `everyone`:** this pings *everyone* who can see that channel, whether they're active right now or not — it's meant for a channel people already expect update pings in (like a dedicated `#releases` channel), not a general chat channel where it would be disruptive.

The top-level `ping_mode`/`ping_id` in `config/repos.yml` are the *defaults* — every repo uses these unless its own block overrides them.

Only the *first* message of a batch gets a ping — so if five releases land at once for one repo because the monitor catches up on things it missed, you'll be pinged once for that repo, not five times. If you're watching multiple repos, each one pings independently on its own terms. (More on batches in [Catching up on multiple items](#catching-up-on-multiple-items) below.)

## How it behaves

This section explains what actually happens behind the scenes, so you know what to expect.

### Polling and state

"Polling" just means "checking in periodically to see if anything changed." The workflow polls every 5 minutes (GitHub's fastest allowed schedule for this kind of automation), and can also be run manually any time via **Actions → Run workflow**.

Every single time it runs, here's exactly what happens, for each update type (commits and/or releases, per that repo's `monitor:` setting) on each repo in `config/repos.yml`, in turn:

1. It asks GitHub's API: "what are the most recent releases/commits on this repo?"
2. It compares the newest one against what's saved in that repo's bookmark file inside `STATE_DIR` — i.e., "is this something I've already told you about, or is it new?"
3. If nothing's changed since last time, it does nothing else for that repo/update-type and moves on. **This is the normal result for almost every single run** — most of the time, nothing new has happened, and that's expected, not a sign anything is broken.
4. If something *is* new, it posts about it in Discord, then updates that bookmark file so it doesn't mention that same thing again next time.

One more detail: scheduled/manual monitoring runs use a concurrency group with `cancel-in-progress: false`, so an active monitor run is allowed to finish and save its state instead of being canceled by a newer monitoring run. Push/PR validation runs use their own separate concurrency group, so validation does not queue behind the monitor.

### Which commits actually get reported

Commit tracking doesn't just repost every single commit message verbatim — it tries to pull out the actual, user-facing content of each one, and skips the ones that don't have any:

- **Bot and automation commits are always skipped** — anything from `github-actions[bot]`, Dependabot, and similar automated identities.
- **Merge commits are always skipped.**
- For everything else, it looks at the commit message and tries to extract meaningful **Added / Removed / Changed / Fixed** information — either from an already-structured message (changelog-style headings and bullet points are recognized automatically), or from an ordinary sentence that uses a word like "added," "fixed," "removed," or "changed." A commit that's just a bare version bump with no other detail (`Bump to v1.2.3`, nothing else) isn't reported on its own.
- Commits that only touch things like formatting, linting, comments, the README, CI config, or dependency bumps — with nothing else described — are treated as non-update "noise" and skipped, though the notification for the next real commit will mention how many noise commits were skipped along the way.

When something *is* reported, the Discord message shows a **Change Log** broken into `### Added`, `### Removed`, `### Changed`, and `### Fixed` sections (whichever apply), plus a **Version** field pulled from the commit message if one is mentioned (falls back to "Unknown" if it isn't).

This filtering only affects individual commit notifications — it has no effect on release tracking, which always uses a release's actual, unedited release notes.

### Release notes options

Two settings, both release-tracking-only, control how much of a release's notes actually get posted:

- **`short_summary: true`** shortens the release notes to `summary_lines` lines (8 by default) instead of posting them in full, and adds a "View full release" link at the end pointing back to GitHub. Useful for projects that write very long release notes.
- **`include_release_assets: true`** (the default) lists that release's downloadable files as a "Release Assets" section with a direct download link for each one. Set it to `false` if you'd rather not see that list. Only the first 10 assets are listed — if a release has more than that, the section ends with a note like `Showing 10 of 14 release assets. View the GitHub release for all assets.` rather than omitting the rest without saying so.

Only actual GitHub *releases* are reported — draft releases and prereleases are skipped, the same as GitHub's own "latest release" definition on a repo's main page.

### Catching up on multiple items

If this monitor is ever offline for a while (say, GitHub has an outage, or you paused the workflow), it doesn't just tell you about the single newest thing when it comes back — it catches up and tells you about *everything* it missed for that repo, oldest first, up to 10 items in one run. This applies independently to each repo — and to each update type (commits vs. releases) on that repo — so one repo catching up on a large backlog doesn't affect the cap for any other repo.

If more than 10 things happened while it was away, the oldest ones beyond that limit are skipped (with a note saying how many were skipped) rather than flooding your channel with a wall of messages — you'll still always be told about the current latest one.

In one unusual edge case — extremely heavy activity, or a force-push that rewrites a repo's history — the last thing it remembers might no longer be found anywhere in the recent history at all. When that happens, it can't figure out exactly what was missed, so it just reports the single latest item along with a note explaining that it couldn't reconstruct the full gap, rather than guessing.

Only the very first item in a batch like this gets a ping and a note attached; the rest post quietly, so you're not pinged repeatedly.

**Behind the scenes, progress is saved after each item, not just once at the very end.** So if Discord or GitHub has a temporary hiccup partway through posting a batch of, say, 5 releases, whichever ones already posted successfully are remembered — the next run will pick up from there instead of accidentally posting those same ones again.

### First run behavior

The first time this monitor checks a *given* repo for a *given* update type — whether that's because the whole workflow is brand new, or because you just added a repo (or turned on `monitor: both`) that it hasn't seen before — there's no bookmark yet, so it has nothing to compare against. By default, it treats whatever the current latest release/commit is as "new" and posts a notification about it, ping included (subject to the once-per-batch ping rule described above).

This is handy as a quick way to confirm everything is wired up correctly. But if you're adding an established repo with a long release/commit history, it does mean you'll get a notification about something that isn't actually new — it already existed before you added it.

If you'd rather skip that, set `NOTIFY_ON_INITIAL_RUN` to `"false"`. With that set, the first time any given repo/update-type is checked, it will silently save the current latest release/commit as its starting bookmark, without posting anything — you'll only start getting notified about things that happen *after* that point. Since this is decided per repo (and per update type), it also covers anything you add to your config later, not just the very first run of the whole workflow.

### Message length and splitting

Discord messages have a length limit. The ping, title, and author/link details are sent together as one message along with as much of the release notes or commit change log as will fit. If there's more content than fits in one message, it continues into additional "(continued)" messages — and it's careful to only ever break between whole lines, so you'll never see a bullet point, a link, or a heading get awkwardly cut off in the middle. (Only in the rare case of one single, absurdly long line with no line breaks at all does it fall back to breaking mid-line — but even then, it never splits in the middle of a single word.)

### Accent colors

Each notification is shown with a colored accent stripe: blurple for commits and gold for releases by default, red for the inactivity reminder. You can change any of these globally (`commit_accent_color`, `release_accent_color`, `inactivity_accent_color` at the top of `repos.yml`) or per repo (`accent_color`, which overrides whichever default would otherwise apply for that repo's commit or release notifications) — see [Configuring repos.yml](#configuring-reposyml) for the exact field names and accepted formats.

### Forum channel behavior

Everything described above still applies to a `type: forum` repo — the one difference is **where things get posted**: instead of dropping a message into a shared, ongoing channel, **each new release or commit gets posted as its own new Forum post (a "thread")**, using that repo's own effective webhook.

- Each thread's title looks like `{Project Name} Updated! — {tag}` for releases, or `{Project Name} Updated! — Version {version}` for an individual commit notification (or "Unknown" in place of the version if none could be found in the commit message). (Forum post titles have a 100-character limit in Discord, so an unusually long one gets shortened automatically.)
- If one release or commit's notes are too long to fit in a single message, the extra "(continued)" messages are posted as replies *inside that same thread* — they don't create new, separate posts.
- If several new releases or commits show up in one run, **each one gets its own separate thread** — three new releases means three new Forum posts, not one post containing three messages. As above, only the first (oldest) one in that batch gets a ping.

**Forum tags (optional).** You can have a repo's posts automatically tagged with one or more of your Forum channel's tags, using a `tags:` list in that repo's block in `config/repos.yml`:

```yaml
repos:
  - repo: owner/project
    type: forum
    tags:
      - 111111111111111111
      - 222222222222222222
```

Discord allows at most 5 tags per post. This is genuinely the fiddly part: unlike user/role IDs, Discord doesn't give you a simple right-click **Copy ID** for Forum tags. The most reliable way to find one is to open your server in Discord's web app (in a browser, not the desktop app) → the Forum channel's settings → **Tags**, then open your browser's Developer Tools (F12) → **Network** tab, and look through the channel data Discord loads there — it lists each tag's name right next to its numeric ID. This whole feature is entirely optional; leave `tags:` out and nothing gets tagged.

**About the multi-part delivery resume files, and why you usually won't see them in your repo:**

Alongside each repo's regular bookmark file, `STATE_DIR` can hold a small temporary resume file for a long Discord delivery — one for normal-channel multi-part messages and one for Forum threads.

- The file records the repo, update type, item, part position, thread/message context, and a content fingerprint.
- If a run stops after part 2 of a 4-part notification, the next run can continue from the saved position instead of starting the whole notification again.
- If the saved content no longer matches the current content, the workflow refuses to mix the old and new parts; for Forum posts it discards the stale resume state and starts a fresh Forum post.
- If a saved Forum thread was deleted and Discord returns 404 while continuing it, the workflow creates a fresh Forum post and continues there.
- Once an item finishes completely, its temporary delivery state is removed.

So seeing only the ordinary bookmark files in `STATE_DIR` is the normal, healthy state. Resume files mainly appear after a delivery was interrupted partway through.

### Reliability

The monitor is designed to recover from common temporary failures, while making persistent problems visible instead of silently looping forever.

- Plain network errors such as timeouts and connection failures are retried up to 3 times with a short backoff.
- HTTP `429` rate-limit responses are retried up to 5 times, using `Retry-After` when Discord or GitHub supplies it, with each individual wait capped at 15 seconds.
- GitHub's **primary REST API rate limit** is handled separately. When GitHub reports `403` or `429` together with `X-RateLimit-Remaining: 0`, the workflow reads `X-RateLimit-Reset`. If the reset is only a short time away, it waits up to 30 seconds for the reset. Otherwise it stops checking more repositories for that run rather than wasting the rest of the run on requests that cannot succeed.
- GitHub and Discord HTTP `5xx` responses are retried a small number of times, and HTTP `408` is retried as a transient request failure.
- Discord sends are throttled with at least a half-second between messages to reduce the chance of hitting Discord's rate limits.
- GitHub permanent errors such as `404`, `401`, or unauthorized `403` are treated as persistent configuration/access problems rather than endlessly retried.
- Discord webhook errors `401`, `403`, and `404` are also treated as permanent. A deleted Forum thread is handled specially: the workflow creates a replacement Forum post and continues the interrupted delivery.
- Permanent failures alert immediately, once per persistent issue. Other transient failures are tracked per repository/update type and trigger a failure alert after 3 consecutive failed runs. Primary GitHub rate-limit exhaustion is tracked separately and alerts after 3 consecutive affected runs.
- Failure-alert state is stored in `STATE_DIR`, so the workflow does not forget an ongoing problem between runs. A successful recovery clears the corresponding failure state.
- Monitor runs serialize through the `repo-monitor` concurrency group with `cancel-in-progress: false`, so an active monitoring run is allowed to finish rather than being canceled by a newer scheduled run.
- Validation runs use a separate `repo-monitor-validation` concurrency group and are limited to config-checking events.


## Links in release notes / commit messages

Some projects write plain issue/PR references like `#123` in their release notes; others (especially auto-generated changelogs) write the full web address instead, like `https://github.com/owner/repo/pull/123`. GitHub turns both of these into clickable links automatically on its own website — but Discord does neither on its own. So this workflow converts both styles into proper clickable Discord links before posting, so they work no matter which style the original text used. If something is already written as a clickable link, it's left alone rather than being turned into a broken double-link. (If you're monitoring a repo on a GitHub Enterprise Server instance via `github_api_base_url`, these links correctly point back to that instance rather than github.com.)

## Inactivity reminder

This is unrelated to releases or commits — it's a safety net for a separate GitHub rule: **if a repo goes 60 days with absolutely no commits pushed to it, GitHub automatically disables any scheduled workflows in that repo**, this one included. That can happen silently, with no warning from GitHub — you'd typically only notice once you realize notifications have quietly stopped.

This matters most if you followed the suggestion in Step 1 to put this workflow in its own small, dedicated repo rather than one that already gets commits for other reasons. A repo used for nothing but hosting this workflow only ever gets a new commit when the workflow itself finds something new to report — so if every project you're monitoring goes quiet for a couple of months, that hosting repo could genuinely rack up 60 days of silence and get shut off without you ever being told.

To guard against that, every scheduled/manual monitoring run also quietly checks one extra thing: "has it been at least 59 days since the last commit to this repo?" If so — and only if you haven't already been sent this particular reminder — it posts a one-time Discord message titled **"Workflow Re-enable Reminder"**, pinging whoever you've configured via the top-level `ping_mode`/`ping_id` defaults in `config/repos.yml`.

This reminder always uses your **top-level `webhook:` default** specifically — not any per-repo webhook override. **If you don't set a top-level `webhook:` in `repos.yml` at all (relying only on per-repo overrides), the inactivity reminder is silently skipped**, since there's no obvious channel it should go to. If you want this safety net, make sure `webhook:` is set at the top level even if every one of your repos also sets its own override.

Whether the reminder posts as a Forum thread or a plain message is worked out automatically: if that top-level default webhook is also used by at least one `type: forum` repo in your config, the reminder posts as its own new Forum thread there (separate from any thread used for an actual release or commit); otherwise it posts as a normal message.

**Important: this message can't actually stop GitHub from disabling the workflow** — that decision is entirely GitHub's, made outside this workflow, and nothing a workflow run does from the inside can prevent it. What this message *does* do is warn you a day ahead of time, so once the workflow does get disabled, it isn't a surprise — you'll already know to go to your repo's **Actions** tab, click the workflow, and click **Re-enable workflow**, which starts everything running again immediately and takes about ten seconds.

A few more details worth knowing:

- You won't be spammed with this every 5 minutes once the 59-day mark passes — it's remembered (using GitHub's own Actions cache), so you'll only be told about a given quiet stretch once, not on every single run afterward.
- Any new commit to the repo — including this workflow's own bookmark-file update, the moment anything you're monitoring becomes active again — resets the clock automatically. You won't hear about it again unless another full 59 days of total silence pass.
- If the repo stays inactive even after you re-enable it, expect to be reminded again roughly every 59 days for as long as that remains true.
- This is fully automatic — there's nothing extra to set up or configure beyond having a top-level `webhook:` set. It reuses the same top-level `ping_mode`/`ping_id` you already filled in for regular notifications.

## Troubleshooting

**My `validate` job is skipped.** That's normal on a scheduled run or a manual `Run workflow`. The `validate` job only runs for `push` and `pull_request` events that touch `config/repos.yml` or the workflow file. Scheduled/manual runs use the `monitor` job instead. If you changed `CONFIG_FILE` away from `config/repos.yml`, make sure the validation job's own `CONFIG_FILE` setting points to the same file.

**The whole run fails immediately, before checking anything, and the log mentions something like `Invalid repository entry`, `Missing ping_id`, `Invalid type`, `Invalid monitor`, or `has no webhook`.** Something in `config/repos.yml` isn't formatted correctly — a missing `/` in a repo name, a `ping:` value that isn't `user`/`role`/`everyone`/`none`, a `user`/`role` ping with no `ping_id` set, a repo block missing its required `type:`, a `monitor:` value that isn't `commits`/`releases`/`both`, a repo with no webhook available to it (neither its own nor a top-level default), or a setting misplaced outside the `repos:` section, for example. The error message names the exact line number to check. A mistake in the config file blocks the *entire* run, since the workflow can't tell what to check yet. Fix the line and it'll pick back up on the next scheduled run (or click **Run workflow** to retry immediately).

**The log says `Config file not found`.** The workflow's `CONFIG_FILE` setting (Step 3) points somewhere that doesn't actually have a `repos.yml` there — usually because the file was never committed, or it's saved at a different path than `CONFIG_FILE` says. Double-check the file exists in the repo at that exact path.

**I got a Repo Monitor failure alert.** Permanent GitHub/Discord access errors are alerted immediately; transient repository failures are alerted after 3 consecutive failed runs. Check the exact error in the Actions log first. A successful run clears the corresponding failure state, so a one-time blip should not keep alerting forever.

**One repo in my config keeps failing, but the others post fine.** That's expected — see [Configuring repos.yml](#configuring-reposyml). The run still shows as failed overall so you notice, but the log will tell you exactly which repo via a line like `Error while monitoring OWNER/REPO: ...`, and every other repo in your config is still checked and posted about normally.

**Nothing posted, and there's no error in the Actions tab.** This is very likely completely normal — most runs find nothing new, since most 5-minute windows don't have a new release or commit in them. Open the run's log and look for a line like `No new release.` or `No new commit.` for the repo you're expecting — if you see that, everything is working correctly. If you expected something to have posted and didn't, double-check that repo's `repo:` value in `config/repos.yml` is spelled exactly right (`owner/repo-name`, and capitalization matters), that its `monitor:` setting actually includes the kind of update you're expecting, and that the release/commit genuinely exists on the repo's default (main) branch — the commit monitor only ever watches the default branch. If it's a commit you expected and it isn't showing up, also check [Which commits actually get reported](#which-commits-actually-get-reported) — not every commit qualifies.

**The workflow doesn't seem to run on its own schedule at all.** GitHub automatically disables scheduled workflows in a repo that's had no commit activity for 60 days. If that's happened, go re-enable it manually from the Actions tab — this workflow tries to warn you about this a day in advance over Discord (see [Inactivity reminder](#inactivity-reminder) above), so you shouldn't be caught completely off guard, but the fix either way is the same: **Actions tab → the workflow → Re-enable workflow**. Scheduled runs can also sometimes be delayed by several minutes when GitHub itself is under heavy load — that's a limitation on GitHub's end, not something this workflow controls.

**Discord shows an error, or nothing shows up in the channel.** Double-check the `webhook:` value that repo is actually using (its own, or the top-level default) in `config/repos.yml` — make sure it's a real, current webhook URL, and that it hasn't been deleted or regenerated on the Discord side (regenerating a webhook changes its URL, which would break it). The run's log will show you the exact error Discord sent back, which usually explains exactly what's wrong.

**A Forum-channel repo fails, especially with an error mentioning `thread_name`.** This almost always means that repo's effective webhook is pointing at the wrong kind of channel. Forum posting needs a webhook that was created *on a Forum channel specifically* — using a regular text channel's webhook for a `type: forum` repo (or vice versa) will fail immediately on the very first post. Go back to Discord, create a new webhook on the correct channel type, and update `repos.yml` with the new URL.

**The workflow fails specifically at the "Save monitor state" step, and `git push` is the part that errors.** This usually means the repo has branch protection rules that block direct pushes to the main branch, even from GitHub's own automation. Either loosen that rule specifically for the `github-actions[bot]` account, or use a separate, unprotected repo just for this workflow's bookmark files.

**You want to see a notification you've already gotten again, to test something.** Open `STATE_DIR` in your repo and find the bookmark file for that specific repo and update type — it's named automatically after the repo, e.g. `ChrisTitusTech__winutil_release.txt` or `ChrisTitusTech__winutil_commit.txt`. Either delete its contents, or change it to an older release tag / commit ID than the current one. The next run will then treat the current latest one as "new" again for that repo.

## Security & permissions notes

- This workflow only asks GitHub for permission to read its own repo and write (commit) back to it — nothing more. It cannot touch issues, pull requests, other repos, or your account settings.
- That permission only applies to the repo the *workflow file itself* lives in — not necessarily the repo(s) you list in `config/repos.yml`. This matters if you're trying to monitor a *different*, *private* repo than the one hosting the workflow — see the FAQ below for what to do in that case.
- **Your Discord webhook URL(s) are stored as plain, visible text in `config/repos.yml`, not as a GitHub secret.** Anyone who can view the hosting repo can see them — and, per the [Webhook](#a-few-words-explained-before-we-start) definition above, anyone with that URL can post messages into your Discord channel. This is a real change from setups that keep the webhook in a secret: if you'd rather it stayed hidden, keep the repo hosting this workflow private, or restrict who can view it. This trade-off is what makes it possible for different repos to post to entirely different webhooks without you needing to create a separate GitHub secret for each one — but it does mean `repos.yml` itself should be handled with the same care you'd give the webhook URL directly.
- `ping_id` and the list of repos you're watching are also stored as plain, visible text in `config/repos.yml` — the same visibility consideration applies to anyone who can view the repo.

## FAQ

**Can I use this to monitor a private repo?** Only if it's the *same* repo you put the workflow file into — the automatic permission GitHub gives the workflow only covers its own repo. To watch a *different* private repo, you'd need to create a personal access token with read access to that repo, add it as an additional secret, and use it in place of the automatic one in the script (this applies to all repos the script requests with it, so it's the simplest option when the repos you're adding are yours). Public repos can always be monitored, regardless of which repo hosts the workflow, and can be freely mixed with the hosting repo itself in the same `repos.yml` list.

**Can I watch more than one repo, or post to more than one Discord channel?** Yes to both, and you don't need more than one workflow file for either:
- **To watch several repos:** just add more blocks to `repos:` in `config/repos.yml` — see [Configuring repos.yml](#configuring-reposyml). Each repo can optionally have its own ping settings, accent color, and (on Forum repos) its own Forum tags; any repo that doesn't specify these just uses the config's top-level defaults.
- **To post different repos to different Discord channels (even different servers):** give the repos that should go elsewhere their own `webhook:` override — see [Routing different repos to different Discord channels](#routing-different-repos-to-different-discord-channels).

**Can I run more than one copy of this workflow on the same repo?** Yes — useful if you want entirely independent scheduling or state for some reason. Just make sure each workflow *file* has its own unique `name` and `concurrency: group`, and that their `STATE_DIR` values don't collide if they'd otherwise process the exact same repo/update-type (different repos are safe either way, since state filenames are generated from the repo name). If two monitors accidentally share a `concurrency: group`, they'll interfere with each other.

**Do I need a separate `STATE_DIR` for each repo in my config?** No — one shared `STATE_DIR` works for any number of repos. A uniquely-named bookmark file is created automatically for each repo/update-type, so they never collide.

**Can I use a Forum-channel webhook for a `type: normal` repo, or the other way around?** No. A Discord webhook is tied to one specific channel of one specific type, and Discord's API expects each kind to be talked to differently. Make sure `type:` and that repo's effective webhook actually match — see [Routing different repos to different Discord channels](#routing-different-repos-to-different-discord-channels).

**Can I make it check more or less often than every 5 minutes?** Yes. Find the `cron:` line under `on: schedule:` near the top of the file and change it. 5 minutes is the fastest GitHub allows; you can make it check less often if you don't need near-instant notifications — e.g. `*/15 * * * *` for every 15 minutes, or `0 * * * *` for once an hour.

**How do I pause or completely turn this off?** To pause it temporarily: go to the **Actions** tab, click the workflow, click the **⋯** menu, and choose **Disable workflow**. You can turn it back on the same way, whenever you like. To remove it permanently, just delete the workflow file from `.github/workflows/`.

**Will this notify me about draft releases or prereleases?** No, not by default — it only notifies about a repo's actual "latest release," the same definition GitHub itself uses, which excludes drafts and prereleases.

**Will this notify me about every single commit?** No — see [Which commits actually get reported](#which-commits-actually-get-reported). Bot commits, merge commits, and commits with no describable user-facing change are filtered out automatically.

**Will I hit GitHub's rate limits by running this?** Normal use is designed to stay comfortably within GitHub's authenticated API limits, and the workflow explicitly handles both ordinary `429` responses and primary rate-limit exhaustion. If the primary limit is actually exhausted, the workflow will wait briefly when the reset is imminent or skip the remaining repositories for that run instead of hammering the API.

**Why did I get a Discord message titled "Workflow Re-enable Reminder"?** That's expected — not an error, and nothing is broken. It means the repo hosting this workflow hasn't had a commit in about 59 days, which is right before GitHub's own 60-day cutoff for automatically disabling scheduled workflows. See [Inactivity reminder](#inactivity-reminder) above for the full explanation; the short version is: check the **Actions** tab in a day or so, and if the workflow has been disabled, click **Re-enable workflow**.
