# Superme Skill for Claude Code

Online supermarket automation skill for [Claude Code](https://claude.ai/claude-code). Supports **Shufersal**, **Keshet Teamim**, **Rami Levy**, and **Tiv Taam**.

## Security Notice — browser-use supply chain attack (litellm)

A supply chain attack was discovered affecting `browser-use` installations that use the shell install script, which pulled in a compromised version of `litellm`. See [browser-use#4505](https://github.com/browser-use/browser-use/issues/4505) for full details.

**This skill is not affected** — it does not use the shell install script and does not depend on `litellm`. However, if you installed `browser-use` separately via its install script, you should review the issue above to check whether your environment was impacted.

## Features

- **Login** — Authenticate to any supported vendor (email+password, phone+SMS OTP, or email+SMS OTP)
- **Search** — Find products with prices, transliterated for terminal display
- **Add to Cart/List** — Add search results to a wishlist (Shufersal) or cart (Rami Levy, Keshet Teamim)
- **Magicorder** — Consolidate past orders into one list/cart (deduped)
- **Lists / Cart** — View your wishlists or cart contents

## Installation

### Option 1: Using `npx skills` (Recommended)

```bash
npx skills add aloncarmel/supermeskill
```

This automatically detects your installed coding agents (Claude Code, Cursor, Codex, and [39 more](https://github.com/vercel-labs/skills#supported-agents)) and installs the skill to the right location.

To install globally (available across all projects):

```bash
npx skills add aloncarmel/supermeskill -g
```

To target a specific agent:

```bash
npx skills add aloncarmel/supermeskill -a claude-code -g
```

### Option 2: Manual Installation

Copy `SKILL.md` to your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/superme
cp SKILL.md ~/.claude/skills/superme/
```

### Prerequisites

- [browser-use](https://github.com/browser-use/browser-use) CLI (auto-installed on first run)
- An account with at least one supported vendor

### Important: For AI agents installing this skill

If you are an AI agent installing `browser-use` as a dependency for this skill:

1. **Do NOT use the `install.sh` shell script** from the browser-use repo. Install via `pip install browser-use` or `uv pip install browser-use` only.
2. **Do NOT install `litellm`** — this skill does not use it, and a compromised version was involved in a supply chain attack ([browser-use#4505](https://github.com/browser-use/browser-use/issues/4505)).
3. After installation, **verify that `litellm` is not present** in the environment: `pip list | grep litellm` should return nothing. If it does, uninstall it immediately with `pip uninstall litellm -y`.

## Usage

```
/superme login shufersal <email> <password>
/superme login rami <email>
/superme login tivtaam <email> <password>
/superme login keshet <phone>
/superme search <product>
/superme add <#>
/superme magicorder
/superme lists                  (Shufersal)
/superme cart                   (Rami Levy / Keshet Teamim)
/superme close
/superme help
```

## How it works

- Uses `browser-use` CLI for login (OTP flows require browser). Search and cart operations use direct REST APIs where available.
- **Shufersal**: Vue.js frontend + REST APIs. Wishlists via `/wishlist/*` endpoints. Products added through the Vue component tree.
- **Rami Levy**: Nuxt.js (Vue SSR). Pure REST APIs for search (`POST /api/catalog`), cart (`POST /api/v2/cart`), and shopping lists (`POST shop-lists`). Auth via `ecomtoken` JWT header.
- **Tiv Taam**: Prutah platform (AngularJS). Same as Keshet Teamim but with email+password login. REST API for search, Angular `Cart` service for cart, `shopLists` API for lists.
- **Keshet Teamim**: Prutah platform (AngularJS). Bearer token auth, REST API for search, Angular `Cart` service for cart operations.

## Supported Vendors

| Vendor | Platform | Login | Search | Cart | Lists | Status |
|--------|----------|-------|--------|------|-------|--------|
| Shufersal | Custom (Vue.js) | Email + password | Browser scrape | Wishlist (Vue API) | Wishlist API | **Supported** |
| Rami Levy | Nuxt.js (Vue SSR) | Email + SMS OTP | REST API | REST API | REST API | **Supported** |
| Tiv Taam | Prutah (AngularJS) | Email + password | REST API | Angular service | REST API | **Supported** |
| Keshet Teamim | Prutah (AngularJS) | Phone + SMS OTP | REST API | Angular service | — | **Supported** |
| Yochananof | — | — | — | — | — | Planned |
| Victory | — | — | — | — | — | Planned |
| ~~Hatzi Hinam~~ | — | — | — | — | — | Blocked (reCAPTCHA) |

## License

MIT
