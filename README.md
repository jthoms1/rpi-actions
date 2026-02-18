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

Each pipeline stage is defined as a Claude Code command in the `commands/` directory. Developers can run stages locally for exploration before committing to a full pipeline run:

```
# Research stage — produces research/YYYY-MM-DD-<description>.md
/research_codebase Read issue #42 using gh and research what changes
are needed to add rate limiting.

# Plan stage — produces plans/YYYY-MM-DD-<description>.md
/create_plan research/2025-01-15-add-rate-limiting.md

# Implementation stage — executes the plan
/implement_plan plans/2025-01-15-add-rate-limiting.md
```

The commands are interactive — they ask clarifying questions and present options between stages, so you can review and course-correct before proceeding.

---

## Installation

### Prerequisites

- A GitHub repository
- An [Anthropic API key](https://console.anthropic.com/)
- The [Claude GitHub App](https://github.com/apps/claude) installed on your repo

### Step 1: Install claude-code-action

The quickest path is through the Claude Code CLI:

```bash
claude
> /install-github-app
```

This installs the Claude GitHub App and walks you through adding `ANTHROPIC_API_KEY` to your repository secrets.

Alternatively, set it up manually:

1. Install the [Claude GitHub App](https://github.com/apps/claude) on your repository.
2. Add your API key as a repository secret:
   ```bash
   gh secret set ANTHROPIC_API_KEY
   ```
   Or go to **Settings > Secrets and variables > Actions** and add it there.

### Step 2: Create the GitHub label

```bash
gh label create "agent:run" --description "Triggers the Research → Plan → Implement agent pipeline" --color 5319E7
```

Or create it manually: go to **Issues > Labels > New label**, name it `agent:run`, and pick a color.

### Step 3: Add workflow files

Create the following files in your repository:

**`.github/workflows/agent-pipeline.yml`** — Triggers the full pipeline when `agent:run` is applied to an issue.

```yaml
name: Agent Pipeline

on:
  issues:
    types: [labeled]

jobs:
  run-pipeline:
    if: github.event.label.name == 'agent:run'
    runs-on: ${{ vars.RUNS_ON || 'ubuntu-latest' }}
    permissions:
      contents: write
      issues: write
      pull-requests: write
      id-token: write
    steps:
      - uses: actions/checkout@v4

      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: |
            REPO: ${{ github.repository }}
            ISSUE: #${{ github.event.issue.number }}
            TASK: ${{ github.event.issue.title }}
            DESCRIPTION:
            ${{ github.event.issue.body }}

            Execute the three-stage agent pipeline defined in CLAUDE.md.
            Create a feature branch, run Research → Plan → Implement,
            and open a PR linking back to issue #${{ github.event.issue.number }}.
          claude_args: |
            --model claude-opus-4-6
            --max-turns 50

      - name: Remove trigger label
        if: always()
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.removeLabel({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              name: 'agent:run',
            });
```

**`.github/workflows/slash-commands.yml`** — Parses `/replan` and `/reresearch` on PR comments.

```yaml
name: Slash Commands

on:
  issue_comment:
    types: [created]

jobs:
  run:
    runs-on: ${{ vars.RUNS_ON || 'ubuntu-latest' }}
    steps:
      - uses: wow-actions/slash-commands@v1
        with:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          CONFIG_FILE: .github/slash-commands.yml
```

**`.github/slash-commands.yml`** — Config for the slash command action.

```yaml
pulls:
  replan:
    reactions: ["eyes"]
    dispatch: true
    comment: >
      Re-planning in progress. Claude will re-read the research, regenerate
      the plan with your feedback, and re-implement.

  reresearch:
    reactions: ["eyes"]
    dispatch: true
    comment: >
      Re-researching in progress. Claude will redo the research stage
      with your feedback, then regenerate the plan and implementation.
```

**`.github/workflows/agent-rerun.yml`** — Handles dispatched re-run events.

```yaml
name: Agent Re-Run

on:
  repository_dispatch:
    types: [replan, reresearch]

jobs:
  rerun:
    runs-on: ${{ vars.RUNS_ON || 'ubuntu-latest' }}
    permissions:
      contents: write
      issues: write
      pull-requests: write
      id-token: write
    steps:
      - name: Determine context
        id: context
        uses: actions/github-script@v7
        with:
          script: |
            const payload = context.payload.client_payload;
            const command = context.payload.action;
            const stage = command === 'reresearch' ? 'research' : 'plan';
            const feedback = payload.input || '';
            const prNumber = payload.github?.payload?.issue?.number;
            const commentAuthor = payload.github?.payload?.comment?.user?.login || '';

            let prAuthor = '';
            if (prNumber) {
              const { data: pr } = await github.rest.pulls.get({
                owner: context.repo.owner,
                repo: context.repo.repo,
                pull_number: prNumber,
              });
              prAuthor = pr.user.login;
            }

            const commitStrategy = (commentAuthor === prAuthor) ? 'force-push' : 'append';

            core.setOutput('stage', stage);
            core.setOutput('feedback', feedback);
            core.setOutput('pr_number', prNumber || '');
            core.setOutput('commit_strategy', commitStrategy);

      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Checkout PR branch
        if: steps.context.outputs.pr_number != ''
        run: gh pr checkout ${{ steps.context.outputs.pr_number }}
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: |
            REPO: ${{ github.repository }}
            PR: #${{ steps.context.outputs.pr_number }}
            RE-RUN FROM STAGE: ${{ steps.context.outputs.stage }}
            COMMIT STRATEGY: ${{ steps.context.outputs.commit_strategy }}
            REVIEWER FEEDBACK:
            ${{ steps.context.outputs.feedback }}

            Follow the re-run protocol in CLAUDE.md.
            Re-run from the "${{ steps.context.outputs.stage }}" stage,
            incorporating the reviewer feedback above.
            Use the "${{ steps.context.outputs.commit_strategy }}" commit strategy.
          claude_args: |
            --model claude-opus-4-6
            --max-turns 50
```

**`.github/workflows/claude.yml`** — Standard `@claude` interaction for ad-hoc questions on PRs and issues.

```yaml
name: Claude Assistant

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned, labeled]
  pull_request_review:
    types: [submitted]

jobs:
  claude-response:
    runs-on: ${{ vars.RUNS_ON || 'ubuntu-latest' }}
    permissions:
      contents: write
      issues: write
      pull-requests: write
      id-token: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          claude_args: |
            --model claude-opus-4-6
```

**`.github/workflows/monthly-agent-cleanup.yml`** — Sweeps `docs/agent-runs/` from main monthly.

```yaml
name: Monthly Agent Docs Cleanup

on:
  schedule:
    - cron: '0 0 1 * *'
  workflow_dispatch:

jobs:
  cleanup:
    runs-on: ${{ vars.RUNS_ON || 'ubuntu-latest' }}
    steps:
      - uses: actions/checkout@v4

      - name: Remove agent run artifacts
        run: |
          if [ -d "docs/agent-runs" ] && [ "$(ls -A docs/agent-runs)" ]; then
            git config user.name "github-actions[bot]"
            git config user.email "github-actions[bot]@users.noreply.github.com"
            git rm -r docs/agent-runs/*
            git commit -m "chore: monthly cleanup of agent run artifacts"
            git push
          fi
```

### Step 4: Add CLAUDE.md

Create a `CLAUDE.md` file in your repository root. This is the core of the pipeline — `claude-code-action` reads it automatically on every invocation. See [CLAUDE.md](#claudemd-reference) below for the full template.

### Summary of files

```
your-repo/
├── CLAUDE.md
├── commands/
│   ├── research_codebase.md    # /research_codebase command
│   ├── create_plan.md          # /create_plan command
│   └── implement_plan.md       # /implement_plan command
├── .github/
│   ├── slash-commands.yml
│   └── workflows/
│       ├── agent-pipeline.yml
│       ├── agent-rerun.yml
│       ├── claude.yml
│       ├── monthly-agent-cleanup.yml
│       └── slash-commands.yml
```

The `commands/` directory contains Claude Code custom commands that define the process for each pipeline stage. These are available as `/research_codebase`, `/create_plan`, and `/implement_plan` in any Claude Code session within the repo.

### Step 5 (Optional): Use a self-hosted runner

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
- **Secrets stay on the runner**: Your `ANTHROPIC_API_KEY` is available to jobs running on the machine. Treat the runner with the same security posture as any machine holding API keys.

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

Use the custom commands defined in `commands/` for exploratory work or to run stages one at a time:

**Research** — investigate the codebase and produce a structured research document:
```
/research_codebase Read issue #42 using gh and research what changes
are needed to add rate limiting.
```
Output: `research/2025-01-15-add-rate-limiting.md`

**Plan** — read the research and create a phased implementation plan:
```
/create_plan research/2025-01-15-add-rate-limiting.md
```
Output: `plans/2025-01-15-add-rate-limiting.md`

**Implement** — execute the plan with phased verification:
```
/implement_plan plans/2025-01-15-add-rate-limiting.md
```
Output: Code changes as described in the plan.

Each command is interactive and will pause for your input between steps. When running via the GitHub Actions pipeline, the same methodology is used but output goes to `docs/agent-runs/<feature-name>/` instead.

---

## CLAUDE.md Reference

Copy this into your repository root and customize as needed. The pipeline protocol references the commands in `commands/` as the authoritative process definitions for each stage.

```markdown
# Agent Pipeline Protocol

## Commands

The `commands/` directory contains three Claude Code commands that define the process
for each pipeline stage: `/research_codebase`, `/create_plan`, and `/implement_plan`.
These commands are the authoritative reference for how each stage should be executed.
When running the pipeline, follow their methodology while using the output paths
specified below.

## Three-Stage Pipeline

When executing the agent pipeline (triggered by an `agent:run` label on an issue),
follow these stages in order:

### Stage 1: Research
- Follow the research methodology defined in `commands/research_codebase.md`.
- Investigate the codebase relevant to the task using parallel sub-agents.
- Read documentation, identify constraints, and gather context.
- Document what exists — do not suggest improvements or critique the implementation.
- Write findings to `docs/agent-runs/<feature-name>/research.md`.
- Commit with message: `[research] Add research for <feature-name>`

### Stage 2: Plan
- Follow the planning methodology defined in `commands/create_plan.md`.
- Read your research output.
- Create a concrete implementation plan with phased changes, approach, and tradeoffs.
- Include automated and manual success criteria for each phase.
- Write to `docs/agent-runs/<feature-name>/plan.md`.
- Commit with message: `[plan] Add implementation plan for <feature-name>`

### Stage 3: Implement
- Follow the implementation methodology defined in `commands/implement_plan.md`.
- Read both research.md and plan.md.
- Execute the code changes described in the plan, phase by phase.
- Run automated verification after each phase.
- Commit with message: `[implement] <description of changes>`

### Metadata
After all stages, create or update `docs/agent-runs/<feature-name>/run-metadata.yml`
with timestamps and model info for each stage completed.

### PR Format
Open a PR with this description format:

    ## Agent Context
    - [Research](docs/agent-runs/<feature-name>/research.md)
    - [Plan](docs/agent-runs/<feature-name>/plan.md)

    ## Summary
    <Summary of what was implemented and why>

    Closes #<issue-number>

### Feature Name Convention
Derive `<feature-name>` from the issue title, converted to kebab-case.
Example: "Add rate limiting to API endpoints" → `add-rate-limiting-to-api-endpoints`.

---

## Re-Run Protocol

When triggered by a `/replan` or `/reresearch` slash command, the prompt will include
the stage to re-run from, the reviewer's feedback, and the commit strategy to use.

Steps:
1. Read the reviewer feedback from the prompt.
2. Re-run from the specified stage, carrying forward earlier artifacts where applicable.
   - `/replan`: Keep `research.md`, regenerate `plan.md`, then re-implement.
   - `/reresearch`: Redo all three stages from scratch.
3. Apply the specified commit strategy:
   - `force-push`: Rebase/amend the relevant stage commits (same author re-running).
   - `append`: Add new commits on top (different person re-running).
4. Update `run-metadata.yml` to reflect the re-run with the feedback reason.

---

## Research Document Structure

    # Research: <feature-name>

    ## Task
    <Restate the task from the issue>

    ## Codebase Analysis
    <Relevant files, patterns, and architecture notes>

    ## Constraints & Dependencies
    <What limits the solution space>

    ## Open Questions (Resolved)
    <Questions that came up during research and how they were resolved>

## Plan Document Structure

    # Plan: <feature-name>

    ## Approach
    <High-level description of the solution>

    ## Files to Change
    <List of files with what changes in each>

    ## Tradeoffs
    <What alternatives were considered and why this approach was chosen>

    ## Sequence
    <Order of implementation steps>
```
