---
name: grill-with-issues
description: Grill the user about a GitHub issue until you reach a shared understanding, then rewrite the issue as an implementable ticket and route it — needs-prototype when it changes what the user sees, otherwise ready-for-agent when it fits one /implement session, needs-spec when it doesn't. Use with an issue number or URL, e.g. an issue labeled needs-triage.
disable-model-invocation: true
---

Interview me relentlessly about every aspect of this issue until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer. Ask the questions in batches, with a recommended answer on each one. If a question can be answered by exploring the codebase, explore the codebase instead. Do not act on it until the user confirms we have reached a shared understanding.

This is `grill-me` working on an issue instead of a loose plan. It is a separate skill: it does not replace or change `grill-me`.

Read the `## Issue Tracker` section of the project's instructions file (`AGENTS.md`, or `CLAUDE.md` when there is no `AGENTS.md`), then `docs/agents/issue-tracker.md` — together they have the tracker's commands and the triage label vocabulary. If either is missing, stop immediately:

> ⛔ Issue tracker not configured (missing `## Issue Tracker` in the instructions file or `docs/agents/issue-tracker.md`).
> Run `/setup-project` before using this skill.

## Process

### 1. Read the issue

The argument is an issue number or URL. If there is none, ask for it — this skill has no conversation-only mode. Fetch the issue with the tracker's "Ver" command and read its full body and every comment. Check for a parent too ("Ver pai"): a parent spec is context, and its decisions are not up for re-asking.

### 2. Explore before asking

Read `docs/agents/architecture.md` first, if it exists — module map, test seams, invariants. Then read the code the issue touches, and `docs/prd.md` if the issue maps to a requirement. Everything the code, the issue or the PRD already answers is not a question. Use the vocabulary of `architecture.md` in the questions.

### 3. Grill in batches

Ask the questions in batches, grouped by branch of the design tree, ordered so a batch never depends on an answer from the same batch. Each question carries your recommended answer. When a later answer opens a new branch, ask the next batch. Keep going until nothing is left unresolved.

Ask product and risk decisions only: behavior, scope, edge cases a user would notice, what is out of scope. Seams, module boundaries and slicing are code decisions — decide them yourself and state them, as `to-spec` and `to-tickets` do.

### 4. Confirm the shared understanding

Summarize what was agreed in plain product language and ask the user to confirm. Nothing below happens before an explicit confirmation. If the user corrects the summary, apply the correction and confirm again.

### 5. Rewrite the issue as a ticket

Write the new body to `.scratch/issue-<N>/body.md` and apply it with the tracker's edit command from that file (`--body-file` or the tracker's equivalent — never an inline `--body "..."`: markdown carries backticks, quotes, `$` and `!`, which break inline quoting differently in Bash and PowerShell). The body uses the ticket format `implement` expects, the same as the `<issue-template>` in `to-tickets`:

<ticket-template>

## What to build

The end-to-end behaviour this issue makes work, from the user's perspective — not layer-by-layer implementation.

## Acceptance criteria

- [ ] Criterion 1
- [ ] Criterion 2

</ticket-template>

Add `## Spec` (`#N` of the parent) and `## Blocked by` only when the issue already has a parent spec or blockers. Avoid specific file paths and code snippets — they go stale fast.

### 6. Route

**Visible first.** Before deciding the size, check whether the issue changes anything the user sees: a screen, a layout, interface text, a navigation flow. This is your decision, do not ask.

- **Visible** → `gh issue edit <N> --add-label "needs-prototype" --remove-label "needs-triage"`, in a single operation. Leave the size decision to `close-prototype`, and skip the rest of this step. Tell the user the next step is a session that updates the project's prototype (`docs/agents/prototype.md`), followed by `/close-prototype <N>` in that same session.
- **Not visible** → route by size, below.

**Size.** Decide whether the work fits one fresh context window, as a single `/implement` session — the same criterion `to-spec` and `to-tickets` use. This is a code decision: decide it yourself, do not ask.

Move the labels in a single operation, never add without removing (the vocabulary is exclusive):

- **Fits** → `gh issue edit <N> --add-label "ready-for-agent" --remove-label "needs-triage"`. Tell the user they can run `/implement <N>`.
- **Doesn't fit** → `gh issue edit <N> --add-label "needs-spec" --remove-label "needs-triage"`, and post a comment with the summary of the conversation — the decisions, the rejected options and why, what is out of scope — written to a file and sent with `--body-file`. `to-spec` reads that comment as source, so write it for someone who did not see this conversation. Tell the user the next step is `/to-spec <N>`.

The label edit is the tracker's "Editar labels" command; the comment is `gh issue comment <N> --body-file <file>` on GitHub, or the tracker's equivalent. Do not close the issue.
