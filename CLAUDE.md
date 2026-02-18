# rpi-actions

This repo is a distributable template that people copy into their own GitHub projects to get the Research-Plan-Implement (RPI) agent pipeline. It is not itself a project that uses the pipeline.

## Repo Structure

```
.claude/commands/
  research_codebase.md   # /research_codebase — defines the research methodology
  create_plan.md         # /create_plan — defines the planning methodology
  implement_plan.md      # /implement_plan — defines the implementation methodology

.github/workflows/
  agent-pipeline.yml     # Runs R→P→I as isolated jobs when "agent:run" label is applied
  agent-rerun.yml        # Handles /replan and /reresearch slash commands on PRs
  claude.yml             # Standard @claude interaction for ad-hoc questions
  monthly-agent-cleanup.yml  # Sweeps docs/agent-runs/ from main monthly

README.md                # Primary user-facing documentation; includes the CLAUDE.md
                         # template users copy into their own repos
```

## Key Conventions

- The **README** is the canonical source for installation instructions, usage docs, and the CLAUDE.md template (under "CLAUDE.md Reference"). If the pipeline protocol changes, update the README template and the corresponding workflow files together.
- **Workflow files** have detailed header comments explaining their trigger, behavior, required secrets, and optional variables. Maintain these comments when editing.
- The three **commands** define the methodology for each pipeline stage. The workflows reference these commands via CLAUDE.md instructions in the user's repo — they don't duplicate the command logic.
- There is no build system or test suite. This repo is Markdown and YAML. Validate changes by reviewing workflow syntax and checking that the README stays accurate.
