# Superme Skill for Claude Code

Online supermarket automation skill for [Claude Code](https://claude.ai/claude-code). Supports **Shufersal** and **Keshet Teamim**.

## Features

- **Login** — Authenticate to Shufersal (email/password) or Keshet Teamim (phone/SMS OTP)
- **Search** — Find products with prices, transliterated for terminal display
- **Add to Cart/List** — Add search results to a wishlist (Shufersal) or cart (Keshet Teamim)
- **Magicorder** — Consolidate your last 5 orders into one list/cart (deduped)
- **Lists / Cart** — View your wishlists or cart contents

## Installation

Copy `SKILL.md` to your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/superme
cp SKILL.md ~/.claude/skills/superme/
```

### Prerequisites

- [browser-use](https://github.com/browser-use/browser-use) CLI: `uv tool install browser-use --python 3.12`
- A Shufersal Online account and/or a Keshet Teamim account

## Usage

```
/superme login shufersal <email> <password>
/superme login keshet <phone>
/superme search <product>
/superme add <#>
/superme magicorder
/superme lists                  (Shufersal)
/superme cart                   (Keshet Teamim)
/superme close
/superme help
```

## How it works

- Uses `browser-use` CLI to automate a Chromium browser session
- **Shufersal**: Interacts with Vue.js frontend and REST APIs. Manages wishlists via `/wishlist/*` endpoints. Products added through the Vue component tree.
- **Keshet Teamim**: Runs on the Prutah platform (AngularJS). Uses Bearer token auth and direct REST API for search. Products added to server-side cart via Angular `Cart` service.

## Supported Vendors

| Vendor | Platform | Login | Status |
|--------|----------|-------|--------|
| Shufersal | Custom (Vue.js) | Email + password | Supported |
| Keshet Teamim | Prutah (AngularJS) | Phone + SMS OTP | Supported |
| Rami Levy | — | — | Planned |
| Yochananof | — | — | Planned |
| Victory | — | — | Planned |

## License

MIT
