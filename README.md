# GitHub → Discord Repo Monitors

Two GitHub Actions workflow templates that monitor a GitHub repository and automatically post Discord notifications when new commits or releases are detected.

The templates are designed to be reusable: copy the workflow you want into a repository you control, configure the monitor settings, add a Discord webhook as a repository secret, and GitHub Actions will handle the monitoring and notification process.

> **Important:** This README documents how the supplied templates work. The templates themselves are not modified by this README update.

---

## Templates

This repository provides two independent GitHub Actions workflow templates:

| Template | Monitors | Notification contents |
|---|---|---|
| `repo-commit-monitor-template.yml` | New commits | Project name, author, short commit SHA, commit link, and commit message |
| `repo-release-monitor-template.yml` | New published releases | Project name, version/tag, author, release link, and release notes |

You can use either template independently, or use **both on the same repository**.

The commit monitor and release monitor are separate workflows and use separate state files by default, allowing them to track commits and releases independently.

---

# Quick Start

## 1. Choose a template

Copy one or both template files into the `.github/workflows/` directory of a repository you control.

The repository containing the workflow **does not have to be the repository being monitored**. It only needs to be a repository where GitHub Actions can run and where the workflow can save its monitoring state.

Example:

```text
your-repository/
└── .github/
    └── workflows/
        ├── my-project-commit-monitor.yml
        └── my-project-release-monitor.yml
```

The workflow filename itself does not matter as long as it:

- Uses the `.yml` extension
- Is located inside `.github/workflows/`

### Creating the workflow on GitHub.com

Open the repository where you want the monitor to run and choose:

**Add file → Create new file**

Then enter a path such as:

```text
.github/workflows/release-monitor.yml
```

GitHub will create the required `.github/` and `workflows/` directories automatically when the path is entered.

Paste the selected template into the new file and commit it.

### Creating the workflow locally

Inside a local clone of the repository, create the workflow directory if necessary:

```bash
mkdir -p .github/workflows
```

Save the template there, then commit and push it normally:

```bash
git add .github/workflows/<filename>.yml
git commit -m "Add repo monitor"
git push
```

---

# Workflow Configuration

Each template contains an `env:` block under the `monitor` job. These values control which repository is monitored, how it appears in Discord, where the monitor stores its state, and who should be pinged.

| Variable | Description |
|---|---|
| `REPO` | The GitHub repository to monitor in `OWNER/REPO` format, for example `ChrisTitusTech/winutil`. |
| `PROJECT_NAME` | The name displayed in the Discord notification title. |
| `STATE_FILE` | The file used to remember the last commit or release that was processed. Use a different filename for each monitor when multiple monitors are installed in the same workflow repository. |
| `PING_MODE` | Controls whether a user, role, `@everyone`, or nobody is pinged. |
| `PING_ID` | The Discord user ID or role ID used when `PING_MODE` is `user` or `role`. It is not required for `everyone` or `none`. |
| `NOTIFY_ON_INITIAL_RUN` | Controls whether the first run sends a notification for the currently newest item or only records it as the starting state. |

### Example configuration

```yaml
env:
  REPO: "OWNER/REPO"
  PROJECT_NAME: "Project Name"
  STATE_FILE: "last_commit.txt"
  PING_MODE: "user"
  PING_ID: "YOUR_DISCORD_USER_ID_HERE"
  NOTIFY_ON_INITIAL_RUN: "true"
```

For the release monitor, the default state filename is `last_release.txt`.

---

# Naming the Workflow

At the top of the workflow, change the `name:` value to something unique and recognizable.

Example:

```yaml
name: MyProject Commit Monitor
```

This is the display name shown in the GitHub Actions interface.

## Concurrency Group

Each monitor also has a concurrency group, for example:

```yaml
concurrency:
  group: repo-commit-monitor
```

Change this to a unique value when adding multiple monitor workflows to the same workflow repository.

For example:

```yaml
concurrency:
  group: myproject-commit-monitor
```

and:

```yaml
concurrency:
  group: myproject-release-monitor
```

The purpose of this setting is to prevent overlapping runs of the same monitor. The templates use `cancel-in-progress: true`, so an earlier in-progress run can be cancelled when another run for the same concurrency group starts.

---

# Discord Webhook Setup

The monitors send notifications through a Discord webhook.

