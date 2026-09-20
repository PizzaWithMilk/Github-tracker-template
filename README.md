# Repo Monitor Templates

## What this actually does

These are ready-to-use automation files for GitHub. Once you set one up, it quietly checks a GitHub repo (project) every 5 minutes, and if something new shows up — a new release or a new commit — it automatically posts a message about it in a Discord channel. You don't need to run anything on your own computer; GitHub runs it for you, on a timer, forever (or until you turn it off). It also watches out for a separate GitHub quirk that can silently turn it off on its own — see [Inactivity reminder](#inactivity-reminder) below.

There are four template files. You'll use one or two of them, not all four — the table below helps you pick.

## A few words explained, before we start

If you already know GitHub Actions and Discord webhooks well, skip this part. If you're newer to this, these quick definitions will make the rest of this README much easier to follow:

- **Repo (repository):** a project's folder of files on GitHub.
- **Workflow:** an automation recipe GitHub runs for you. It's a text file ending in `.yml`, and it lives in a special folder called `.github/workflows/` inside a repo. The files in this README *are* workflows.
- **Webhook:** a special web address (URL) that Discord gives you for one specific channel. Any program that sends a message to that address will have its message posted in that channel — no bot, no login, no password needed on Discord's side. Anyone who has the URL can post to your channel with it, so it's treated like a secret.
- **Secret (in GitHub):** a private value — like a password or webhook URL — that you store in your repo's settings. Workflows can use it, but it's never shown in logs and no one browsing your repo can see its value once it's saved.
- **State file:** a small text (or JSON) file this workflow keeps inside your repo, purely to remember "the last thing I already told you about." Think of it like a bookmark. You never need to edit it yourself.
- **Cron schedule:** a standard way of telling a computer "run this automatically, on repeat, at these times" — e.g. "every 5 minutes." You'll see a line like `*/5 * * * *` in the file; you don't need to understand that syntax to use the template as-is.
- **Commit / push:** "commit" means saving a change permanently in a repo's history; "push" means sending that saved change up to GitHub. When this README says the workflow "commits" or "pushes" something, it's the automation doing this on its own — you don't have to do anything for that part.
- **Forum channel vs. regular (text) channel:** explained in the next section.

## Which template do I need?

|  | Notifies you about | Regular text channel | Forum channel |
|---|---|---|---|
| **Releases** | New GitHub Releases (version tags, with release notes) | `Repo_Release_Tracker_Template.yml` | `Repo_Release_Tracker_Template_Forum_Channel.yml` |
| **Commits** | New commits pushed to the default branch | `Repo_Commit_Tracker_Template.yml` | `Repo_Commit_Tracker_Template_Forum_Channel.yml` |

**Not sure which kind of Discord channel you have?** Look at the channel in Discord:

- If it looks like a normal chat — one continuous stream of messages — that's a **regular text channel**. Use the left column.
- If it shows a grid/list of separate titled "posts," each one opening into its own conversation, with a **"New Post"** button — that's a **Forum channel**. Use the right column.

Most Discord channels are regular text channels, so if you're not sure, that's the safer guess.

You can use more than one of these at the same time on the same repo — for example, releases going to a forum channel, and commits going to a regular text channel. Each one is set up separately, following the same steps below.

