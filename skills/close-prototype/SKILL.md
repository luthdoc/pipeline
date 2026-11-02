---
name: close-prototype
description: Close the prototype step opened by /prototype. Initial prototype (docs/agents/prototype.md without a link) — record the approved artifact's name and link in docs/agents/prototype.md by a direct commit to main, after the user OKs the diff. Change (an issue labeled needs-prototype) — in the same session where the prototype was adjusted, merge the issue's decisions with the decisions taken on the prototype, confirm them, rewrite the issue as an implementable ticket that points to the approved prototype, and route it — ready-for-agent when it fits one /implement session, needs-spec when it doesn't.
disable-model-invocation: true
---

## Moment

Run this in the same session as `/prototype`. The moment is the one it detected: run `git fetch origin` and read `docs/agents/prototype.md` from `origin/main`.

- **Initial prototype** (file without a link): there is no issue. Follow only "Initial prototype", right below, and stop there.
- **Change** (file with a link, `needs-prototype` issue): skip to "Change". From there on, the process is the usual one.

### Initial prototype

1. Confirm with the user that the prototype is approved and which artifact it is (name and link). Without explicit approval, write nothing.
2. Write `Nome:` and `Link:` in `docs/agents/prototype.md`, on an up-to-date copy of `origin/main`. Do not touch the rest of the file.
3. Follow the pipeline's small-scope rule: show the diff, wait for the user's OK, and only then commit directly to `main` and push. Never through a PR opened during the adjustments, and never force push. If the push is rejected, run `git pull --rebase` from `origin/main` and show the diff again.
4. Tell the user the prototype is recorded and that the next visible changes come in through a `needs-prototype` issue (`/prototype <N>`).

### Change

Run this in the same session where the project's prototype was adjusted for the issue. The prototype is the project's single claude.ai artifact, recorded in `docs/agents/prototype.md`. What changed on it during this session is half of the source; the issue is the other half.

Read the `## Issue Tracker` section of the project's instructions file (`AGENTS.md`, or `CLAUDE.md` when there is no `AGENTS.md`), then `docs/agents/issue-tracker.md` — together they have the tracker's commands and the triage label vocabulary. If either is missing, stop immediately:

> ⛔ Issue tracker not configured (missing `## Issue Tracker` in the instructions file or `docs/agents/issue-tracker.md`).
> Run `/setup-project` before using this skill.

## Process

### 1. Find the issue

The argument is an issue number or URL. If there is none, use the issue this session has been working on. If the session has none either, ask for it. Fetch the issue with the tracker's "Ver" command and read its full body and every comment.

### 2. Gather the two sources

- **The issue** — the decisions from the grill, in its body and comments.
- **The prototype** — every change made to the artifact in this session, and every decision the user took while adjusting it: what was added, removed, moved or reworded, and what was tried and rejected. Read `docs/agents/prototype.md` for the artifact's name and link, and note the version of the artifact the user approved.

Where the prototype contradicts the issue, the prototype wins: it is the later decision. Say so in the summary.

### 3. Confirm the shared understanding

Summarize the result in plain product language: what the user will see and do, what changed from the original issue, what is out of scope. Ask the user to confirm explicitly. Nothing below happens before that confirmation. If the user corrects the summary, apply the correction and confirm again.

### 4. Rewrite the issue as a ticket

Write the new body to `.scratch/issue-<N>/body.md` and apply it with the tracker's edit command from that file (`--body-file` or the tracker's equivalent — never an inline `--body "..."`: markdown carries backticks, quotes, `$` and `!`, which break inline quoting differently in Bash and PowerShell). The body uses the ticket format `implement` expects, the same as `grill-with-issues` and the `<issue-template>` in `to-tickets`, plus the reference to the approved prototype:

<ticket-template>

## What to build

The end-to-end behaviour this issue makes work, from the user's perspective — not layer-by-layer implementation.

## Acceptance criteria

- [ ] Criterion 1
- [ ] Criterion 2

## Prototype

[Artifact name](artifact link) — approved version: [version or line the user approved].

</ticket-template>

Add `## Spec` (`#N` of the parent) and `## Blocked by` only when the issue already has a parent spec or blockers. Avoid specific file paths and code snippets — they go stale fast.

### 5. Route by size

Decide whether the work fits one fresh context window, as a single `/implement` session — the same criterion `grill-with-issues`, `to-spec` and `to-tickets` use. This is a code decision: decide it yourself, do not ask.

Move the labels in a single operation, never add without removing (the vocabulary is exclusive):

- **Fits** → `gh issue edit <N> --add-label "ready-for-agent" --remove-label "needs-prototype"`. Tell the user they can run `/implement <N>`.
- **Doesn't fit** → `gh issue edit <N> --add-label "needs-spec" --remove-label "needs-prototype"`, and post a comment with the summary of the conversation — the decisions, the rejected options and why, what is out of scope — written to a file and sent with `--body-file`. `to-spec` reads that comment as source, so write it for someone who did not see this conversation. Tell the user the next step is `/to-spec <N>`.

The label edit is the tracker's "Editar labels" command; the comment is `gh issue comment <N> --body-file <file>` on GitHub, or the tracker's equivalent. Do not close the issue.

### 6. Visual principles

If the session settled a visual principle that applies to other screens too — not just this one — and the project has a design doc (`docs/design.md` or `DESIGN.md`), suggest recording it there, with the exact text. Do not apply it without the user's OK.
