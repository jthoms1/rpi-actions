# Agent Pipeline

A three-stage AI agent pipeline — **Research**, **Plan**, **Implement** — that turns GitHub Issues into reviewed pull requests. Powered by [`claude-code-action`](https://github.com/anthropics/claude-code-action).

Label an issue with `agent:run`, and the agent autonomously researches your codebase, writes an implementation plan, builds the feature, and opens a PR. Reviewers see the research and plan alongside the code, and can send the agent back to any stage with `/replan` or `/reresearch` slash commands.

## Overview

### How it works

1. You create a GitHub Issue describing a task and apply the `agent:run` label.
2. The agent runs three stages, committing after each:

    ```
    commit 1: [research]  — Investigates the codebase, constraints, and context
    commit 2: [plan]      — Produces a concrete implementation plan
    commit 3+: [implement] — Executes the code changes
    ```

3. The agent opens a PR with the research and plan files linked at the top, so reviewers get the full reasoning alongside the diff.
4. If a reviewer spots an issue, they comment `/replan` or `/reresearch` with feedback and the agent re-runs from that stage.

### What reviewers see

The PR description links directly to the agent's artifacts:

```markdown
## Agent Context
- [Research](docs/agent-runs/add-rate-limiting/research.md)
- [Plan](docs/agent-runs/add-rate-limiting/plan.md)

## Summary
Added rate limiting middleware to all API endpoints using a token bucket
algorithm with Redis-backed state...

Closes #42
```

The research and plan files are also visible in the PR's "Files changed" tab, where reviewers can leave inline comments on specific lines of the agent's reasoning.

### Artifact lifecycle

Research and plan files live on the feature branch and land on `main` when the PR merges. A monthly cleanup action sweeps `docs/agent-runs/` to prevent accumulation. The PR itself is the permanent record — GitHub preserves the full diff forever, so the artifacts remain viewable through the PR even after cleanup.

### Re-runs with slash commands

Reviewers trigger re-runs by commenting on the PR with a slash command and their feedback:

```
/replan Use a streaming approach instead of batch processing.
The current plan doesn't account for backpressure.
```

```
/reresearch The research missed the existing rate limiter in src/middleware/.
Also check docs/ for the API contract constraints.
```

`/replan` keeps the research and regenerates the plan + implementation. `/reresearch` redoes all three stages. The agent gets an `eyes` reaction immediately to confirm the command was received.

**Commit strategy on re-runs:** if the person who triggered the re-run is the PR author, the agent force-pushes for a clean diff. If it's a different reviewer, the agent appends new commits to preserve the history.

### Local execution

Each pipeline stage is defined as a Claude Code skill in the `skills/` directory. Developers can run stages locally for exploration before committing to a full pipeline run:

```
# Research stage — produces research/YYYY-MM-DD-<description>.md
/research-codebase Read issue #42 using gh and research what changes
are needed to add rate limiting.

# Plan stage — produces plans/YYYY-MM-DD-<description>.md
/create-plan research/2025-01-15-add-rate-limiting.md

# Implementation stage — executes the plan
/implement-plan plans/2025-01-15-add-rate-limiting.md
```

The skills are interactive — they ask clarifying questions and present options between stages, so you can review and course-correct before proceeding.

---

## Installation

### Prerequisites

- A GitHub repository
- The [Claude GitHub App](https://github.com/apps/claude) installed on your repo

### Step 1: Install claude-code-action

The quickest path is through the Claude Code CLI:

```bash
claude
> /install-github-app
```

This installs the Claude GitHub App and walks you through adding `CLAUDE_CODE_OAUTH_TOKEN` to your repository secrets.

Alternatively, set it up manually:

1. Install the [Claude GitHub App](https://github.com/apps/claude) on your repository.
2. Add your OAuth token as a repository secret:
   ```bash
   gh secret set CLAUDE_CODE_OAUTH_TOKEN
   ```
   Or go to **Settings > Secrets and variables > Actions** and add it there.

> **Using an Anthropic API key instead?** The workflows use `claude_code_oauth_token` by default. If you prefer to use an [Anthropic API key](https://console.anthropic.com/) directly, replace `claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}` with `anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}` in each workflow file, and store your API key as the `ANTHROPIC_API_KEY` secret.

### Step 2: Create the GitHub label

```bash
gh label create "agent:run" --description "Triggers the Research → Plan → Implement agent pipeline" --color 5319E7
```

Or create it manually: go to **Issues > Labels > New label**, name it `agent:run`, and pick a color.

### Step 3: Add workflow files

Copy the following workflow files from this repo into your repository:

- [`.github/workflows/agent-pipeline.yml`](.github/workflows/agent-pipeline.yml) — Triggers the full pipeline when `agent:run` is applied to an issue.
- [`.github/workflows/agent-rerun.yml`](.github/workflows/agent-rerun.yml) — Handles dispatched re-run events (`/replan` and `/reresearch`).
- [`.github/workflows/claude.yml`](.github/workflows/claude.yml) — Standard `@claude` interaction for ad-hoc questions on PRs and issues.
- [`.github/workflows/monthly-agent-cleanup.yml`](.github/workflows/monthly-agent-cleanup.yml) — Sweeps `docs/agent-runs/` from main monthly.

### Step 4: Add skills

Copy the skill files from this repo into your repository:

- [`.claude/skills/research-codebase/SKILL.md`](.claude/skills/research-codebase/SKILL.md) — `/research-codebase` skill
- [`.claude/skills/create-plan/SKILL.md`](.claude/skills/create-plan/SKILL.md) — `/create-plan` skill
- [`.claude/skills/implement-plan/SKILL.md`](.claude/skills/implement-plan/SKILL.md) — `/implement-plan` skill

### Step 5: Add CLAUDE.md

Create a `CLAUDE.md` file in your repository root. This is the core of the pipeline — `claude-code-action` reads it automatically on every invocation. It should define the three-stage pipeline protocol, re-run protocol, and document structure templates for research and plan outputs.

### Summary of files

```
your-repo/
├── CLAUDE.md
├── .claude/
│   └── skills/
│       ├── research-codebase/SKILL.md   # /research-codebase skill
│       ├── create-plan/SKILL.md        # /create-plan skill
│       └── implement-plan/SKILL.md     # /implement-plan skill
└── .github/
    └── workflows/
        ├── agent-pipeline.yml
        ├── agent-rerun.yml
        ├── claude.yml
        └── monthly-agent-cleanup.yml
```

### Step 6 (Optional): Use a self-hosted runner

By default, all workflows use GitHub-hosted runners (`ubuntu-latest`). The runner VM does very little — Claude Code mostly makes API calls to Anthropic — so you can save on GitHub Actions minutes by running on your own hardware.

Set the repository variable `RUNS_ON` to `self-hosted` and the workflows will use your runner instead:

1. Go to **Settings > Secrets and variables > Actions > Variables**.
2. Click **New repository variable**.
3. Name: `RUNS_ON`, Value: `self-hosted`.

If the variable is not set, workflows fall back to `ubuntu-latest`.

See [Self-Hosted Runner Setup](#self-hosted-runner-setup) below for full instructions on setting up a Raspberry Pi or other machine as a runner.

---

## Self-Hosted Runner Setup

A self-hosted runner eliminates GitHub Actions minutes costs entirely. Since Claude Code's workload is I/O-bound (API calls, git operations, light file editing), even a Raspberry Pi handles it comfortably.

### Why self-host?

| | GitHub-hosted | Self-hosted |
|---|---|---|
| Runner cost | ~$0.008/min (~$0.24 per 30-min run) | $0 marginal (you own the hardware) |
| API cost | Anthropic API tokens | Same |
| Availability | Subject to queuing | Always ready |
| Setup | Zero | One-time setup |
| Maintenance | Zero | You keep it updated |

The Anthropic API cost is the dominant expense regardless of runner choice. Self-hosting removes the runner cost entirely.

### Hardware requirements

Claude Code on the runner needs very little:
- **CPU**: Any modern ARM or x86 processor (Raspberry Pi 4/5, old laptop, cheap VPS)
- **RAM**: 1-2 GB free (Node.js + git + gh CLI)
- **Disk**: A few GB for the runner agent, repo checkouts, and Node.js
- **Network**: Stable internet connection (the runner polls GitHub for jobs)

### Raspberry Pi setup

#### 1. Install the OS

Use Raspberry Pi Imager to flash **Raspberry Pi OS Lite (64-bit)** to an SD card. Enable SSH during setup.

#### 2. Install dependencies

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Node.js (claude-code-action needs it)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo bash -
sudo apt install -y nodejs

# Install git and gh CLI
sudo apt install -y git
sudo mkdir -p -m 755 /etc/apt/keyrings
wget -qO- https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null
sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update && sudo apt install -y gh

# Verify
node --version && git --version && gh --version
```

#### 3. Create a runner user

```bash
sudo useradd -m -s /bin/bash runner
sudo usermod -aG sudo runner
sudo su - runner
```

#### 4. Install the GitHub Actions runner

Go to your repository on GitHub: **Settings > Actions > Runners > New self-hosted runner**.

GitHub will show commands specific to your repo. They look like this:

```bash
# Download (GitHub shows the current URL — use that, not this example)
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-arm64-2.321.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.321.0/actions-runner-linux-arm64-2.321.0.tar.gz
tar xzf ./actions-runner-linux-arm64-2.321.0.tar.gz

# Configure (GitHub provides your specific token)
./config.sh --url https://github.com/YOUR-ORG/YOUR-REPO --token YOUR_TOKEN

# Install and start as a service
sudo ./svc.sh install
sudo ./svc.sh start
```

> Use `linux-arm64` for Raspberry Pi, `linux-x64` for x86 machines.

#### 5. Verify the runner is online

Go to **Settings > Actions > Runners** — your runner should show as "Idle".

#### 6. Set the repository variable

```bash
gh variable set RUNS_ON --body "self-hosted"
```

Or go to **Settings > Secrets and variables > Actions > Variables** and add `RUNS_ON` with value `self-hosted`.

All workflows will now use your Raspberry Pi.

### Other machines (VPS, old laptop, etc.)

The steps are the same — install Node.js, git, and gh, then follow GitHub's self-hosted runner setup. A $4-6/month VPS (Hetzner CAX11, Oracle Cloud free tier) works well if you don't have hardware on hand.

### Security considerations

- **Private repos only**: GitHub [recommends](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners#self-hosted-runner-security) self-hosted runners for private repositories. On public repos, anyone can fork and trigger workflows on your runner.
- **Keep the runner updated**: Periodically update the runner agent, OS, Node.js, and gh CLI.
- **Limit network exposure**: The runner only needs outbound HTTPS access to GitHub and Anthropic APIs. No inbound ports required.
- **Secrets stay on the runner**: Your `CLAUDE_CODE_OAUTH_TOKEN` is available to jobs running on the machine. Treat the runner with the same security posture as any machine holding API keys.

### Switching back to GitHub-hosted

```bash
gh variable delete RUNS_ON
```

Or set it back to `ubuntu-latest` with `gh variable set RUNS_ON --body "ubuntu-latest"`. The workflows fall back to GitHub-hosted runners immediately.

---

## Usage

### Running the full pipeline

1. Create a GitHub Issue with a clear description of the task.
2. Apply the `agent:run` label.
3. The agent creates a feature branch, runs all three stages, and opens a PR.
4. Review the PR — the research and plan are in the "Files changed" tab.
5. Merge when satisfied.

### Requesting a re-plan

If the plan needs changes but the research is sound, comment on the PR:

```
/replan <your feedback>
```

The agent keeps `research.md`, regenerates `plan.md` incorporating your feedback, and re-implements.

### Requesting re-research

If the research missed something fundamental, comment on the PR:

```
/reresearch <your feedback>
```

The agent redoes all three stages from scratch, incorporating your feedback.

### Asking ad-hoc questions

You can also ask the agent questions about the PR without triggering a re-run:

```
@claude Why did you choose a token bucket algorithm over sliding window?
```

This uses the standard `claude.yml` workflow and doesn't modify any files.

### Running stages locally

Use the skills defined in `skills/` for exploratory work or to run stages one at a time:

**Research** — investigate the codebase and produce a structured research document:
```
/research-codebase Read issue #42 using gh and research what changes
are needed to add rate limiting.
```
Output: `research/2025-01-15-add-rate-limiting.md`

**Plan** — read the research and create a phased implementation plan:
```
/create-plan research/2025-01-15-add-rate-limiting.md
```
Output: `plans/2025-01-15-add-rate-limiting.md`

**Implement** — execute the plan with phased verification:
```
/implement-plan plans/2025-01-15-add-rate-limiting.md
```
Output: Code changes as described in the plan.

Each command is interactive and will pause for your input between steps. When running via the GitHub Actions pipeline, the same methodology is used but output goes to `docs/agent-runs/<feature-name>/` instead.