Everything in this README applies to all four template files unless a section specifically says "forum channel templates only." The [Forum channel behavior](#forum-channel-behavior) section covers everything that's different about the forum versions.

## Quick start (short version)

If you've done this kind of thing before, here's the condensed version — the full walkthrough with screen-by-screen detail is right after this:

1. Change the `name:` field and the `concurrency: group:` value at the top of the file so it doesn't collide with any other monitor workflow in the same repo.
2. Fill in the `env:` block under the `monitor` job (`REPO`, `PROJECT_NAME`, `STATE_FILE`, `PING_MODE`, `PING_ID`, and `THREAD_STATE_FILE` for forum-channel templates).
3. Add a `DISCORD_WEBHOOK` repository secret (repo **Settings → Secrets and variables → Actions → New repository secret**).
4. Commit the file to `.github/workflows/` in any repo you control — it doesn't have to be the repo you're monitoring.

## Full setup walkthrough

Take this step by step — none of it requires programming knowledge, just careful copy/paste.

### Step 1: Put the file where GitHub can run it

**This file needs to live inside a GitHub repo, in a specific folder, for GitHub to notice and run it.** It does **not** have to be the repo you're monitoring — you can create a small, separate repo just for this if you want, or add it to a repo you already have. All it needs is somewhere GitHub Actions can run and occasionally save a tiny state file.

If the `.github/workflows/` folder doesn't already exist in that repo, you'll need to create it. Two ways to do this:

- **Directly on GitHub.com (no software needed):** Open your repo in a browser → click **Add file → Create new file**. In the box where you'd normally type a filename, type the *entire path* including the folders, like this: `.github/workflows/release-monitor.yml`. As soon as you type each `/`, GitHub automatically creates that folder for you — you don't need to create `.github` and `workflows` as separate steps. Paste in the full contents of the template file, then scroll down and click **Commit changes**.
- **On your own computer, using git:** inside a local copy (clone) of the repo, run `mkdir -p .github/workflows` to create the folder, save the template file inside it (keep the `.yml` file ending), then run `git add .github/workflows/<filename>.yml`, `git commit -m "Add repo monitor"`, and `git push` to send it up to GitHub.

The exact filename you save it as inside that folder doesn't matter — `release-monitor.yml`, `otd-monitor.yml`, anything — as long as it ends in `.yml` and sits inside `.github/workflows/`.

### Step 2: Give it a unique name

Near the top of the file, you'll see two things to change:

- `name:` — this is just the label you'll see for this workflow in GitHub's **Actions** tab. Change it to something you'll recognize, e.g. `"winutil Release Monitor"`.
- `concurrency: group:` — this needs to be **unique** compared to any other monitor workflow you set up in the same repo. If you ever add a second monitor to the same repo and forget to give it a different value here, the two monitors will interfere with each other and randomly cancel each other's runs.

### Step 3: Fill in the settings block

Further down in the file, under `jobs: monitor: env:`, you'll find a block of settings to fill in. Here's what each one means:

| Setting | What it means |
|---|---|
| `REPO` | The GitHub repo you want to watch, written as `owner/repo-name`. For example, to watch `github.com/ChrisTitusTech/winutil`, you'd enter `ChrisTitusTech/winutil`. |
| `PROJECT_NAME` | Just a display name, used in the title of the Discord message — e.g. `"winutil"`. Purely cosmetic; call it whatever you like. |
| `STATE_FILE` | The name of the small bookmark file (explained above) this workflow keeps to remember the last release/commit it already told you about. The default name is fine unless you're adding a second monitor to the same repo — in that case, give each monitor its own filename so they don't overwrite each other's bookmark. You can put it in a subfolder if you like (e.g. `state/last_release.txt`) — the folder gets created automatically. |
| `THREAD_STATE_FILE` | *Forum-channel templates only.* A second, separate bookmark file, used only to recover if something goes wrong halfway through posting. Full explanation in [Forum channel behavior](#forum-channel-behavior) below — for now, just give it its own unique filename, same rule as `STATE_FILE`. |
| `PING_MODE` | Controls who gets notified/pinged when something new posts. Explained in detail in the next section. |
| `PING_ID` | The Discord ID of the specific person or role to ping (only needed for some `PING_MODE` settings). Explained in the next section. |
| `NOTIFY_ON_INITIAL_RUN` | `"true"` (the default) or `"false"`. Controls what happens the very first time this monitor ever runs — see [First run behavior](#first-run-behavior) below. |

### Step 4: Create and add the Discord webhook

This is how the workflow is actually able to post into your Discord channel.

1. In Discord, go to the channel you want notifications posted into. Open its settings (for a regular text channel: the gear icon, or right-click → **Edit Channel**; for a forum channel: right-click the channel → **Edit Channel**).
2. Go to **Integrations → Webhooks → New Webhook** (or **Create Webhook**).
3. Give it any name you like, then click **Copy Webhook URL**. This URL is the "password" that lets this workflow post messages — anyone who has it can post to that channel, so don't paste it anywhere public.

   > **Using a forum-channel template?** Make sure you create this webhook **on the Forum channel itself** — not on a regular text channel, and not inside one specific thread. A forum channel's webhook works a little differently under the hood, so if you accidentally use the wrong kind of webhook with the wrong kind of template, it will fail the very first time it tries to post (see [Troubleshooting](#troubleshooting) if that happens).

4. Now go back to your GitHub repo (the one you put the workflow file in) → **Settings → Secrets and variables → Actions → New repository secret**.
5. For the secret's name, type exactly `DISCORD_WEBHOOK`. For the value, paste the webhook URL you copied. Click **Add secret**.

That's it for the webhook — you never paste the URL directly into the workflow file itself, only into this secret, which keeps it hidden.

You do **not** need to do anything for `GITHUB_TOKEN` — GitHub creates and provides this automatically for every workflow run. This workflow uses it to talk to GitHub's API a bit more freely, and to save its own bookmark file back into the repo.

### Step 5: Turn it on and check it worked

Commit and push the file if you haven't already (or click **Commit changes** if you used the GitHub.com website in Step 1). Once it's saved, the workflow will start running automatically every 5 minutes, forever, without you needing to do anything else.

To check it's working right away, without waiting:

1. Go to your repo's **Actions** tab.
2. Click on the workflow's name in the list on the left.
3. Click the **Run workflow** button, then confirm.
4. After a few seconds, click into that run and open up its steps to read the log. You should see printed lines like `No new release.` (meaning it checked and there was nothing new — this is a completely normal, healthy result) or `New releases to report: 1` (meaning it found something and should have posted it to Discord).

A run with a green checkmark ✅ means everything worked. A red ✕ means something failed — see [Troubleshooting](#troubleshooting).

## Choosing who gets pinged

"Pinged" means Discord sends someone a notification, the same as if you typed `@username` in a message. `PING_MODE` controls whether that happens, and for whom.

Set `PING_MODE` to one of these four values:

| `PING_MODE` | What it does | Also set `PING_ID` to... |
|---|---|---|
| `user` (default) | Pings one specific person | Their Discord **user ID** (a long number — see below for how to get it) |
| `role` | Pings everyone who has a specific role | The **role ID** |
| `everyone` | Pings `@everyone` in the channel (notifies everyone who can see it) | *(leave `PING_ID` as-is, it's not used)* |
| `none` | Posts the update with no ping at all — just a normal message | *(leave `PING_ID` as-is, it's not used)* |

**How to find a Discord user ID or role ID:** In Discord, go to **User Settings → Advanced**, and turn on **Developer Mode**. Once that's on, you can right-click any person's name or any role and a new option appears: **Copy User ID** or **Copy Role ID**. Paste that number into `PING_ID`.

**A note on `everyone`:** this pings *everyone* who can see that channel, whether they're active right now or not — it's meant for a channel people already expect update pings in (like a dedicated `#releases` channel), not a general chat channel where it would be disruptive.

Only the *first* message of a batch gets a ping — so if five releases land at once because the monitor catches up on things it missed, you'll be pinged once, not five times. (More on this in [Catching up on multiple items](#catching-up-on-multiple-items) below.)

You don't need to edit any code for any of this — `PING_MODE` and `PING_ID` are the only two settings involved.

## How it behaves

This section explains what actually happens behind the scenes, so you know what to expect.

### Polling and state

"Polling" just means "checking in periodically to see if anything changed." Both templates poll every 5 minutes (GitHub's fastest allowed schedule for this kind of automation), and can also be run manually any time via **Actions → Run workflow**.

Every single time it runs, here's exactly what happens:

1. It asks GitHub's API: "what are the most recent releases/commits on this repo?"
2. It compares the newest one against what's saved in `STATE_FILE` (the bookmark file) — i.e., "is this something I've already told you about, or is it new?"
3. If nothing's changed since last time, it does nothing else and quietly finishes. **This is the normal result for almost every single run** — most of the time, nothing new has happened, and that's expected, not a sign anything is broken.
4. If something *is* new, it posts about it in Discord, then updates the bookmark file so it doesn't mention that same thing again next time.

One more detail: if, for some reason, a run is somehow still going when the next scheduled run starts (this shouldn't normally happen — a run usually finishes in a few seconds), GitHub will cancel the older one rather than letting two copies run at once and potentially conflict with each other.

### Catching up on multiple items

If this monitor is ever offline for a while (say, GitHub has an outage, or you paused the workflow), it doesn't just tell you about the single newest thing when it comes back — it catches up and tells you about *everything* it missed, oldest first, up to 10 items in one run.

If more than 10 things happened while it was away, the oldest ones beyond that limit are skipped (with a note saying how many were skipped) rather than flooding your channel with a wall of messages — you'll still always be told about the current latest one.

In one unusual edge case — extremely heavy activity, or a force-push that rewrites a repo's history — the last thing it remembers might no longer be found anywhere in the recent history at all. When that happens, it can't figure out exactly what was missed, so it just reports the single latest item along with a note explaining that it couldn't reconstruct the full gap, rather than guessing.

Only the very first item in a batch like this gets a ping and a note attached; the rest post quietly, so you're not pinged repeatedly.

**Behind the scenes, progress is saved after each item, not just once at the very end.** So if Discord or GitHub has a temporary hiccup partway through posting a batch of, say, 5 releases, whichever ones already posted successfully are remembered — the next run will pick up from there instead of accidentally posting those same ones again.

### Forum channel behavior

*(This section only applies to the two forum-channel templates.)* Everything described above still applies — the one difference is **where things get posted**: instead of dropping a message into a shared, ongoing channel, **each new release or commit gets posted as its own new forum post (a "thread")**.

- Each thread's title looks like `{PROJECT_NAME} Updated! — {tag}` for releases, or `{PROJECT_NAME} Updated! — Commit {short commit ID}` for commits. (Forum post titles have a 100-character limit in Discord, so an unusually long one gets shortened automatically.)
- If one release or commit's notes are too long to fit in a single message, the extra "(continued)" messages are posted as replies *inside that same thread* — they don't create new, separate posts.
- If several new releases or commits show up in one run, **each one gets its own separate thread** — three new releases means three new forum posts, not one post containing three messages. As above, only the first (oldest) one in that batch gets a ping.

**About `THREAD_STATE_FILE` and why you usually won't see it in your repo — this is expected, not a problem:**

This file is a temporary safety net, not a permanent record. Here's exactly when it exists and when it doesn't:

- While a single release or commit is in the middle of being posted (say, its notes need 3 separate messages), this file briefly holds a note saying "I'm partway through posting this one, here's exactly where I left off."
- The moment that item finishes posting completely — even if it only ever needed one message — this file is automatically deleted again, before the run ends.
- It is only left behind, and only shows up as a real file in your repo, if a run genuinely fails or errors out *in the middle* of posting a long item (for example, Discord has a temporary outage right as it's sending message 2 of 3). In that case, the next run reads this file and continues posting into the *same* thread exactly where it left off — instead of starting a confusing duplicate thread for the same release.

**In short: if you look in your repo and don't see this file, that's the normal, healthy state — it means every release/commit so far has posted completely successfully.** You'd only expect to actually see this file sitting there if a previous run failed partway through, and even then, it should disappear again as soon as a later run finishes posting that item successfully.

### First run behavior

The very first time this monitor ever runs, there's no bookmark yet (`STATE_FILE` doesn't exist), so it has nothing to compare against. By default, it treats whatever the *current* latest release/commit is as "new" and posts a notification about it, ping included.

This is handy as a quick way to confirm everything is wired up correctly. But if you're adding this to a repo that already has a long history of releases, it does mean you'll get pinged once about something that isn't actually new — it already existed before you set this up.

If you'd rather skip that, set `NOTIFY_ON_INITIAL_RUN` to `"false"`. With that set, the very first run will silently save the current latest release/commit as its starting bookmark, without posting anything — and you'll only start getting notified about things that happen *after* that point.

### Message length and splitting

Discord messages have a length limit. The ping, title, and author/link details are sent together as one message along with as much of the release notes or commit message as will fit. If there's more content than fits in one message, it continues into additional "(continued)" messages — and it's careful to only ever break between whole lines, so you'll never see a bullet point, a link, or a heading get awkwardly cut off in the middle. (Only in the rare case of one single, absurdly long line with no line breaks at all does it fall back to breaking mid-line — but even then, it never splits in the middle of a single word.)

### Reliability

In plain terms: this workflow is built to quietly recover from small, temporary network problems on its own, rather than immediately giving up and failing.

- If a network request times out or briefly fails to connect, it automatically tries again a few times, waiting a little longer between each attempt, before giving up.
- Both GitHub and Discord can sometimes respond with "you're sending requests too fast, slow down" (this is called a rate limit, HTTP error code 429) — this can happen if a single run needs to send several messages in a row. When that happens, the workflow automatically waits the amount of time it's told to, then tries again, up to 5 times, instead of just failing.
- To help avoid triggering that "too fast" response in the first place, it also deliberately waits a fraction of a second between each Discord message it sends.
- Any *other* kind of error (like an invalid webhook URL, or a repo that doesn't exist) is treated as a real problem, not a temporary glitch — it's **not** retried, and the run fails immediately so you'll see it clearly in the Actions tab rather than it silently retrying forever.

### Links in release notes / commit messages

Some projects write plain issue/PR references like `#123` in their release notes; others (especially auto-generated changelogs) write the full web address instead, like `https://github.com/owner/repo/pull/123`. GitHub turns both of these into clickable links automatically on its own website — but Discord does neither on its own. So this workflow converts both styles into proper clickable Discord links before posting, so they work no matter which style the original text used. If something is already written as a clickable link, it's left alone rather than being turned into a broken double-link.

### Inactivity reminder

This is unrelated to releases or commits — it's a safety net for a separate GitHub rule: **if a repo goes 60 days with absolutely no commits pushed to it, GitHub automatically disables any scheduled workflows in that repo**, this one included. That can happen silently, with no warning from GitHub — you'd typically only notice once you realize notifications have quietly stopped.

This matters most if you followed the suggestion in Step 1 to put this workflow in its own small, dedicated repo rather than one that already gets commits for other reasons. A repo used for nothing but hosting this workflow only ever gets a new commit when the workflow itself finds something new to report — so if the project you're monitoring goes quiet for a couple of months, that hosting repo could genuinely rack up 60 days of silence and get shut off without you ever being told.

To guard against that, every single run also quietly checks one extra thing: "has it been at least 59 days since the last commit to this repo?" If so — and only if you haven't already been sent this particular reminder — it posts a one-time Discord message titled **"GitHub Workflow Re-enable Reminder"**, pinging whoever you've already configured via `PING_MODE`/`PING_ID`.

**Important: this message can't actually stop GitHub from disabling the workflow** — that decision is entirely GitHub's, made outside this workflow, and nothing a workflow run does from the inside can prevent it. What this message *does* do is warn you a day ahead of time, so once the workflow does get disabled, it isn't a surprise — you'll already know to go to your repo's **Actions** tab, click the workflow, and click **Re-enable workflow**, which starts everything running again immediately and takes about ten seconds.

A few more details worth knowing:

- You won't be spammed with this every 5 minutes once the 59-day mark passes — it's remembered (using GitHub's own Actions cache), so you'll only be told about a given quiet stretch once, not on every single run afterward.
- Any new commit to the repo — including this workflow's own bookmark-file update, the moment the thing you're monitoring becomes active again — resets the clock automatically. You won't hear about it again unless another full 59 days of total silence pass.
- If the repo stays inactive even after you re-enable it, expect to be reminded again roughly every 59 days for as long as that remains true.
- This is fully automatic — there's nothing extra to set up or configure. It reuses the same `PING_MODE` and `PING_ID` settings you already filled in for regular notifications.

## Troubleshooting

**Nothing posted, and there's no error in the Actions tab.** This is very likely completely normal — most runs find nothing new, since most 5-minute windows don't have a new release or commit in them. Open the run's log and look for a line like `No new release.` or `No new commit.` — if you see that, everything is working correctly. If you expected something to have posted and didn't, double-check that `REPO` is spelled exactly right (`owner/repo-name`, and capitalization matters), and that the release/commit genuinely exists on the repo's default (main) branch — the commit monitor only ever watches the default branch.

**The workflow doesn't seem to run on its own schedule at all.** GitHub automatically disables scheduled workflows in a repo that's had no commit activity for 60 days. If that's happened, go re-enable it manually from the Actions tab — these templates try to warn you about this a day in advance over Discord (see [Inactivity reminder](#inactivity-reminder) above), so you shouldn't be caught completely off guard, but the fix either way is the same: **Actions tab → the workflow → Re-enable workflow**. Scheduled runs can also sometimes be delayed by several minutes when GitHub itself is under heavy load — that's a limitation on GitHub's end, not something this workflow controls.

**Discord shows an error, or nothing shows up in the channel.** First, double check the `DISCORD_WEBHOOK` secret is set on the *same repo* the workflow file lives in (Settings → Secrets and variables → Actions). Also check that the webhook still exists and hasn't been deleted or regenerated on the Discord side — regenerating a webhook changes its URL, which would break it. The run's log will show you the exact error Discord sent back, which usually explains exactly what's wrong.

**A forum-channel template fails, especially with an error mentioning `thread_name`.** This almost always means the `DISCORD_WEBHOOK` secret is pointing at the wrong kind of channel. Forum-channel templates need a webhook that was created *on a Forum channel specifically* — using a regular text channel's webhook with a forum-channel template (or vice versa) will fail immediately on the very first post. Go back to Discord, create a new webhook on the correct channel type, and update the secret with the new URL.

**The workflow fails specifically at the "Save monitor state" step, and `git push` is the part that errors.** This usually means the repo has branch protection rules that block direct pushes to the main branch, even from GitHub's own automation. Either loosen that rule specifically for the `github-actions[bot]` account, or use a separate, unprotected repo just for this workflow's bookmark file.

**You want to see a notification you've already gotten again, to test something.** Open `STATE_FILE` in your repo, and either delete its contents, or change it to an older release tag / commit ID than the current one. The next run will then treat the current latest one as "new" again and post about it.

## Security & permissions notes

- This workflow only asks GitHub for permission to read its own repo and write (commit) back to it — nothing more. It cannot touch issues, pull requests, other repos, or your account settings.
- That permission only applies to the repo the *workflow file itself* lives in — not necessarily the repo you're monitoring. This matters if you're trying to monitor a *different*, *private* repo than the one hosting the workflow — see the FAQ below for what to do in that case.
- Settings like `PING_ID`, `REPO`, and `PROJECT_NAME` are stored as plain, visible text in the workflow file — anyone who can view your repo can see them. Only `DISCORD_WEBHOOK` needs to be kept private, which is exactly why it's the one thing stored as a secret instead of typed directly into the file.

## FAQ

**Can I use this to monitor a private repo?** Only if it's the *same* repo you put the workflow file into — the automatic permission GitHub gives the workflow only covers its own repo. To watch a *different* private repo, you'd need to create a personal access token with read access to that repo, add it as an additional secret, and use it in place of the automatic one in the script. Public repos can always be monitored, regardless of which repo hosts the workflow.

**Can I set up more than one of these on the same repo?** Yes — any combination of the four templates. Just make sure each one has its own unique `name`, `concurrency: group`, `STATE_FILE`, and (for forum-channel templates) `THREAD_STATE_FILE`, as covered in the setup steps above. If two monitors accidentally share any of those, they'll interfere with each other.

**Can I use a forum-channel template with a regular text channel, or the other way around?** No. A Discord webhook is tied to one specific channel of one specific type, and the two kinds of templates talk to Discord differently. Use whichever template matches the channel you actually have — see the table near the top of this README.

**Can I make it check more or less often than every 5 minutes?** Yes. Find the `cron:` line under `on: schedule:` near the top of the file and change it. 5 minutes is the fastest GitHub allows; you can make it check less often if you don't need near-instant notifications — e.g. `*/15 * * * *` for every 15 minutes, or `0 * * * *` for once an hour.

**How do I pause or completely turn this off?** To pause it temporarily: go to the **Actions** tab, click the workflow, click the **⋯** menu, and choose **Disable workflow**. You can turn it back on the same way, whenever you like. To remove it permanently, just delete the workflow file from `.github/workflows/`.

**Will this notify me about draft releases or prereleases?** No, not by default — it only notifies about a repo's actual "latest release," the same definition GitHub itself uses, which excludes drafts and prereleases. If you specifically want prereleases included too, that requires a small edit inside the script itself (removing one filter condition) — it's not something you can turn on just by changing a setting.

**Will I hit GitHub's rate limits by running this?** No, in virtually all normal cases. GitHub limits how many API requests you can make without logging in to 60 per hour, but this workflow automatically authenticates every request using the token GitHub provides for free, which raises that limit dramatically — high enough that you'd need to be running an unusually large number of these monitors from a single repo before it would ever become a concern.

**Why did I get a Discord message titled "GitHub Workflow Re-enable Reminder"?** That's expected — not an error, and nothing is broken. It means the repo hosting this workflow hasn't had a commit in about 59 days, which is right before GitHub's own 60-day cutoff for automatically disabling scheduled workflows. See [Inactivity reminder](#inactivity-reminder) above for the full explanation; the short version is: check the **Actions** tab in a day or so, and if the workflow has been disabled, click **Re-enable workflow**.
