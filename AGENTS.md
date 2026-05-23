# AGENTS.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

This is the discovery and documentation layer for the [J8k3/CyberChef](https://github.com/J8k3/CyberChef) payments fork. It contains the recipe catalog, user-facing README, screenshots, and validation status tables. Implementation lives in the CyberChef fork; this repo is what users read first and what recipe links point to.

## Session Start

- At the start of a session, sync with `origin/master` before doing substantive work.
- Preferred command: `git pull --rebase origin master`
- Only do this automatically when the worktree is clean. If local changes are already present, inspect before rebasing.

## Commit Scope

- Keep commits small and reviewable by default.
- Prefer one commit per logical change — a single coherent unit a reviewer can evaluate independently.
- Group related changes (e.g., a new feature + its test + the knowledge-base entry it required) into one commit when they can't be evaluated independently.
- Prefer squash or amend for iterative follow-ups — if a second commit only fixes or extends the immediately preceding one, squash rather than leaving noise in the log.
- Do not split a change just to make it look smaller; split when a reviewer would genuinely benefit from evaluating the pieces independently.
- When CI flags a lint or test failure after a push, fix locally and **amend or squash into the failing commit** (using `git push --force-with-lease`) rather than adding a new fix commit on top.

## Content Maintenance

This repo documents what the CyberChef fork does. Keep it in sync with the fork:

- When a payment operation is added, renamed, or removed in `J8k3/CyberChef`, update the recipe catalog table in `README.md`.
- When a recipe URL changes (due to op rename or arg reorder), update the link in `README.md`.
- When APC cross-validation results change, update the Validation Status table.
- Screenshots live in `screenshots/`. Replace when the UI changes materially; do not accumulate stale screenshots.

## Knowledge Contribution

When working in this repo and new payment domain knowledge surfaces — a PCI rule, an algorithm edge case, an APC API constraint — write it back into `W:\aws-payment-cryptography-mcp\payment-knowledge-base.md` in the same session. Do not defer.

## Security Constraint

**Never mention AWS, APC, or AWS Payment Cryptography in any user-facing content in this repo.** Internal dev notes are fine, but the README and recipe catalog are public-facing.