In the repository where the workflow is installed, go to:

**Settings → Secrets and variables → Actions → New repository secret**

Create a repository secret named exactly:

```text
DISCORD_WEBHOOK
```

Paste the Discord webhook URL into the secret value.

The webhook URL is obtained from the Discord channel where notifications should be posted:

**Discord channel → Edit Channel → Integrations → Webhooks**

### Why the webhook is a secret

The webhook URL is stored in GitHub Actions as `DISCORD_WEBHOOK` rather than directly in the workflow file. This keeps the actual webhook URL out of the repository's source code.

Do **not** replace the secret with the real webhook URL directly inside the YAML file.

---

# GitHub Token

The templates also use:

```text
GITHUB_TOKEN
```

GitHub automatically provides this token to the workflow, so there is no manual secret setup required.

The templates use it when available for GitHub API requests. In particular, it helps provide a higher API rate limit than unauthenticated GitHub API access.

The workflow itself also requests:

```yaml
permissions:
  contents: write
```

This is required because the monitor stores its last-seen state in a file and commits that state back to the repository where the workflow is installed.

---

# Choosing Who Gets Pinged

Set `PING_MODE` to one of four values:

| `PING_MODE` | Behavior | `PING_ID` |
|---|---|---|
| `user` | Pings one specific Discord user | Required: user ID |
| `role` | Pings a Discord role | Required: role ID |
| `everyone` | Pings `@everyone` | Not required |
| `none` | Sends no ping | Not required |

The default template setting is `user`.

## Getting a Discord User or Role ID

In Discord, enable Developer Mode:

**User Settings → Advanced → Developer Mode**

Then right-click the person or role and choose **Copy User ID** or **Copy Role ID**.

Paste the resulting ID into `PING_ID`.

### `@everyone` behavior

`PING_MODE=everyone` explicitly enables an `@everyone` mention. This notifies members with access to the channel and is intended for channels where update notifications are expected.

The templates use Discord's `allowed_mentions` configuration so that the selected ping type is explicitly controlled instead of simply relying on mention text inside the message.

### Multi-message notifications

When a commit message or release notes are too long for one Discord message, the notification can be split into multiple messages. The configured ping is only included on the **first message** of that notification, preventing a long release or changelog from repeatedly pinging the same person, role, or everyone.

---

# Initial Run Behavior

The first time a monitor runs, it has no previous state to compare against.

This is controlled by:

```yaml
NOTIFY_ON_INITIAL_RUN: "true"
```

### When set to `true`

The first run treats the newest commit or release as a new item and sends a notification for it.

### When set to `false`

The first run silently records the current newest commit or release in the state file and does **not** send a notification.

This is useful when you want to start monitoring a repository without immediately receiving a notification for something that already existed before the monitor was installed.

---

# Monitoring Schedule

Both templates contain:

```yaml
on:
  schedule:
    - cron: "*/5 * * * *"
  workflow_dispatch:
```

The scheduled run checks for changes every **5 minutes**.

The `workflow_dispatch` trigger also allows you to start the workflow manually from the GitHub Actions interface.

## Manual test

To test the monitor without waiting for the next scheduled run:

1. Open the repository's **Actions** tab.
2. Select the monitor workflow.
3. Choose **Run workflow**.
4. Start the workflow manually.

---

# Commit Monitor Behavior

`repo-commit-monitor-template.yml` watches the commit history returned by the GitHub API.

The monitor requests recent commits and processes them from newest to older commits. It remembers the last processed commit using the configured state file, which defaults to:

```text
last_commit.txt
```

## Commit notification contents

The first message for a commit contains:

```text
PROJECT_NAME Updated!

Author: AUTHOR
Commit: SHORT_SHA
Link: COMMIT_URL

Change Log:
COMMIT_MESSAGE
```

The actual Discord notification uses Discord Components v2 and separators to visually divide the notification header, metadata, and changelog content.

The commit notification includes:

- The project name
- The commit author
- The first 7 characters of the commit SHA
- A direct link to the commit on GitHub
- The commit message

---

# Release Monitor Behavior

`repo-release-monitor-template.yml` watches the repository's published GitHub Releases.

The monitor requests recent releases and remembers the last processed release using the configured state file, which defaults to:

```text
last_release.txt
```

## Release filtering

The release monitor ignores:

- Draft releases
- Prereleases

It therefore monitors published, non-prerelease releases.

## Release notification contents

The first message for a release contains:

```text
PROJECT_NAME Updated!

Version: TAG
Author: AUTHOR
Release: RELEASE_URL

Release Notes:
RELEASE_NOTES
```

The release notification includes:

- The project name
- The release tag/version
- The release author
- A direct link to the GitHub release
- The release notes

---

# Catch-Up Behavior

The monitors do not only check whether the newest item changed. They attempt to detect **all items that appeared since the last saved state**.

For example, if a monitor last saw commit `A` and the repository later received commits `B`, `C`, and `D`, the monitor can identify all three new commits rather than only posting `D`.

The same behavior applies to releases.

## Processing order

When multiple new items are detected, the monitor reverses the API's newest-first result so notifications are sent in chronological order from the older unseen item toward the newest one.

This means a missed batch is reported in the order the items appeared rather than newest-first.

---

# Per-Run Notification Limits

To prevent a large push or release batch from flooding the Discord channel, each monitor has a maximum number of individual items it will report during one run.

The templates currently use:

```text
10 items per run
```

That means:

- Up to 10 new commits can be posted by the commit monitor in one run.
- Up to 10 new releases can be posted by the release monitor in one run.

If more than 10 items are detected, the older items beyond the limit are not posted individually. The notification includes a note indicating how many earlier items were omitted from individual messages.

The state is still advanced through the items that are processed, so the monitor does not intentionally repost already handled notifications on the next successful run.

---

# Missing-State / History Gap Handling

A monitor can encounter a situation where the saved last-seen commit or release is no longer present in the recent API results.

For example, this can happen when the repository's history contains more changes than the monitor's recent API query returns, or when the previously recorded item is no longer available in the returned history.

When that happens, the monitor cannot reconstruct the complete sequence of missed items.

Instead, it:

1. Detects that the previous state cannot be found.
2. Marks the situation as a history gap.
3. Reports the newest available commit or release only.
4. Adds a note explaining that the previous state could not be found in recent history.

This prevents the monitor from inventing a sequence of notifications that it cannot reliably reconstruct.

---

# State Files

The monitor needs to remember what it has already processed between workflow runs.

The state is stored in a plain text file containing the latest processed identifier:

### Commit monitor

```text
last_commit.txt
```

The file contains the processed commit SHA.

### Release monitor

```text
last_release.txt
```

The file contains the processed release tag.

## Multiple monitors in one repository

If you install more than one monitor that would otherwise use the same state filename, change `STATE_FILE` so each monitor has its own state.

For example:

```yaml
STATE_FILE: "project-a-commits.txt"
```

and:

```yaml
STATE_FILE: "project-a-releases.txt"
```

This prevents one monitor from overwriting another monitor's tracking information.

## State persistence

After notifications are processed, the workflow stages the state file, commits it using the GitHub Actions bot identity, pulls with rebase, and pushes the updated state back to the workflow repository.

The templates also save progress **after each individual commit or release** rather than waiting until the entire batch has finished. This reduces the chance of already-posted items being reposted after a partial failure, such as a Discord outage.

---

# Discord Message Formatting

The templates use Discord **Components v2** rather than a traditional embed-only notification.

The first message contains a structure similar to:

```text
[Optional Ping]

## Project Updated!
────────────────
Author: ...
Commit: ...
Link: ...
────────────────
## Change Log:
...
```

For releases, the structure is equivalent but uses version/release information and:

```text
## Release Notes:
```

The templates also insert separator components between Markdown heading sections found in the commit message or release notes. This is intended to preserve the section-oriented structure of GitHub-generated changelogs when they are displayed in Discord.

---

# Long Commit Messages and Release Notes

Discord has message-size and component-count limits. The templates account for those limits instead of attempting to send an arbitrarily large changelog as one message.

The templates use an internal text budget of approximately **3,800 characters** per message and a component limit of **36 total components**, intentionally staying below the higher platform limits.

## How continuation messages work

When the content is too long for the first notification, the monitor creates one or more continuation messages.

Continuation messages are labeled in the format:

```text
## Change Log (continued) — Part 2/3:
```

or:

```text
## Release Notes (continued) — Part 2/3:
```

The total number of parts is included in the label.

### Content boundaries

