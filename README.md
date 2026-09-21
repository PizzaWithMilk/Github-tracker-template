# Repo Monitor Templates

## What this actually does

These are ready-to-use automation files for GitHub. Once you set one up, it quietly checks one or more GitHub repos (projects) every 5 minutes, and if something new shows up — a new release or a new commit — it automatically posts a message about it in a Discord channel. You don't need to run anything on your own computer; GitHub runs it for you, on a timer, forever (or until you turn it off). Which repos it watches, and how notifications for each one behave, is controlled by a small separate config file — see [Monitoring multiple repositories](#monitoring-multiple-repositories). It also watches out for a separate GitHub quirk that can silently turn it off on its own — see [Inactivity reminder](#inactivity-reminder) below.

There are four template files. You'll use one or two of them, not all four — the table below helps you pick.

## A few words explained, before we start

If you already know GitHub Actions and Discord webhooks well, skip this part. If you're newer to this, these quick definitions will make the rest of this README much easier to follow:

- **Repo (repository):** a project's folder of files on GitHub.
- **Workflow:** an automation recipe GitHub runs for you. It's a text file ending in `.yml`, and it lives in a special folder called `.github/workflows/` inside a repo. The files in this README *are* workflows.
- **Config file:** a separate small text file, `config/repos.yml`, that lists which repo(s) to watch and who to ping. It's not a workflow itself — it's data the workflow reads. Explained in full in [Step 2](#step-2-create-your-reposyml-config-file).
- **Webhook:** a special web address (URL) that Discord gives you for one specific channel. Any program that sends a message to that address will have its message posted in that channel — no bot, no login, no password needed on Discord's side. Anyone who has the URL can post to your channel with it, so it's treated like a secret.
- **Secret (in GitHub):** a private value — like a password or webhook URL — that you store in your repo's settings. Workflows can use it, but it's never shown in logs and no one browsing your repo can see its value once it's saved.
- **State directory:** a small folder this workflow keeps inside your repo, purely to remember "the last thing I already told you about" — one small bookmark file per repo you're watching, named automatically. You never need to create or edit these files yourself.
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

Each template file can watch **more than one repository at once** — see [Monitoring multiple repositories](#monitoring-multiple-repositories) below. You can also run more than one of these template *files* at the same time — for example, releases going to a forum channel, and commits going to a regular text channel — and they can even share the same config file; see [Sharing one repos.yml across templates](#sharing-one-reposyml-across-templates).

Everything in this README applies to all four template files unless a section specifically says "forum channel templates only." The [Forum channel behavior](#forum-channel-behavior) section covers everything that's different about the forum versions.

## Quick start (short version)

If you've done this kind of thing before, here's the condensed version — the full walkthrough with screen-by-screen detail is right after this:

1. Change the `name:` field and the `concurrency: group:` value at the top of the file so it doesn't collide with any other monitor workflow in the same repo — including another copy of the same template, or its forum/non-forum counterpart, since they default to the same group name.
2. Create a `config/repos.yml` file listing the repo(s) you want to watch, your default ping settings, and (optionally) per-repo overrides — see [Monitoring multiple repositories](#monitoring-multiple-repositories).
3. Fill in the workflow's own `env:` block: `CONFIG_FILE` (only if you put `repos.yml` somewhere other than the default path), `STATE_DIR`, and `NOTIFY_ON_INITIAL_RUN` as needed.
4. Add a `DISCORD_WEBHOOK` repository secret (repo **Settings → Secrets and variables → Actions → New repository secret**).
5. Commit both files to the repo — the workflow to `.github/workflows/`, and `repos.yml` wherever `CONFIG_FILE` points (`config/repos.yml` by default).

## Full setup walkthrough

Take this step by step — none of it requires programming knowledge, just careful copy/paste.

### Step 1: Put the file where GitHub can run it

**This file needs to live inside a GitHub repo, in a specific folder, for GitHub to notice and run it.** It does **not** have to be the repo you're monitoring — you can create a small, separate repo just for this if you want, or add it to a repo you already have. All it needs is somewhere GitHub Actions can run and occasionally save a tiny state file.

If the `.github/workflows/` folder doesn't already exist in that repo, you'll need to create it. Two ways to do this:

- **Directly on GitHub.com (no software needed):** Open your repo in a browser → click **Add file → Create new file**. In the box where you'd normally type a filename, type the *entire path* including the folders, like this: `.github/workflows/release-monitor.yml`. As soon as you type each `/`, GitHub automatically creates that folder for you — you don't need to create `.github` and `workflows` as separate steps. Paste in the full contents of the template file, then scroll down and click **Commit changes**.
- **On your own computer, using git:** inside a local copy (clone) of the repo, run `mkdir -p .github/workflows` to create the folder, save the template file inside it (keep the `.yml` file ending), then run `git add .github/workflows/<filename>.yml`, `git commit -m "Add repo monitor"`, and `git push` to send it up to GitHub.

The exact filename you save it as inside that folder doesn't matter — `release-monitor.yml`, `otd-monitor.yml`, anything — as long as it ends in `.yml` and sits inside `.github/workflows/`.

### Step 2: Create your repos.yml config file

This is where you tell the workflow which repo(s) to watch and who to ping — it's a separate file from the workflow itself.

1. In the same repo you put the workflow file in, create a new file at `config/repos.yml` (same two methods as Step 1 — type the full path including the `config/` folder when creating it on GitHub.com, or `mkdir -p config` locally).
2. Fill it in following this shape:

```yaml
ping_mode: user                          # user | role | everyone | none
ping_id: "YOUR_DISCORD_USER_ID_HERE"     # required only when ping_mode is user or role

repos:
  - repo: OWNER/REPO
    type: normal                          # normal | forum - must match the kind of channel this template posts to
```

`ping_mode` and `ping_id` at the top are the *defaults* — used by every repo below that doesn't set its own. See [Choosing who gets pinged](#choosing-who-gets-pinged) for what each `ping_mode` value does, and [Monitoring multiple repositories](#monitoring-multiple-repositories) for the full list of fields a repo block can have (per-repo ping overrides, Forum tags, and — importantly — how one `repos.yml` can be shared across several of these templates at once).

Every repo block needs, at minimum, `repo:` (written as `OWNER/REPO`, and it has to be the *first* field in the block) and `type:` (`normal` for a regular text-channel template, `forum` for a Forum-channel one — this has to match whichever template *this particular workflow file* is). There's no field for a custom display name — Discord will show the repo's own name (the part after the last `/`), e.g. `owner/winutil` is shown as `winutil`.

3. If `CONFIG_FILE` in the workflow's `env:` block still says `config/repos.yml` (the default), you don't need to change anything else in the workflow file — just make sure `repos.yml` actually lives at that exact path in the repo.

### Step 3: Fill in the workflow's settings block

Further down in the file, under `jobs: monitor: env:`, you'll find a small block of settings:

| Setting | What it means |
|---|---|
| `CONFIG_FILE` | Path to the `repos.yml` file you created in Step 2, relative to the root of this repo. The default, `config/repos.yml`, is fine as long as you put the file there — change this only if you saved it somewhere else, or you want this particular workflow to read a *different* config file than another monitor in the same repo (see [Sharing one repos.yml across templates](#sharing-one-reposyml-across-templates)). |
| `STATE_DIR` | The folder (inside this repo) where the workflow keeps its bookmark files — one small file per repo listed in `repos.yml`, named automatically, so you never have to think about individual filenames. The default, `.github/monitor-state`, is fine for almost everyone; change it only if you specifically want these files stored somewhere else. |
| `NOTIFY_ON_INITIAL_RUN` | `"true"` (the default) or `"false"`. Controls what happens the first time this monitor checks a given repo — see [First run behavior](#first-run-behavior) below. |

### Step 4: Create and add the Discord webhook

This is how the workflow is actually able to post into your Discord channel.

1. In Discord, go to the channel you want notifications posted into. Open its settings (for a regular text channel: the gear icon, or right-click → **Edit Channel**; for a forum channel: right-click the channel → **Edit Channel**).
2. Go to **Integrations → Webhooks → New Webhook** (or **Create Webhook**).
3. Give it any name you like, then click **Copy Webhook URL**. This URL is the "password" that lets this workflow post messages — anyone who has it can post to that channel, so don't paste it anywhere public.

   > **Using a forum-channel template?** Make sure you create this webhook **on the Forum channel itself** — not on a regular text channel, and not inside one specific thread. A forum channel's webhook works a little differently under the hood, so if you accidentally use the wrong kind of webhook with the wrong kind of template, it will fail the very first time it tries to post (see [Troubleshooting](#troubleshooting) if that happens).

4. Now go back to your GitHub repo (the one you put the workflow file in) → **Settings → Secrets and variables → Actions → New repository secret**.
5. For the secret's name, type exactly `DISCORD_WEBHOOK`. For the value, paste the webhook URL you copied. Click **Add secret**.

That's it for the webhook — you never paste the URL directly into the workflow file itself, only into this secret, which keeps it hidden.

You do **not** need to do anything for `GITHUB_TOKEN` — GitHub creates and provides this automatically for every workflow run. This workflow uses it to talk to GitHub's API a bit more freely, and to save its own bookmark files back into the repo.

### Step 5: Turn it on and check it worked

Commit and push both files if you haven't already (or click **Commit changes** if you used the GitHub.com website). Once they're saved, the workflow will start running automatically every 5 minutes, forever, without you needing to do anything else.

To check it's working right away, without waiting:

1. Go to your repo's **Actions** tab.
2. Click on the workflow's name in the list on the left.
3. Click the **Run workflow** button, then confirm.
4. After a few seconds (a bit longer if you're watching several repos, or a rate limit briefly kicks in), click into that run and open up its steps to read the log. You should see printed lines like `No new release.` (meaning it checked and there was nothing new — this is a completely normal, healthy result) or `New releases to report for OWNER/REPO: 1` (meaning it found something and should have posted it to Discord). If you listed several repos in `repos.yml`, you'll see a block of lines like this for *each matching* repo, one after another, in the same run.

A run with a green checkmark ✅ means everything worked. A red ✕ means something failed — see [Troubleshooting](#troubleshooting).

## Monitoring multiple repositories

Every repo you want to watch goes in the `repos:` list inside `config/repos.yml` (see [Step 2](#step-2-create-your-reposyml-config-file)):

```yaml
ping_mode: user
ping_id: "YOUR_DISCORD_USER_ID_HERE"

repos:
  - repo: ChrisTitusTech/winutil
    type: normal

  - repo: torvalds/linux
    type: normal

  - repo: someuser/some-other-repo
    type: normal
```

Each repo block needs `repo:` and `type:` at minimum. `type` has to be either `normal` or `forum`, and has to match the kind of template this particular workflow file is — see [Sharing one repos.yml across templates](#sharing-one-reposyml-across-templates) below if you want to mix both kinds using one shared file.

**Each repo can also override the ping settings just for that one repo**, with `ping:` and `ping_id:`:

```yaml
repos:
  - repo: ChrisTitusTech/winutil
    type: normal
    ping: role
    ping_id: 123456789012345678

  - repo: torvalds/linux
    type: normal
    ping: everyone

  - repo: someuser/some-other-repo
    type: normal
```

Any repo that leaves `ping:`/`ping_id:` off just uses `repos.yml`'s top-level `ping_mode`/`ping_id` instead — so you only need to add these for repos that should behave differently. `everyone` and `none` don't need a `ping_id` at all, so you can leave it out entirely (as with the `torvalds/linux` block above). See [Choosing who gets pinged](#choosing-who-gets-pinged) below for what each mode does.

*(Forum-channel templates only: there's also an optional `tags:` list for applying Forum tags to that repo's posts. See [Forum channel behavior](#forum-channel-behavior) below.)*

A few more things worth knowing about how this works:

- **Pinging happens per repo, not once for the whole run.** If three of your listed repos each happen to have something new in the same 5-minute check, each one pings independently — using its own ping settings, whether that's the config's default or a per-repo override — so you'd get three pings, not one shared ping for the whole run. Within a single repo's own batch of catch-up items, only the first one still gets pinged; see [Catching up on multiple items](#catching-up-on-multiple-items).
- Each repo's bookmark file (inside `STATE_DIR`) is completely independent, so one repo catching up on a backlog of missed items has no effect on any other repo in the list.
- If one repo in the list is broken somehow (renamed, deleted, a typo, or a repeated API error), it won't stop the *other* repos from being checked and posted about in that same run — but the overall run will still show as failed (a red ✕) in the Actions tab so you notice. Check the run's log for a line starting with `Error while monitoring` to see exactly which repo it was.
- One exception: if `repos.yml` itself is malformed — a missing `/` in a repo name, an invalid `type` or `ping` value, a `user`/`role` ping with no `ping_id` given, or a setting placed somewhere it isn't allowed, for example — the *entire* run fails immediately, before any repo gets checked at all. See [Troubleshooting](#troubleshooting).

There's no field for a custom display name — Discord always shows the repo's own name (the part after the last `/`) as the project name in the message.

If you'd rather keep a repo fully separate from the rest, either give its workflow its own `CONFIG_FILE` pointed at a different file (see Step 4), or set up a whole separate copy of the workflow with its own `STATE_DIR`, `name`, and `concurrency: group:`.

## Sharing one repos.yml across templates

Every repo block in `repos.yml` needs a `type:` — `normal` or `forum` — and each workflow file only pays attention to the repos whose `type` matches the kind of channel *it* posts to. A non-forum template's run skips every `type: forum` block entirely, as if it weren't there; a forum-channel template does the reverse.

This means **all four template files can point at the exact same `config/repos.yml`** (the default `CONFIG_FILE` setting already matches across all of them), and each repo automatically goes to the right kind of channel:

```yaml
repos:
  - repo: owner/project-one
    type: normal      # picked up by the non-forum templates only

  - repo: owner/project-two
    type: forum        # picked up by the forum-channel templates only
```

A few things worth being clear about:

- **`type` doesn't decide releases vs. commits — which templates you deploy does.** If you set up both `Repo_Release_Tracker_Template.yml` and `Repo_Commit_Tracker_Template.yml` pointed at the same config, every `type: normal` repo in it gets *both* release notifications and commit notifications. If you only want one or the other for a given repo, only deploy the matching template(s) — `type` alone can't turn tracking off.
- You don't have to share a config file at all. If you'd rather keep things fully separate — say, a different set of repos for releases than for commits — just point each template's `CONFIG_FILE` setting at its own file instead (see Step 4).
- A `type: forum` repo will make a forum-channel template try to post there — and it'll fail (see [Troubleshooting](#troubleshooting)) if that template's `DISCORD_WEBHOOK` isn't actually a Forum-channel webhook. `type` tells a workflow which repos to *handle*; it doesn't change which channel that workflow's own webhook actually points to.

## Choosing who gets pinged

"Pinged" means Discord sends someone a notification, the same as if you typed `@username` in a message. `ping_mode` in `config/repos.yml` controls whether that happens, and for whom.

Set `ping_mode` (at the top of `config/repos.yml`, or per-repo — see [Monitoring multiple repositories](#monitoring-multiple-repositories)) to one of these four values:

| `ping_mode` | What it does | Also set `ping_id` to... |
|---|---|---|
| `user` (default) | Pings one specific person | Their Discord **user ID** (a long number — see below for how to get it) |
| `role` | Pings everyone who has a specific role | The **role ID** |
| `everyone` | Pings `@everyone` in the channel (notifies everyone who can see it) | *(not needed — leave it out)* |
| `none` | Posts the update with no ping at all — just a normal message | *(not needed — leave it out)* |

**How to find a Discord user ID or role ID:** In Discord, go to **User Settings → Advanced**, and turn on **Developer Mode**. Once that's on, you can right-click any person's name or any role and a new option appears: **Copy User ID** or **Copy Role ID**. Paste that number into `ping_id` (quoting it, like `ping_id: "123456789012345678"`, is a safe habit for the top-level default, though not required).

**A note on `everyone`:** this pings *everyone* who can see that channel, whether they're active right now or not — it's meant for a channel people already expect update pings in (like a dedicated `#releases` channel), not a general chat channel where it would be disruptive.

The top-level `ping_mode`/`ping_id` in `config/repos.yml` are the *defaults* — every repo uses these unless its own block overrides them. See [Monitoring multiple repositories](#monitoring-multiple-repositories) above for that per-repo override format.

Only the *first* message of a batch gets a ping — so if five releases land at once for one repo because the monitor catches up on things it missed, you'll be pinged once for that repo, not five times. If you're watching multiple repos, each one pings independently on its own terms — see [Monitoring multiple repositories](#monitoring-multiple-repositories) for the full picture. (More on batches in [Catching up on multiple items](#catching-up-on-multiple-items) below.)

You don't need to edit any code for any of this — `ping_mode` and `ping_id`, in `config/repos.yml`, are the only settings involved.

## How it behaves

This section explains what actually happens behind the scenes, so you know what to expect.

### Polling and state

"Polling" just means "checking in periodically to see if anything changed." Every template polls every 5 minutes (GitHub's fastest allowed schedule for this kind of automation), and can also be run manually any time via **Actions → Run workflow**.

Every single time it runs, here's exactly what happens, for each matching repo in `config/repos.yml` (see [Sharing one repos.yml across templates](#sharing-one-reposyml-across-templates)) in turn:

1. It asks GitHub's API: "what are the most recent releases/commits on this repo?"
2. It compares the newest one against what's saved in that repo's bookmark file inside `STATE_DIR` — i.e., "is this something I've already told you about, or is it new?"
3. If nothing's changed since last time, it does nothing else for that repo and moves on. **This is the normal result for almost every single run** — most of the time, nothing new has happened, and that's expected, not a sign anything is broken.
4. If something *is* new, it posts about it in Discord, then updates that repo's bookmark file so it doesn't mention that same thing again next time.

One more detail: if, for some reason, a run is somehow still going when the next scheduled run starts (this shouldn't normally happen — a run usually finishes in well under a minute), GitHub will cancel the older one rather than letting two copies run at once and potentially conflict with each other. If that cancellation happens to land in the middle of a run, whichever items it hadn't finished saving state for yet may get reported again on the next run — a rare edge case, but worth knowing about if you're watching an unusually large number of repos.

### Catching up on multiple items

If this monitor is ever offline for a while (say, GitHub has an outage, or you paused the workflow), it doesn't just tell you about the single newest thing when it comes back — it catches up and tells you about *everything* it missed for that repo, oldest first, up to 10 items in one run. This applies independently to each repo in your config — one repo catching up on a large backlog doesn't affect the 10-item cap for any other repo.

If more than 10 things happened while it was away, the oldest ones beyond that limit are skipped (with a note saying how many were skipped) rather than flooding your channel with a wall of messages — you'll still always be told about the current latest one.

In one unusual edge case — extremely heavy activity, or a force-push that rewrites a repo's history — the last thing it remembers might no longer be found anywhere in the recent history at all. When that happens, it can't figure out exactly what was missed, so it just reports the single latest item along with a note explaining that it couldn't reconstruct the full gap, rather than guessing.

Only the very first item in a batch like this gets a ping and a note attached; the rest post quietly, so you're not pinged repeatedly.

**Behind the scenes, progress is saved after each item, not just once at the very end.** So if Discord or GitHub has a temporary hiccup partway through posting a batch of, say, 5 releases, whichever ones already posted successfully are remembered — the next run will pick up from there instead of accidentally posting those same ones again.

### Forum channel behavior

*(This section only applies to the two forum-channel templates.)* Everything described above still applies — the one difference is **where things get posted**: instead of dropping a message into a shared, ongoing channel, **each new release or commit gets posted as its own new forum post (a "thread")**.

- Each thread's title looks like `{Project Name} Updated! — {tag}` for releases, or `{Project Name} Updated! — Commit {short commit ID}` for commits. (Forum post titles have a 100-character limit in Discord, so an unusually long one gets shortened automatically.)
- If one release or commit's notes are too long to fit in a single message, the extra "(continued)" messages are posted as replies *inside that same thread* — they don't create new, separate posts.
- If several new releases or commits show up in one run, **each one gets its own separate thread** — three new releases means three new forum posts, not one post containing three messages. As above, only the first (oldest) one in that batch gets a ping.

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

**About the thread-resume files, and why you usually won't see them in your repo — this is expected, not a problem:**

Alongside each repo's regular bookmark file, `STATE_DIR` also holds a small, temporary resume file for each repo in `config/repos.yml` (named automatically) — a safety net, not a permanent record, used only to recover if something goes wrong halfway through posting a long item.

- While a single release or commit is in the middle of being posted (say, its notes need 3 separate messages), this file briefly holds a note saying "I'm partway through posting this one — here's exactly which thread, and where I left off."
- The moment that item finishes posting completely — even if it only ever needed one message — this file is automatically deleted again, before the run ends.
- It is only left behind, and only shows up as a real file in your repo, if a run genuinely fails or errors out *in the middle* of posting a long item (for example, Discord has a temporary outage right as it's sending message 2 of 3). In that case, the next run reads this file and continues posting into the *same* thread exactly where it left off — instead of starting a confusing duplicate thread for the same release.

**In short: if `STATE_DIR` only contains the regular bookmark files, that's the normal, healthy state — it means every release/commit so far has posted completely successfully.** You'd only expect to see one of these resume files sitting there if a previous run failed partway through, and even then, it should disappear again as soon as a later run finishes posting that item successfully.

The [Inactivity reminder](#inactivity-reminder) below posts as its own brand-new Forum thread when it fires — it never needs a resume file, since it always fits in a single message.

### First run behavior

The first time this monitor checks a *given* repo — whether that's because the whole workflow is brand new, or because you just added a repo to `config/repos.yml` that it hasn't seen before — there's no bookmark for that repo yet, so it has nothing to compare against. By default, it treats whatever that repo's *current* latest release/commit is as "new" and posts a notification about it, ping included (subject to the once-per-repo ping rule described above).

This is handy as a quick way to confirm everything is wired up correctly. But if you're adding an established repo with a long release/commit history, it does mean you'll get a notification about something that isn't actually new — it already existed before you added it.

If you'd rather skip that, set `NOTIFY_ON_INITIAL_RUN` to `"false"`. With that set, the first time any given repo is checked, it will silently save that repo's current latest release/commit as its starting bookmark, without posting anything — you'll only start getting notified about things that happen *after* that point, for that repo. Since this is decided per repo, it also covers any repo you add to your config later, not just the very first run of the whole workflow.

### Message length and splitting

Discord messages have a length limit. The ping, title, and author/link details are sent together as one message along with as much of the release notes or commit message as will fit. If there's more content than fits in one message, it continues into additional "(continued)" messages — and it's careful to only ever break between whole lines, so you'll never see a bullet point, a link, or a heading get awkwardly cut off in the middle. (Only in the rare case of one single, absurdly long line with no line breaks at all does it fall back to breaking mid-line — but even then, it never splits in the middle of a single word.)

### Reliability

In plain terms: this workflow is built to quietly recover from small, temporary problems on its own, rather than immediately giving up and failing.

- If a network request times out or briefly fails to connect, it automatically tries again a few times, waiting a little longer between each attempt, before giving up.
- Both GitHub and Discord can sometimes respond with "you're sending requests too fast, slow down" (this is called a rate limit, HTTP error code 429) — this can happen if a single run needs to send several messages in a row, especially when watching several repos at once. When that happens, the workflow automatically waits the amount of time it's told to (capped at 15 seconds per wait, so it never stalls indefinitely), then tries again, up to 5 times, instead of just failing.
- To help avoid triggering that "too fast" response in the first place, it also deliberately waits at least half a second between every single Discord message it sends — including between messages for entirely different repos in the same run.
- Any *other* kind of error (like an invalid webhook URL, or a repo that doesn't exist) is treated as a real problem, not a temporary glitch — it's **not** retried, and the run fails immediately so you'll see it clearly in the Actions tab rather than it silently retrying forever. When watching multiple repos, one repo hitting this kind of error doesn't stop the others from being checked (see [Monitoring multiple repositories](#monitoring-multiple-repositories)) — but it does mean that run's overall result still shows as failed, so you notice.
- If you run more than one of these monitor workflows in the same repo (see the FAQ), it's normal for two of them to occasionally try to save their state at almost the same moment. Saving state automatically retries a few times if that happens, so a timing collision like that doesn't fail the run or lose either workflow's progress.

## Links in release notes / commit messages

Some projects write plain issue/PR references like `#123` in their release notes; others (especially auto-generated changelogs) write the full web address instead, like `https://github.com/owner/repo/pull/123`. GitHub turns both of these into clickable links automatically on its own website — but Discord does neither on its own. So this workflow converts both styles into proper clickable Discord links before posting, so they work no matter which style the original text used. If something is already written as a clickable link, it's left alone rather than being turned into a broken double-link.

## Inactivity reminder

This is unrelated to releases or commits — it's a safety net for a separate GitHub rule: **if a repo goes 60 days with absolutely no commits pushed to it, GitHub automatically disables any scheduled workflows in that repo**, this one included. That can happen silently, with no warning from GitHub — you'd typically only notice once you realize notifications have quietly stopped.

This matters most if you followed the suggestion in Step 1 to put this workflow in its own small, dedicated repo rather than one that already gets commits for other reasons. A repo used for nothing but hosting this workflow only ever gets a new commit when the workflow itself finds something new to report — so if every project you're monitoring goes quiet for a couple of months, that hosting repo could genuinely rack up 60 days of silence and get shut off without you ever being told.

To guard against that, every single run also quietly checks one extra thing: "has it been at least 59 days since the last commit to this repo?" If so — and only if you haven't already been sent this particular reminder — it posts a one-time Discord message titled **"Workflow Re-enable Reminder"**, pinging whoever you've configured via the top-level `ping_mode`/`ping_id` defaults in `config/repos.yml`. On a forum-channel template, this reminder is posted as its own brand-new Forum thread (separate from any thread used for an actual release or commit), since a forum channel has no shared "main" stream to drop a plain message into.

**Important: this message can't actually stop GitHub from disabling the workflow** — that decision is entirely GitHub's, made outside this workflow, and nothing a workflow run does from the inside can prevent it. What this message *does* do is warn you a day ahead of time, so once the workflow does get disabled, it isn't a surprise — you'll already know to go to your repo's **Actions** tab, click the workflow, and click **Re-enable workflow**, which starts everything running again immediately and takes about ten seconds.

A few more details worth knowing:

- You won't be spammed with this every 5 minutes once the 59-day mark passes — it's remembered (using GitHub's own Actions cache), so you'll only be told about a given quiet stretch once, not on every single run afterward.
- Any new commit to the repo — including this workflow's own bookmark-file update, the moment anything you're monitoring becomes active again — resets the clock automatically. You won't hear about it again unless another full 59 days of total silence pass.
- If the repo stays inactive even after you re-enable it, expect to be reminded again roughly every 59 days for as long as that remains true.
- This is fully automatic — there's nothing extra to set up or configure. It reuses the same top-level `ping_mode`/`ping_id` you already filled in for regular notifications.

## Troubleshooting

**The whole run fails immediately, before checking anything, and the log mentions something like `Invalid repository entry`, `Missing 'ping_id'`, `Invalid type`, or another config-related error.** Something in `config/repos.yml` isn't formatted correctly — a missing `/` in a repo name, a `ping:` value that isn't `user`/`role`/`everyone`/`none`, a `user`/`role` ping with no `ping_id` set, a repo block missing its required `type:`, or a setting misplaced outside the `repos:` section, for example. The error message names the exact line number to check. Unlike a single unreachable repo, a mistake in the config file blocks the *entire* run, since the workflow can't tell what to check yet. Fix the line and it'll pick back up on the next scheduled run (or click **Run workflow** to retry immediately).

**The log says `Config file not found`.** The workflow's `CONFIG_FILE` setting (Step 4) points somewhere that doesn't actually have a `repos.yml` there — usually because the file was never committed, or it's saved at a different path than `CONFIG_FILE` says. Double-check the file exists in the repo at that exact path.

**One repo in my config keeps failing, but the others post fine.** That's expected — see [Monitoring multiple repositories](#monitoring-multiple-repositories). The run still shows as failed overall so you notice, but the log will tell you exactly which repo via a line like `Error while monitoring OWNER/REPO: ...`, and every other repo in your config is still checked and posted about normally.

**Nothing posted, and there's no error in the Actions tab.** This is very likely completely normal — most runs find nothing new, since most 5-minute windows don't have a new release or commit in them. Open the run's log and look for a line like `No new release.` or `No new commit.` for the repo you're expecting — if you see that, everything is working correctly. If you expected something to have posted and didn't, double-check that repo's `repo:` value in `config/repos.yml` is spelled exactly right (`owner/repo-name`, and capitalization matters), that its `type:` matches this particular template, and that the release/commit genuinely exists on the repo's default (main) branch — the commit monitor only ever watches the default branch.

**The workflow doesn't seem to run on its own schedule at all.** GitHub automatically disables scheduled workflows in a repo that's had no commit activity for 60 days. If that's happened, go re-enable it manually from the Actions tab — these templates try to warn you about this a day in advance over Discord (see [Inactivity reminder](#inactivity-reminder) above), so you shouldn't be caught completely off guard, but the fix either way is the same: **Actions tab → the workflow → Re-enable workflow**. Scheduled runs can also sometimes be delayed by several minutes when GitHub itself is under heavy load — that's a limitation on GitHub's end, not something this workflow controls.

**Discord shows an error, or nothing shows up in the channel.** First, double check the `DISCORD_WEBHOOK` secret is set on the *same repo* the workflow file lives in (Settings → Secrets and variables → Actions). Also check that the webhook still exists and hasn't been deleted or regenerated on the Discord side — regenerating a webhook changes its URL, which would break it. The run's log will show you the exact error Discord sent back, which usually explains exactly what's wrong.

**A forum-channel template fails, especially with an error mentioning `thread_name`.** This almost always means the `DISCORD_WEBHOOK` secret is pointing at the wrong kind of channel. Forum-channel templates need a webhook that was created *on a Forum channel specifically* — using a regular text channel's webhook with a forum-channel template (or vice versa) will fail immediately on the very first post. Go back to Discord, create a new webhook on the correct channel type, and update the secret with the new URL.

**The workflow fails specifically at the "Save monitor state" step, and `git push` is the part that errors.** This usually means the repo has branch protection rules that block direct pushes to the main branch, even from GitHub's own automation. Either loosen that rule specifically for the `github-actions[bot]` account, or use a separate, unprotected repo just for this workflow's bookmark files.

**You want to see a notification you've already gotten again, to test something.** Open `STATE_DIR` in your repo and find the bookmark file for that specific repo — it's named automatically after the repo, e.g. `ChrisTitusTech__winutil_release.txt`. Either delete its contents, or change it to an older release tag / commit ID than the current one. The next run will then treat the current latest one as "new" again for that repo and post about it.

## Security & permissions notes

- This workflow only asks GitHub for permission to read its own repo and write (commit) back to it — nothing more. It cannot touch issues, pull requests, other repos, or your account settings.
- That permission only applies to the repo the *workflow file itself* lives in — not necessarily the repo(s) you list in `config/repos.yml`. This matters if you're trying to monitor a *different*, *private* repo than the one hosting the workflow — see the FAQ below for what to do in that case.
- Settings like `ping_id` and the list of repos you're watching live in `config/repos.yml`, stored as plain, visible text right alongside the workflow file — anyone who can view your repo can see them. Only `DISCORD_WEBHOOK` needs to be kept private, which is exactly why it's the one thing stored as a secret instead of typed directly into a file.

## FAQ

**Can I use this to monitor a private repo?** Only if it's the *same* repo you put the workflow file into — the automatic permission GitHub gives the workflow only covers its own repo. To watch a *different* private repo, you'd need to create a personal access token with read access to that repo, add it as an additional secret, and use it in place of the automatic one in the script (this applies to all repos the script requests with it, so it's the simplest option when the repos you're adding are yours). Public repos can always be monitored, regardless of which repo hosts the workflow, and can be freely mixed with the hosting repo itself in the same `repos.yml` list.

**Can I watch more than one repo, or set up more than one of these on the same repo?** Two different things, both possible:
- **To watch several repos with one workflow file:** just add more blocks to `repos:` in `config/repos.yml` — see [Monitoring multiple repositories](#monitoring-multiple-repositories). Each repo can optionally have its own ping settings (and, on forum-channel templates, its own Forum tags); any repo that doesn't specify these just uses the config's top-level defaults.
- **To run more than one separate monitor** (e.g. because you want entirely independent state, or you want to combine any of the four template types) — that's also fine, and they can even share the exact same `config/repos.yml` — see [Sharing one repos.yml across templates](#sharing-one-reposyml-across-templates). Just make sure each workflow *file* has its own unique `name` and `concurrency: group`, and that their `STATE_DIR` values don't collide if they'd otherwise process the exact same repo (different repos are safe either way, since state filenames are generated from the repo name). If two monitors accidentally share a `concurrency: group`, they'll interfere with each other.

**Do I need a separate `STATE_DIR` for each repo in my config?** No — one shared `STATE_DIR` works for any number of repos. A uniquely-named bookmark file is created automatically for each repo, so they never collide.

**Can I use a forum-channel template with a regular text channel, or the other way around?** No. A Discord webhook is tied to one specific channel of one specific type, and the two kinds of templates talk to Discord differently. Use whichever template matches the channel you actually have — see the table near the top of this README.

**Can I make it check more or less often than every 5 minutes?** Yes. Find the `cron:` line under `on: schedule:` near the top of the file and change it. 5 minutes is the fastest GitHub allows; you can make it check less often if you don't need near-instant notifications — e.g. `*/15 * * * *` for every 15 minutes, or `0 * * * *` for once an hour.

**How do I pause or completely turn this off?** To pause it temporarily: go to the **Actions** tab, click the workflow, click the **⋯** menu, and choose **Disable workflow**. You can turn it back on the same way, whenever you like. To remove it permanently, just delete the workflow file from `.github/workflows/`.

**Will this notify me about draft releases or prereleases?** No, not by default — it only notifies about a repo's actual "latest release," the same definition GitHub itself uses, which excludes drafts and prereleases. If you specifically want prereleases included too, that requires a small edit inside the script itself (removing one filter condition) — it's not something you can turn on just by changing a setting.

**Will I hit GitHub's rate limits by running this?** No, in virtually all normal cases. GitHub limits how many API requests you can make without logging in to 60 per hour, but this workflow automatically authenticates every request using the token GitHub provides for free, which raises that limit dramatically. Watching several repos in your config means one extra API call per repo per run, which barely moves the needle — you'd need to be watching an unusually large number of repos from a single repo before this would ever become a concern.

**Why did I get a Discord message titled "Workflow Re-enable Reminder"?** That's expected — not an error, and nothing is broken. It means the repo hosting this workflow hasn't had a commit in about 59 days, which is right before GitHub's own 60-day cutoff for automatically disabling scheduled workflows. See [Inactivity reminder](#inactivity-reminder) above for the full explanation; the short version is: check the **Actions** tab in a day or so, and if the workflow has been disabled, click **Re-enable workflow**.
