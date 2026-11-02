---
name: prototype
description: >
  Open the prototype session for the project's screens and work out which moment it is
  from the state of the repo. Initial prototype: `docs/agents/prototype.md` has no link —
  create the artifact from `docs/prd.md` on `origin/main`. Change: there is already a link
  and an issue labeled needs-prototype came in — change the existing artifact with the
  issue's decisions. In both cases the session stays open for adjustments and ends with
  /close-prototype. Use: /prototype (initial prototype) or /prototype 12 (needs-prototype issue).
disable-model-invocation: true
---

# Skill: prototype

The project's prototype is a claude.ai artifact, a single one per project, with the screens. It never becomes a route in the code. This skill opens the session where it is created or changed, and `/close-prototype`, in the same session, closes the step.

## 1. Find the moment

Run `git fetch origin` and read `docs/agents/prototype.md` from `origin/main` (`git show origin/main:docs/agents/prototype.md`), not from the local branch, which may be stale.

- **No link** (or no file) → **initial prototype**. If an issue came as the argument, say it has to wait: the project needs its initial prototype first.
- **Link and a `needs-prototype` issue** as the argument → **change**.
- **Link and no issue** → ask which `needs-prototype` issue. Do not create a second artifact.

Tell the user in one line which moment you detected.

## 2. Initial prototype

1. Read `docs/prd.md` from `origin/main` (`git show origin/main:docs/prd.md`). If it does not exist, stop and tell the user: the initial prototype comes from the PRD, and the previous step is `/to-prd`.
2. Look for a design doc on `origin/main` (`docs/design.md`, or `DESIGN.md` at the root).
   - **It exists** → follow its design specifications.
   - **It does not exist** → a minimalist, neutral wireframe: shades of gray, system font, hierarchy by size and weight only, images as a box with an X. No colors, fonts or brand identity.
3. With or without a design doc, the prototype carries the user journey flows and the PRD's specifications.
4. Always create the artifact as a **Design** type (`/design`), a single one per project: mobile (390×844) and desktop (1280×800) artboards, linked to each other, with a navigable Play.
5. Do not invent real data (contact, address, price, people's names). Sample content is marked as a sample.
6. Give the user the artifact's name and link and stay open for adjustments.

**Do not touch the repo during the adjustments:** no commit, branch or PR. The name and link only go into `docs/agents/prototype.md` through `/close-prototype`, after approval.

## 3. Change

Read the issue with the "Ver" command from `docs/agents/issue-tracker.md` (body and every comment). Change the artifact named in `docs/agents/prototype.md` with the issue's decisions, without creating another one, and stay open for adjustments.

## In both moments

- **Adjustments come through this session's chat.** A comment left directly on the artifact does not reach the session; if the user mentions one, ask them to send the request here.
- **Never force push.** If the session's branch was deleted on merge, create a branch with a new name from an up-to-date `origin/main`.
- When the user approves the prototype, the next step is `/close-prototype` in this same session (`/close-prototype <N>` for a change).