The monitor tries to keep each logical line together. A bullet point, Markdown heading, or link that does not fit in the remaining space is moved to the next message rather than being split across messages.

For an unusually long individual line that cannot fit on its own, the fallback logic breaks it on whitespace where possible instead of arbitrarily cutting a word or link in half.

This is especially useful for release notes containing long generated changelogs.

---

# Issue and Pull Request Link Handling

GitHub commonly displays references such as:

```text
#123
```

as clickable issue or pull-request references on GitHub itself. Discord does not automatically convert a bare `#123` into the corresponding GitHub issue link.

To preserve that behavior in Discord, the templates normalize both of the following forms:

```text
#123
```

and a full bare repository URL such as:

```text
https://github.com/OWNER/REPO/issues/123
```

into explicit Markdown links before sending the notification.

Existing Markdown links are skipped so the monitor does not unnecessarily wrap an already-formatted link.

This also allows a single release to contain a mixture of hand-written references and automatically generated GitHub references without requiring the release author to format them specifically for Discord.

---

# Network Retry and Rate-Limit Handling

The templates include retry handling for temporary network failures and Discord/GitHub API rate limiting.

## Network errors

For transient network conditions such as timeouts or DNS-related connection errors, the templates retry the request up to **3 attempts**.

The retry delay increases between attempts using a short backoff.

## HTTP 429 rate limits

GitHub or Discord can return HTTP `429` when requests are being rate limited.

The templates handle this separately:

- Up to **5 rate-limit retry attempts** are allowed.
- The workflow checks the `Retry-After` response when available.
- Individual waits are capped at **15 seconds**.

Other HTTP errors are not treated as temporary by this retry layer and are allowed to fail normally so the actual HTTP error can be reported.

## Discord send throttling

The monitors also enforce a minimum **0.5 second delay** between consecutive Discord webhook sends.

This is particularly important when one commit message or release generates multiple continuation messages, or when a single run discovers several new commits/releases.

---

# GitHub API Details

The templates communicate directly with the GitHub API using Python's standard library rather than requiring an additional third-party Python package.

### Commit monitor endpoint

The commit monitor requests up to 100 recent commits:

```text
https://api.github.com/repos/OWNER/REPO/commits?per_page=100
```

### Release monitor endpoint

The release monitor requests up to 30 recent releases:

```text
https://api.github.com/repos/OWNER/REPO/releases?per_page=30
```

The returned results are treated as newest-first, and the monitor compares those results against the stored state file.

The workflows identify themselves to the GitHub API using a custom `User-Agent` and specify the GitHub API version used by the templates.

---

# GitHub Actions Workflow Structure

Both templates follow the same general workflow structure:

```text
Workflow trigger
      ↓
Checkout monitor state
      ↓
Query GitHub API
      ↓
Compare current state with saved state
      ↓
Determine new commits/releases
      ↓
Format Discord notification
      ↓
Send webhook notification(s)
      ↓
Save state after each processed item
      ↓
Commit and push updated state file
```

The workflow job runs on:

```text
ubuntu-latest
```

and has a 10-minute job timeout.

---

# Common Setup Mistakes

## Forgetting the Discord webhook secret

The workflow expects a repository secret named exactly:

```text
DISCORD_WEBHOOK
```

A different secret name will not match the template's configuration.

## Using the wrong repository format

`REPO` must be written as:

```text
OWNER/REPO
```

For example:

```yaml
REPO: "ChrisTitusTech/winutil"
```

Do not use the full browser URL in this variable.

## Forgetting to change the concurrency group

If you install multiple monitor workflows and reuse the same concurrency group, the workflows can interfere with one another.

Give each monitor its own group value.

## Reusing the same state filename

Multiple monitors should not share the same `STATE_FILE` unless they are intentionally tracking the same type of item. Give each independent monitor its own state file.

## Incorrect `PING_ID`

`PING_ID` must be the numeric Discord ID, not a username, display name, or role name.

`PING_ID` is also unnecessary when using:

```text
PING_MODE=everyone
```

or:

```text
PING_MODE=none
```

---

# Repository Placement

The repository where the workflow runs and the repository being monitored are separate concepts.

For example, you could have:

```text
Monitor repository
└── .github/workflows/my-monitor.yml

Monitored repository
└── OWNER/PROJECT
```

