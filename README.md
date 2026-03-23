# Superme Skill for Claude Code

Online supermarket automation skill for [Claude Code](https://claude.ai/claude-code). Supports **Shufersal**, **Keshet Teamim**, and **Rami Levy**.

## Features

- **Login** — Authenticate to any supported vendor (email/password, phone/SMS OTP, or email/SMS OTP)
- **Search** — Find products with prices, transliterated for terminal display
- **Add to Cart/List** — Add search results to a wishlist (Shufersal) or cart (Rami Levy, Keshet Teamim)
- **Magicorder** — Consolidate past orders into one list/cart (deduped)
- **Lists / Cart** — View your wishlists or cart contents

## Installation

Copy `SKILL.md` to your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/superme
cp SKILL.md ~/.claude/skills/superme/
```

### Prerequisites

- [browser-use](https://github.com/browser-use/browser-use) CLI (auto-installed on first run)
- An account with at least one supported vendor

## Usage

```
/superme login shufersal <email> <password>
/superme login rami <email>
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
- **Keshet Teamim**: Prutah platform (AngularJS). Bearer token auth, REST API for search, Angular `Cart` service for cart operations.

## Supported Vendors

| Vendor | Platform | Login | Search | Cart | Lists | Status |
|--------|----------|-------|--------|------|-------|--------|
| Shufersal | Custom (Vue.js) | Email + password | Browser scrape | Wishlist (Vue API) | Wishlist API | **Supported** |
| Rami Levy | Nuxt.js (Vue SSR) | Email + SMS OTP | REST API | REST API | REST API | **Supported** |
| Keshet Teamim | Prutah (AngularJS) | Phone + SMS OTP | REST API | Angular service | — | **Supported** |
| Yochananof | — | — | — | — | — | Planned |
| Victory | — | — | — | — | — | Planned |
| Tiv Taam | — | — | — | — | — | Planned |
| Osher Ad | — | — | — | — | — | Planned |

## License

MIT
