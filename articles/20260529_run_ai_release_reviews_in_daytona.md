---
title: 'Run AI Release Reviews in Daytona'
description:
  'Use Omni Engineer and Claude Engineer in Daytona workspaces to inspect
  release notes, migration risks, and verification steps before publishing.'
date: 2026-05-29
author: 'Jonah Sills'
tags: ['daytona', 'ai engineering', 'release review']
---

# Run AI Release Reviews in Daytona

Release work is where small assumptions become expensive. A dependency bump can
look harmless until a customer hits a renamed option. A changelog can sound
complete until the migration path skips the one command that every user needs.
A test suite can pass while the release notes still fail to explain what changed
and who needs to care.

That is why a [release review workspace](/definitions/20260529_definition_release_review_workspace.md)
is a useful habit. Instead of reviewing a release from a personal laptop with
unknown tools, old environment variables, and half-remembered setup steps, you
can open a clean Daytona workspace and run the same review from a reproducible
starting point. In this guide, we use two AI engineering tools in parallel:
[Omni Engineer](https://github.com/Doriandarko/omni-engineer) for mapping the
release surface and [Claude Engineer](https://github.com/Doriandarko/claude-engineer)
for challenging risk, migration, and test coverage.

![Daytona AI release review workflow](/assets/20260529_run_ai_release_reviews_in_daytona.svg)

## TL;DR

- Use Daytona to keep AI-assisted release checks isolated from your main
  machine.
- Run Omni Engineer first to summarize the release diff and draft the review
  checklist.
- Run Claude Engineer next to challenge the checklist and look for migration
  risks.
- Keep API keys in local environment variables, never in the repository.
- Publish the final release notes only after the AI findings are verified with
  real commands.

## Prepare the two AI engineer repositories

The companion Dev Container pull requests for this guide are:

- Omni Engineer:
  [Doriandarko/omni-engineer#39](https://github.com/Doriandarko/omni-engineer/pull/39)
- Claude Engineer:
  [Doriandarko/claude-engineer#263](https://github.com/Doriandarko/claude-engineer/pull/263)

Both Dev Containers follow the same principle: make setup repeatable without
committing credentials. Omni Engineer receives `OPENROUTER_API_KEY` from the
local environment. Claude Engineer receives `ANTHROPIC_API_KEY`, and optionally
`E2B_API_KEY`, the same way. In Daytona, that means the workspace can install
the repository dependencies while secrets stay in the user's account or local
shell.

The minimal pattern looks like this:

```json
{
  "image": "mcr.microsoft.com/devcontainers/python:3.11-bookworm",
  "containerEnv": {
    "OPENROUTER_API_KEY": "${localEnv:OPENROUTER_API_KEY}"
  },
  "postCreateCommand": "python -m pip install --upgrade pip && python -m pip install -r requirements.txt"
}
```

Claude Engineer uses the same Python base image and installs its own
`requirements.txt`. The important part is that the Dev Container describes the
workspace, not the release itself. It should make the review possible from a
fresh checkout and leave project-specific release inputs outside the AI tool
repositories.

## Start the Daytona workspaces

Create one Daytona workspace for Omni Engineer and one for Claude Engineer.
Using separate workspaces keeps each assistant's context small and makes their
outputs easier to compare.

For Omni Engineer:

```bash
git clone https://github.com/Doriandarko/omni-engineer.git
cd omni-engineer
export OPENROUTER_API_KEY="your-local-key"
python main.py
```

For Claude Engineer:

```bash
git clone https://github.com/Doriandarko/claude-engineer.git
cd claude-engineer
export ANTHROPIC_API_KEY="your-local-key"
python ce3.py
```

If you use the web interface for Claude Engineer instead of the CLI, run
`python app.py` and open the local URL from the Daytona workspace. For either
tool, avoid pasting private customer data, unpublished security details, or
secret environment values into prompts. A release review should inspect code,
tests, docs, and migration behavior, not collect sensitive material.

## Give Omni Engineer the release surface

Start with the assistant that maps the release. The goal is not to let the tool
decide whether the release ships. The goal is to generate a structured first
pass that a maintainer can verify.

In the release repository, prepare a small bundle of inputs:

```bash
git log --oneline v1.8.0..HEAD > /tmp/release-commits.txt
git diff --stat v1.8.0..HEAD > /tmp/release-diffstat.txt
git diff --name-only v1.8.0..HEAD > /tmp/release-files.txt
```

Then ask Omni Engineer to classify the release surface:

```text
Review these release inputs. Group the changes into user-facing features,
bug fixes, dependency changes, documentation updates, and migration risks.
Return a checklist with commands I should run to verify each risky item.
```

A useful Omni Engineer output should include changed files, a short description
of each user-facing change, and a list of uncertain areas. For example, a
renamed configuration key should trigger a docs check. A database migration
should trigger an upgrade and rollback check. A dependency bump should trigger
the tests that exercise the integration, not only the package manager command.

## Ask Claude Engineer to challenge the checklist

Claude Engineer is useful as the second reviewer because it can be framed as a
skeptic. Give it the Omni checklist, the release diff summary, and the draft
release notes. Ask it to look for missing verification rather than to rewrite
everything.

```text
Act as a release risk reviewer. Here is the draft release checklist and release
note outline. Identify anything that is unsupported by tests, unclear for users,
or likely to break migration from the previous version. Return only actionable
review items with suggested verification commands.
```

This second pass should produce a smaller, sharper list. Good findings look
like this:

- A CLI flag changed but the release notes do not mention the old flag.
- A config default changed but the docs do not describe the default behavior.
- The integration test covers a happy path but not an upgrade path.
- A dependency bump changes supported runtime versions.

Treat those findings as prompts for human verification. If a suggested risk
does not apply, record why. If it does apply, run the command, update the docs,
or adjust the release notes before publishing.

## Turn both outputs into a final checklist

The final release review should be short enough for a maintainer to use during
the release window. A practical format is a Markdown table:

| Area | Risk | Verification | Status |
| --- | --- | --- | --- |
| CLI | Renamed flag may surprise users | `tool --help` and migration note | Done |
| Docs | New config option lacks example | Run docs preview and link section | Done |
| Tests | Upgrade path not exercised | Run upgrade fixture from previous tag | Pending |

This table belongs in the release pull request, release issue, or internal
release checklist. The AI tools help create it, but the final status should only
change after a developer has run the command or inspected the code.

## Keep the workflow reproducible

Daytona is useful here because the environment can be recreated. If a reviewer
questions the release note or asks how a migration risk was checked, you can
open the same workspace, run the same commands, and compare the output. That is
much cleaner than asking every contributor to reconstruct a local setup.

For the best results:

- Keep one workspace per assistant so context stays focused.
- Store release inputs as plain text artifacts that can be reviewed.
- Pass API keys through local environment variables only.
- Copy verified commands into the release checklist.
- Delete temporary files that contain private or unreleased information.

## Conclusion

AI-assisted release review works best when the assistants have a narrow job.
Omni Engineer maps the release surface. Claude Engineer challenges the plan.
Daytona keeps both runs isolated and repeatable. The maintainer still owns the
final decision, but the process catches gaps earlier and leaves a review trail
that is easier to trust.

That combination is the point: a clean workspace, two independent AI passes,
and a human-verified checklist before the release goes out.

## References

- [Daytona](https://www.daytona.io/)
- [Omni Engineer](https://github.com/Doriandarko/omni-engineer)
- [Claude Engineer](https://github.com/Doriandarko/claude-engineer)
- [Dev Containers](https://containers.dev/)