The workflow uses the `REPO` variable to decide which repository's commits or releases to query, while the workflow repository stores the monitor's state file and runs the GitHub Actions job.

This makes it possible to keep monitoring logic in a dedicated repository rather than modifying the project that is being watched.

---

# Using Both Monitors Together

You can run both templates against the same monitored repository.

For example:

```yaml
# Commit monitor
env:
  REPO: "OWNER/REPO"
  PROJECT_NAME: "Project Name"
  STATE_FILE: "last_commit.txt"
```

and:

```yaml
# Release monitor
env:
  REPO: "OWNER/REPO"
  PROJECT_NAME: "Project Name"
  STATE_FILE: "last_release.txt"
```

Using separate concurrency groups and state filenames keeps the two monitors independent.

The result is a Discord channel that can receive notifications for both:

```text
New commit → Commit notification

New release → Release notification
```

---

# Example Notification Flow

Suppose three commits were pushed while the workflow was unable to run:

```text
Commit A  ← last saved state
Commit B
Commit C
Commit D  ← newest
```

On the next successful check, the monitor can identify `B`, `C`, and `D` as new commits and send notifications for them in chronological order.

If there are more than 10 unseen commits/releases in one run, only the configured maximum is posted individually, and the first notification includes a note explaining that earlier items were not shown individually.

If the saved state cannot be found in the recent API response, the monitor reports the newest available item and adds a history-gap warning instead of assuming that all missing history can be reconstructed.

---

# Operational Limits and Design Choices

The templates intentionally use conservative internal limits to keep notifications reliable:

| Setting | Current value | Purpose |
|---|---:|---|
| Scheduled interval | 5 minutes | Regular monitoring interval |
| Job timeout | 10 minutes | Prevent a stuck workflow from running indefinitely |
| Commit API page size | 100 | Recent commit history available to compare |
| Release API page size | 30 | Recent releases available to compare |
| Max commits per run | 10 | Prevent notification flooding |
| Max releases per run | 10 | Prevent notification flooding |
| Discord text budget | ~3,800 characters | Stay below Discord message limits |
| Component budget | 36 components | Stay below Discord component limits |
| Network retry attempts | 3 | Handle temporary connection failures |
| Rate-limit retry attempts | 5 | Handle HTTP 429 responses |
| Rate-limit wait cap | 15 seconds | Prevent excessively long individual waits |
| Discord send spacing | 0.5 seconds | Reduce webhook rate-limit pressure |

These values are implementation details of the supplied templates rather than user-facing configuration variables.

---

# Notes and Limitations

## Drafts and prereleases

The release monitor excludes draft releases and prereleases. This behavior is intentionally built into the release lookup logic rather than exposed as an environment variable.

The template's current behavior therefore focuses on published, non-prerelease releases.

## History availability

The monitors can only compare against the recent history returned by their API queries. If a previously saved commit or release is outside that available history, the monitor cannot reliably reconstruct every missed item and falls back to reporting the newest available one with a warning.

## Message continuation

Very large commit messages or release notes can be split across several Discord messages. The continuation system is designed to preserve line boundaries and section formatting, but a single extremely long unbroken line may still require fallback word-wrapping.

## Scheduled workflow timing

The schedule is configured for every 5 minutes, but scheduled GitHub Actions jobs should be treated as periodic checks rather than a guaranteed exact-to-the-second notification system.

---

# Files in This Repository

```text
README.md
repo-commit-monitor-template.yml
repo-release-monitor-template.yml
```

The two YAML files are reusable workflow templates. The README explains how to configure and deploy them without requiring changes to the template's monitoring logic.

---

# Summary

The repository provides two GitHub Actions templates for forwarding GitHub repository activity to Discord:

- **Commit Monitor** — watches for new commits and posts the author, short SHA, commit URL, and commit message.
- **Release Monitor** — watches for new published releases and posts the version, author, release URL, and release notes.

Both templates support 5-minute polling, manual workflow runs, configurable Discord pings, persistent state tracking, catch-up handling, batching limits, Discord message continuation, GitHub issue/PR link conversion, network retries, rate-limit handling, and progress saving after individual notifications.

The main setup requirements are simply to place the workflow under `.github/workflows/`, configure the environment variables, add the `DISCORD_WEBHOOK` repository secret, and give each independent monitor its own concurrency group and state file.
