---
name: to-spec
description: Turn the current conversation into a spec and publish it to the project issue tracker — no interview, just synthesis of what you've already discussed.
disable-model-invocation: true
---

This skill takes the current conversation context and codebase understanding and produces a **spec**. Do NOT interview the user — just synthesize what you already know.

A spec is not a PRD, and in this repo the two are different live artifacts. The PRD (`docs/prd.md`) owns the product and its requirements, and lists the specs. A spec delivers a slice of those requirements. Never treat one as another name for the other.

Read the `## Issue Tracker` section of the project's instructions file (`AGENTS.md`, or `CLAUDE.md` when there is no `AGENTS.md`), then `docs/agents/issue-tracker.md` — together they have the tracker's commands and the triage label vocabulary. If either is missing, stop immediately:

> ⛔ Issue tracker not configured (missing `## Issue Tracker` in the instructions file or `docs/agents/issue-tracker.md`).
> Run `/setup-project` before using this skill.

## Process

1. Explore the repo to understand the current state of the codebase, if you haven't already. Read `docs/agents/architecture.md` first — it carries the module map, the test seams, and the invariants of this repo. Use its vocabulary throughout the spec, and respect the invariants in the area you're touching.

   **Then read `docs/prd.md`, if it exists.** It is the source of truth for the product and its requirements, and it is where the requirement ids for `## Covers` come from — that field asks for `FR3`, `NFR2`, … and is unfillable without it. Find the requirements this spec delivers and list their ids. If there is no PRD, write "Standalone — no PRD" and say so to the user; if there is one but nothing in it matches, say that too — it usually means the work is real but the PRD is behind.

2. Sketch out the seams at which you're going to test the feature. Existing seams should be preferred to new ones. Use the highest seam possible. If new seams are needed, propose them at the highest point you can. The fewer seams across the codebase, the better - the ideal number is one.

Check with the user that these seams match their expectations.

3. Write the spec using the template below, then publish it to the project issue tracker. Apply the `ready-for-agent` triage label - no need for additional triage.

4. Write a working copy to `.scratch/<spec-slug>/spec.md` — same content, plus the issue number on the **first line**, in the form `#<N>`. That line is not decoration: `implement` locates this file by grepping for the issue number, because the slug is not derivable from the issue. The file is gitignored and disposable; the issue is canonical. If they ever diverge, the issue wins.

5. Say whether the spec fits one context window as a single unit of work. If it does, tell the user they can run `/implement <issue>` directly. If it doesn't, point at `/to-tickets <issue>` — that skill owns the slicing decision, not this one.

<spec-template>

## Covers

The requirement ids from `docs/prd.md` this spec delivers (`FR3`, `NFR2`, …), or "Standalone — no PRD" when the spec did not come from one. This is what lets the reviewer check coverage against a requirement instead of trusting the ticket text — `implement` hands the reviewer the path to this spec, and to the PRD when this field names ids.

## Problem Statement

The problem that the user is facing, from the user's perspective.

## Solution

The solution to the problem, from the user's perspective.

## User Stories

A LONG, numbered list of user stories. Each user story should be in the format of:

1. As an <actor>, I want a <feature>, so that <benefit>

<user-story-example>
1. As a mobile bank customer, I want to see balance on my accounts, so that I can make better informed decisions about my spending
</user-story-example>

This list of user stories should be extremely extensive and cover all aspects of the feature.

## Implementation Decisions

A list of implementation decisions that were made. This can include:

- The modules that will be built/modified
- The interfaces of those modules that will be modified
- Technical clarifications from the developer
- Architectural decisions
- Schema changes
- API contracts
- Specific interactions

Do NOT include specific file paths or code snippets. They may end up being outdated very quickly.

Exception: if a prototype produced a snippet that encodes a decision more precisely than prose can (state machine, reducer, schema, type shape), inline it within the relevant decision and note briefly that it came from a prototype. Trim to the decision-rich parts — not a working demo, just the important bits.

## Testing Decisions

A list of testing decisions that were made. Include:

- A description of what makes a good test (only test external behavior, not implementation details)
- Which modules will be tested
- Prior art for the tests (i.e. similar types of tests in the codebase)

## Out of Scope

A description of the things that are out of scope for this spec.

## Further Notes

Any further notes about the feature.

</spec-template>
