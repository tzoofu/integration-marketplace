---
name: implement-integration
description: Implement an integration from this catalog's playbook into a target repo — find the playbook, inspect the target's stack and conventions, show a plan, apply the setup steps and core pattern adapted to the target, then verify with the target's own checks. Use when the user says "implement <integration> in <repo>", "add <provider> to <repo> using the catalog", or hands over a catalog category or playbook to build.
argument-hint: <integration or category> <target repo path>
allowed-tools: Read, Grep, Glob, Edit, Write, Bash, Agent, AskUserQuestion
disable-model-invocation: true
---

# implement-integration

The reverse of `map-integrations`: it reads a playbook from `catalog/` and builds that integration inside a target repo. The catalog is public, the target usually is not, so information flows one way: **catalog -> target, never target -> catalog**.

## Hard rules

- Read `catalog/`; write only inside the target repo. Never edit anything under `catalog/`, and never read `../pim`.
- Never write the target's repo name, path, project IDs or security findings into this repo. If the playbook is wrong or stale, say so in chat; the user fixes it in a separate catalog edit.
- Env vars: add names only (to the target's `.env.example` or equivalent). Never invent, generate or copy secret values.
- Never commit or push the target repo. Leave changes in its working tree.
- If the target's working tree has unrelated uncommitted changes, ask before editing.

## Step 1: Resolve the input

You need an integration (or category) and a target repo path. If either is missing, ask for it. Do not ask for a path the user already gave or that is already an additional working directory.

Find the playbook: open `catalog/README.md`, then `catalog/<category>/`. If several playbooks fit (different providers, or variants of one), choose by the target's stack once Step 3 has run, or ask the user. If nothing fits, stop and say so; do not invent a playbook from memory.

## Step 2: Read the playbook

Read the whole file plus any `catalog/<category>/assets/` files it links. Note:

- `Playbook confidence`: `high` = extracted from real source; `medium` = one implementation; `low` = synthesized from provider docs. For `low`, tell the user and check the provider's current docs (`docs:` header link) wherever the playbook and docs disagree.
- Variants (for example web/server vs native-mobile) and which one matches the target.
- Prerequisites, env vars and Gotchas; they become the plan and the verification checklist.

## Step 3: Inspect the target repo (read-only)

Use an Explore agent if the repo is large. Collect:

- Framework, language, package manager, and how env vars are loaded.
- Folder layout and naming conventions; where similar integrations or clients already live.
- The scripts for typecheck, lint, build and tests.
- Whether this provider or a competing one is already wired in. If it is wired in a different way, stop and ask rather than layering a second setup on top.

## Step 4: Plan and get approval

Present a short plan before editing:

- Files to add or change (paths in the target).
- Packages to install.
- Env var names to add.
- Manual steps only the user can do: creating accounts, keys or console configuration, with the playbook's exact wording.
- Anything in the playbook you will skip or adapt, and why.

Wait for the user's OK.

## Step 5: Apply

- Follow the playbook's Setup steps in order.
- Adapt the Core pattern to the target's language, folder layout and naming. It is a starter, not text to paste. The exception is `assets/` files the playbook marks copy-verbatim.
- Install packages with the target's own package manager.
- Do not add features, abstractions or config the playbook does not call for.

## Step 6: Verify

- Run the target's own typecheck, lint, build and tests, whichever exist. Fix what you broke; report what was already failing.
- Walk the playbook's Gotchas as a checklist against what you wrote and say which are handled.
- State plainly what could not be verified (live credentials, deployed console settings, third-party callbacks).

## Step 7: Report

Keep it short:

- What was added or changed (file list, not file contents).
- Manual steps the user still has to do.
- Playbook gaps you hit: missing steps, outdated versions or wrong names. Tell the user to fix the catalog separately; do not edit it here.
- Optional follow-up the user may choose: run `map-integrations` on the target so the catalog's adoption count reflects it.

## Task

$ARGUMENTS
