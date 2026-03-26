# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Claude Code **skill** (not a traditional codebase) that automates Israeli online supermarket shopping via browser automation. The entire skill is defined in `SKILL.md`, which Claude Code loads as a slash command (`/superme`). Supports **Shufersal**, **Keshet Teamim**, **Rami Levy**, and **Tiv Taam**; Yochananof and Victory are planned.

## Installation

### Via `npx skills` (Recommended)

```bash
npx skills add aloncarmel/supermeskill -g
```

### Manual

```bash
mkdir -p ~/.claude/skills/superme
cp SKILL.md ~/.claude/skills/superme/
```

Prerequisite: `uv tool install browser-use --python 3.12`

## How It Works

The skill uses `browser-use` CLI to drive a headed Chromium session (`--session superme`) against Shufersal's website. There is no backend or build step — all logic lives in the SKILL.md prompt that instructs Claude how to orchestrate browser commands and API calls.

### Key Patterns

- **Browser automation**: `browser-use --session superme open|eval|cookies` commands. Hebrew text must be injected via `eval` (not `type`).
- **Vendor detection**: `/tmp/superme_vendor.txt` stores `shufersal` or `keshet`. All commands branch behavior based on this file.
- **Shufersal**: Vue.js site. Uses `curl` + exported cookies for REST endpoints (orders), `$.ajax`/`fetch` for CSRF-protected endpoints (wishlists). Wishlist mutations go through Vue instance at `#personalAreaContainer.__vue__`.
- **Keshet Teamim**: Prutah platform (AngularJS). Uses Bearer token auth. Cart operations via `angular.element(document.body).injector().get('Cart')`. Search can be done via direct API calls (`curl` with token).
- **Session state**: Temp files under `/tmp/superme_*` — vendor-specific files for cookies, tokens, cart IDs, etc.

### Known Gotchas

**Shufersal:**
- WAF blocks product names containing `'` — the initial save usually succeeds despite the 426 error dialog.
- Wishlist names cannot contain `/` (causes 500). Use `DD-MM-YYYY` format.
- `document.cookie` is incomplete (missing httpOnly) — always use `browser-use cookies export`.
- Cart items don't persist across sessions; always use wishlists instead.

**Keshet Teamim:**
- Login requires SMS OTP — user must provide the code manually.
- First add-to-cart triggers a delivery options dialog that the user must handle in the browser.
- Prices/availability are branch-specific (default branch `2585`).

## Making Changes

When editing `SKILL.md`:
- The frontmatter (`name`, `description`, `risk`) controls how Claude Code discovers and describes the skill.
- Each `### /superme <command>` section is self-contained — Claude follows the steps sequentially.
- API endpoint details and Vue component access patterns are in the "Key Technical Details" section at the bottom; commands reference these shared patterns.
- Test changes by running the slash commands in Claude Code with a live Shufersal account.
