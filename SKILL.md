---
name: superme
description: "Online supermarket automation. Login, search products, add to cart/wishlists, magicorder from past orders. Supports Shufersal and Keshet Teamim. Usage: /superme login <vendor>, /superme search <product>, /superme add <#>, /superme magicorder, /superme lists, /superme cart, /superme help"
risk: medium
source: local
date_added: "2026-03-21"
---

# /superme — Online Supermarket Automation

Automate Israeli online supermarkets using browser-use CLI. Supports **Shufersal** and **Keshet Teamim**.

---

## Prerequisites (auto-install)

**IMPORTANT:** Before executing ANY command below, always run this first to ensure browser-use is installed:
```bash
which browser-use 2>/dev/null || uv tool install browser-use --python 3.12
```
This is a no-op if already installed, and auto-installs if missing. Run it silently at the start of every `/superme` invocation.

---

## Vendor Detection

The skill supports multiple vendors. The active vendor is stored in `/tmp/superme_vendor.txt` (set during login). All commands read this file to determine which vendor flow to use.

**Vendor identifiers:**
- `shufersal` — Shufersal Online (shufersal.co.il)
- `keshet` — Keshet Teamim (keshet-teamim.co.il)

When a command needs to branch by vendor, read the vendor file:
```bash
cat /tmp/superme_vendor.txt 2>/dev/null || echo "none"
```
If `none`, tell the user to run `/superme login` first.

---

## Session State

The skill tracks state across commands within a conversation using temp files:

### Shared (all vendors)
- `/tmp/superme_vendor.txt` — active vendor identifier
- `/tmp/superme_last_search.json` — last search results with product IDs for `/superme add`

### Shufersal-specific
- `/tmp/superme_cookies.json` — exported browser cookies
- `/tmp/superme_cookie_str.txt` — cookie string for curl API calls
- `/tmp/superme_session_list_id.txt` — current session's wishlist ID (created on first add)

### Keshet Teamim-specific
- `/tmp/superme_keshet_token.txt` — Bearer token for API calls
- `/tmp/superme_keshet_user_id.txt` — user ID for API calls
- `/tmp/superme_keshet_cart_id.txt` — server cart ID

---

## Commands

### `/superme help`

Show available commands:
```
/superme login shufersal <email> <password>   Log in to Shufersal Online
/superme login keshet <phone>                 Log in to Keshet Teamim (SMS OTP)
/superme search <product>                     Search for a product (Hebrew or English)
/superme add <#>                              Add search result # to cart/wishlist
/superme magicorder                           Consolidate last 5 orders into one list
/superme lists                                View Superme wishlists (Shufersal)
/superme cart                                 View current cart (Keshet Teamim)
/superme close                                Close the browser session
```

---

### `/superme login`

Log in to a supermarket. Vendor is required as the first argument.

---

#### `/superme login shufersal`

Log in to Shufersal Online.

**Steps:**

1. Ask the user for credentials if not provided inline. Parse from args if given as `/superme login shufersal user@email.com password123`.

2. Open login page:
```bash
browser-use --headed --session superme open "https://www.shufersal.co.il/online/he/login"
```

3. Wait for load, then fill credentials via JavaScript (Shadow DOM inputs):
```bash
browser-use --session superme eval "document.getElementById('j_username').value = 'EMAIL'; document.getElementById('j_username').dispatchEvent(new Event('input', {bubbles: true}));"
browser-use --session superme eval "document.getElementById('j_password').value = 'PASS'; document.getElementById('j_password').dispatchEvent(new Event('input', {bubbles: true}));"
```

4. Get state, find submit button (`<button type=submit />` inside `<form id=loginForm />`), click it.

5. Wait 5 seconds, verify: `window.location.href` should NOT contain `/login`.

6. After login, export cookies for API calls:
```bash
browser-use --session superme cookies export /tmp/superme_cookies.json
```
Then build the cookie string:
```python
import json
with open('/tmp/superme_cookies.json') as f:
    cookies = json.load(f)
shufersal = [c for c in cookies if 'shufersal' in c.get('domain','')]
cookie_str = '; '.join(f"{c['name']}={c['value']}" for c in shufersal)
with open('/tmp/superme_cookie_str.txt', 'w') as f:
    f.write(cookie_str)
```

7. Save vendor:
```bash
echo "shufersal" > /tmp/superme_vendor.txt
```

**IMPORTANT:** Never store or log the user's password. Only use it in the eval command.

---

#### `/superme login keshet`

Log in to Keshet Teamim via phone number + SMS OTP code.

**Steps:**

1. Ask the user for their phone number if not provided inline. Parse from args if given as `/superme login keshet 0501234567`.

2. Open the homepage:
```bash
browser-use --headed --session superme open "https://www.keshet-teamim.co.il/"
```

3. Wait for load, then click the account/login button to open the verification modal:
```bash
browser-use --session superme eval "document.querySelector('.user-verification-btn, [ng-click*=openVerification], .login-btn').click();"
```

4. Wait 3 seconds for the modal to appear. Fill the phone number via the AngularJS model:
```bash
browser-use --session superme eval "var scope = angular.element(document.querySelector('[ng-model=\"userVerificationCtrl.phoneNumber\"]')).scope(); scope.userVerificationCtrl.phoneNumber = 'PHONE_NUMBER'; scope.\$apply();"
```

5. Submit the phone number to trigger SMS:
```bash
browser-use --session superme eval "angular.element(document.querySelector('[ng-model=\"userVerificationCtrl.phoneNumber\"]')).scope().userVerificationCtrl.send(angular.element(document.querySelector('[ng-model=\"userVerificationCtrl.phoneNumber\"]')).scope().userVerificationCtrl.phoneNumber);"
```

6. **Ask the user for the SMS code.** Tell them: "I've sent an SMS code to your phone. Please enter the code here."

7. Once the user provides the code, fill it via the OTP input:
```bash
browser-use --session superme eval "var inputs = document.querySelectorAll('.otp-input input, [ng-model*=otp], input[type=tel].code-input'); inputs.forEach(function(inp, i) { var char = 'CODE'[i] || ''; inp.value = char; inp.dispatchEvent(new Event('input', {bubbles: true})); });"
```
If the OTP is a single input field instead:
```bash
browser-use --session superme eval "var inp = document.querySelector('.otp-input input, [ng-model*=code]'); inp.value = 'CODE'; inp.dispatchEvent(new Event('input', {bubbles: true}));"
```

8. Click the verify/submit button:
```bash
browser-use --session superme eval "document.querySelector('.verify-btn, button[type=submit], [ng-click*=verify]').click();"
```

9. Wait 5 seconds. Verify login by extracting session data:
```bash
browser-use --session superme eval "var user = angular.element(document.body).injector().get('User'); JSON.stringify({userId: user.session.userId, token: user.session.token, verified: user.isVerified()});"
```

10. Save session data:
```python
import json
# Parse the result from step 9
data = json.loads(RESULT)
with open('/tmp/superme_keshet_token.txt', 'w') as f:
    f.write(data['token'])
with open('/tmp/superme_keshet_user_id.txt', 'w') as f:
    f.write(str(data['userId']))
```

11. Extract cart ID:
```bash
browser-use --session superme eval "angular.element(document.body).injector().get('Cart').serverCartId;"
```
Save to `/tmp/superme_keshet_cart_id.txt`.

12. Save vendor:
```bash
echo "keshet" > /tmp/superme_vendor.txt
```

**IMPORTANT:** Never store or log the user's phone number or OTP code beyond the eval commands.

---

### `/superme search <product>`

Search for a product. Must be logged in first. Behavior depends on active vendor.

---

#### Shufersal search

**Steps:**

1. Check session is active and on shufersal.co.il. If not, navigate to homepage first.

2. Use JavaScript to fill search input and submit (handles Hebrew correctly):
```bash
browser-use --session superme eval "var input = document.getElementById('js-site-search-input'); input.value = 'PRODUCT_HERE'; input.dispatchEvent(new Event('input', {bubbles: true}));"
browser-use --session superme eval "document.querySelector('button[type=submit][aria-label]').click();"
```

3. Wait 5 seconds, get page state.

4. Parse products from state. Each product appears as `<li level=1>` with:
   - `<strong>` — product name
   - Text after — size/weight
   - Next line — brand
   - `<span>` with price
   - Sale prices have "מחיר קודם" (previous price)
   - Product codes appear in `<span id=P_XXXXX>` elements

5. **Save search results** to `/tmp/superme_last_search.json` with product codes, names, prices for use by `/superme add`. Format:
```json
[{"index": 1, "name": "Product Name", "code": "P_XXXXX", "price": "19.90", "size": "400g", "vendor": "shufersal"}, ...]
```

6. Present results using **English only** (terminal can't render Hebrew/RTL). Use transliterated Hebrew names. Columns: #, Product Name, Size, Price, Per 100g, Notes.

7. Filter and prioritize exact matches over loosely related items.

8. Recommend the best value option and tell the user: "Use `/superme add #` to add to your list."

---

#### Keshet Teamim search

**Steps:**

1. Read the auth token from `/tmp/superme_keshet_token.txt`. If missing, tell user to login first.

2. Call the search API directly via curl (no browser needed):
```bash
curl -s 'https://www.keshet-teamim.co.il/v2/retailers/1219/branches/2585/products?appId=4&query=PRODUCT_URL_ENCODED&size=20&from=0&isSearch=true&languageId=1&filters=%7B%22must%22%3A%7B%22exists%22%3A%5B%22family.id%22%2C%22family.categoriesPaths.id%22%2C%22branch.regularPrice%22%5D%2C%22term%22%3A%7B%22branch.isActive%22%3Atrue%2C%22branch.isVisible%22%3Atrue%7D%7D%2C%22mustNot%22%3A%7B%22term%22%3A%7B%22branch.regularPrice%22%3A0%7D%7D%7D' \
  -H 'Accept: application/json' \
  -H "Authorization: Bearer $(cat /tmp/superme_keshet_token.txt)"
```

The `filters` parameter is the URL-encoded version of:
```json
{"must":{"exists":["family.id","family.categoriesPaths.id","branch.regularPrice"],"term":{"branch.isActive":true,"branch.isVisible":true}},"mustNot":{"term":{"branch.regularPrice":0}}}
```

3. Parse the response. Each product in the response has:
   - `id` — product ID (use this for add-to-cart)
   - `localName` — Hebrew product name
   - `brand.name` or `brand.names.1` — brand name
   - `branch.regularPrice` — regular price
   - `branch.salePrice` — sale price (if on sale)
   - `unitOfMeasure` — unit info
   - `weight` — product weight
   - `isWeighable` — whether sold by weight

4. **Save search results** to `/tmp/superme_last_search.json`. Format:
```json
[{"index": 1, "name": "Product Name", "id": "15798599", "price": "12.90", "salePrice": "9.90", "size": "500g", "brand": "Brand", "vendor": "keshet"}, ...]
```

5. Present results using **English only** (terminal can't render Hebrew/RTL). Use transliterated Hebrew names. Columns: #, Product Name, Brand, Size, Price, Sale Price, Notes.

6. Filter and prioritize exact matches over loosely related items.

7. Recommend the best value option and tell the user: "Use `/superme add #` to add to your cart."

---

### `/superme add <#>`

Add a product from the last search results. Behavior depends on active vendor.

---

#### Shufersal add (to wishlist)

**Steps:**

1. Load `/tmp/superme_last_search.json` to get the product by index number.

2. Check if a session wishlist exists by reading `/tmp/superme_session_list_id.txt`:
   - **If file exists**: use the stored list ID
   - **If not**: create a new wishlist (see "Creating a session wishlist" below)

3. Navigate to the wishlist edit page:
```bash
browser-use --session superme open "https://www.shufersal.co.il/online/he/my-account/personal-area/wish-list-2-cart/LIST_ID"
```

4. Wait for load, then add the product via the Vue component:
```javascript
var comp = document.querySelector('#personalAreaContainer').__vue__;
function findComp(vm) {
  if(vm.submitWishList) return vm;
  for(var c of (vm.$children || [])) { var f = findComp(c); if(f) return f; }
  return null;
}
var wl = findComp(comp);

wl.createNewProductSearch({
  code: 'P_XXXXX',
  name: 'Product Name',
  price: {value: 1},
  stock: {stockLevelStatus: {code: 'inStock'}},
  sellingMethod: 'byUnit'
}, 1, 'byUnit');

wl.submitWishList();
```

5. Wait 5 seconds, then verify by checking entries count:
```javascript
store.dispatch('loadWishList', 'LIST_ID').then(function() {
  var entries = store.state.wishList.selectedWishList.entries;
  // entries.length should have increased
});
```

6. If a WAF error dialog appears ("תווים לא חוקיים"), close it — the initial save usually succeeds despite the dialog.

7. Confirm to user: "Added [product name] to your Superme List. [N] items total."

##### Creating a session wishlist

When no session list exists yet:

```javascript
// Must be on a Shufersal page. Use $.ajax (jQuery is loaded on the site):
$.ajax({
  url: ACC.config.encodedContextPath + '/wishlist/createWishlist',
  type: 'POST',
  data: JSON.stringify({name: 'SupermeListDDMMYYYY'}),
  contentType: 'application/json',
  success: function(d) { /* d.shoppingListId is the new list ID */ }
});
```

**IMPORTANT:**
- List name must NOT contain `/` or special characters (causes 500 error). Use format: "Superme List DD-MM-YYYY"
- Save the returned `shoppingListId` to `/tmp/superme_session_list_id.txt`
- The `$.ajax` call must be made from a browser eval on a Shufersal page (CSRF protection)

---

#### Keshet Teamim add (to cart)

**Steps:**

1. Load `/tmp/superme_last_search.json` to get the product by index number.

2. Read cart ID from `/tmp/superme_keshet_cart_id.txt`. If missing, get it from the browser:
```bash
browser-use --session superme eval "angular.element(document.body).injector().get('Cart').serverCartId;"
```

3. Add the product to cart via the Angular Cart service in the browser:
```bash
browser-use --session superme eval "var Cart = angular.element(document.body).injector().get('Cart'); Cart.addLine({product: {id: PRODUCT_ID}, quantity: 1, isCase: false}); JSON.stringify({lines: Object.keys(Cart.lines).length});"
```

4. Wait 3 seconds for the cart to sync to the server. The Cart service auto-saves via `POST /v2/retailers/1219/branches/2585/carts/{cartId}`.

5. **First add-to-cart note:** On the very first add after login, a delivery options dialog may appear (choose "Delivery" or "Self Pickup" with address). If this dialog appears, tell the user: "A delivery options dialog has appeared in the browser. Please select your delivery preference, then I'll continue."

6. Verify the item was added:
```bash
browser-use --session superme eval "var Cart = angular.element(document.body).injector().get('Cart'); JSON.stringify({lines: Object.keys(Cart.lines).length, total: Cart.total.priceForView});"
```

7. Confirm to user: "Added [product name] to your Keshet Teamim cart. [N] items, total: [price]."

---

### `/superme magicorder`

Consolidate recent orders into one list/cart. Behavior depends on active vendor.

---

#### Shufersal magicorder (to wishlist)

**Steps:**

1. Export cookies (if not already done):
```bash
browser-use --session superme cookies export /tmp/superme_cookies.json
```
Build cookie string (same as login step 6).

2. Fetch orders list via API:
```bash
curl -s 'https://www.shufersal.co.il/online/he/my-account/orders' \
  -H 'accept: */*' -H 'content-type: application/json' \
  -H 'x-requested-with: XMLHttpRequest' \
  -b "$(cat /tmp/superme_cookie_str.txt)"
```
Response: `closedOrders[]` array. Take first 5 order codes.

3. Fetch each order's details:
```bash
curl -s "https://www.shufersal.co.il/online/he/my-account/orders/ORDER_CODE" \
  -H 'accept: */*' -H 'content-type: application/json' \
  -H 'x-requested-with: XMLHttpRequest' \
  -b "$(cat /tmp/superme_cookie_str.txt)"
```
Response: `entries[]` with `product.name`, `product.code`, `product.price.formattedValue`.

4. Consolidate products, deduplicate by code. Exclude delivery items ("משלוח", "דמי").

5. Present to user sorted by frequency (most ordered first), English transliteration.

6. Create empty wishlist via `$.ajax` (see "Creating a session wishlist" above). Name: "Superme Magicorder DD-MM-YYYY".

7. Navigate to wishlist edit page, add all products via Vue component (same as `/superme add` step 4, but loop over all products and call `submitWishList()` once at the end).

8. Verify server-side count matches. Close any WAF error dialogs.

9. Give user the link: `https://www.shufersal.co.il/online/he/wish-lists/main`

---

#### Keshet Teamim magicorder (to cart)

**Steps:**

1. Read token and user ID from `/tmp/superme_keshet_token.txt` and `/tmp/superme_keshet_user_id.txt`.

2. Fetch recent orders via browser (the Prutah platform exposes order history through the Angular app):
```bash
browser-use --session superme open "https://www.keshet-teamim.co.il/my-orders"
```

3. Wait for load, then extract order data:
```bash
browser-use --session superme eval "var orders = angular.element(document.body).injector().get('Order'); JSON.stringify(orders.list || orders.orders || []);"
```
If the Order service isn't directly accessible, scrape from the page state. Take the first 5 orders.

4. For each order, extract its product entries. If order details are available via API:
```bash
browser-use --session superme eval "var http = angular.element(document.body).injector().get('\$http'); http.get('/v2/retailers/1219/users/USER_ID/orders/ORDER_ID?appId=4').then(function(r) { window._supermeOrderData = JSON.stringify(r.data); });"
browser-use --session superme eval "window._supermeOrderData;"
```

5. Consolidate products, deduplicate by product ID. Exclude delivery items ("משלוח", "דמי משלוח").

6. Present to user sorted by frequency (most ordered first), English transliteration.

7. Add all products to cart in one batch via Angular:
```bash
browser-use --session superme eval "var Cart = angular.element(document.body).injector().get('Cart'); var products = PRODUCT_ARRAY_JSON; products.forEach(function(p) { Cart.addLine({product: {id: p.id}, quantity: 1, isCase: false}); }); JSON.stringify({lines: Object.keys(Cart.lines).length});"
```

8. Wait 5 seconds for cart sync. Verify count:
```bash
browser-use --session superme eval "var Cart = angular.element(document.body).injector().get('Cart'); JSON.stringify({lines: Object.keys(Cart.lines).length, total: Cart.total.priceForView});"
```

9. Confirm to user: "Added [N] products to your Keshet Teamim cart. Total: [price]. View your cart at https://www.keshet-teamim.co.il/cart"

---

### `/superme lists`

View all Superme wishlists. **Shufersal only.**

**Steps:**

1. Get wishlists via browser API:
```javascript
fetch(ACC.config.encodedContextPath + '/wishlist/getUserWishlists', {
  credentials: 'include',
  headers: {'X-Requested-With': 'XMLHttpRequest'}
}).then(r => r.json())
```

2. Filter for lists whose name starts with "Superme".

3. Present in a table: List Name, Items Count, Date.

4. If no Superme lists found, suggest `/superme search` to start.

5. Link: `https://www.shufersal.co.il/online/he/wish-lists/main`

---

### `/superme cart`

View current cart contents. **Keshet Teamim only.**

**Steps:**

1. Read vendor from `/tmp/superme_vendor.txt`. If not `keshet`, tell user this command is for Keshet Teamim only and suggest `/superme lists` for Shufersal.

2. Get cart contents via the Angular Cart service:
```bash
browser-use --session superme eval "var Cart = angular.element(document.body).injector().get('Cart'); var lines = Cart.lines; var items = Object.keys(lines).map(function(k) { var l = lines[k]; return {id: k, name: l.product.localName, quantity: l.quantity, price: l.product.branch.regularPrice, salePrice: l.product.branch.salePrice}; }); JSON.stringify({items: items, total: Cart.total});"
```

3. Present cart contents using **English only** (transliterate Hebrew names). Columns: #, Product Name, Qty, Unit Price, Line Total.

4. Show cart summary: total items, subtotal, any discounts.

5. Link: `https://www.keshet-teamim.co.il/cart`

---

### `/superme close`

```bash
browser-use --session superme close
```
Also clean up temp files:
```bash
rm -f /tmp/superme_vendor.txt /tmp/superme_last_search.json /tmp/superme_session_list_id.txt /tmp/superme_cookies.json /tmp/superme_cookie_str.txt /tmp/superme_keshet_token.txt /tmp/superme_keshet_user_id.txt /tmp/superme_keshet_cart_id.txt
```

---

## Key Technical Details

### Shufersal

#### API Endpoints (require session cookies)

| Endpoint | Method | Notes |
|----------|--------|-------|
| `/online/he/my-account/orders` | GET | List all orders. Returns `{closedOrders: [...]}` |
| `/online/he/my-account/orders/CODE` | GET | Order details. Returns `{entries: [...]}` |
| `/wishlist/getUserWishlists` | GET | All user wishlists. Via browser fetch only. |
| `/wishlist/createWishlist` | POST | Create empty list. Body: `{name: "..."}`. Via `$.ajax`. |
| `/wishlist/editWishlist` | POST | Edit list (add/remove products). Via Vue store dispatch. |
| `/wishlist/getWishlist?listId=ID` | GET | Get wishlist details with entries. |

#### Cookie Management

- **For curl API calls** (orders): Export cookies via `browser-use cookies export`, build cookie string including httpOnly cookies (JSESSIONID, XSRF-TOKEN).
- **For browser-side API calls** (wishlists): Use `$.ajax` or `fetch` from browser eval — cookies are automatically included.
- Cookies from `document.cookie` are incomplete (missing httpOnly). Always use `cookies export`.

#### Vue Component Access

The wishlist edit page uses Vue.js. Access the component tree:
```javascript
var comp = document.querySelector('#personalAreaContainer').__vue__;
function findComp(vm) {
  if(vm.submitWishList) return vm;
  for(var c of (vm.$children || [])) { var f = findComp(c); if(f) return f; }
  return null;
}
var wl = findComp(comp);
```

Key methods:
- `wl.createNewProductSearch(product, qty, sellingMethod)` — add product to local state
- `wl.submitWishList()` — save all changes to server
- `wl.changesArr` / `wl.productArr` / `wl.entries` — state arrays

---

### Keshet Teamim

#### Platform

Keshet Teamim runs on the **Prutah** platform (white-label Israeli grocery e-commerce). AngularJS 1.8.0, app module `ZuZ`.

**Key constants:**
- Retailer ID: `1219`
- Default Branch ID: `2585`
- App ID: `4`
- Language ID: `1` (Hebrew)

#### API Endpoints

Base URL: `https://www.keshet-teamim.co.il/v2`

| Endpoint | Method | Auth | Notes |
|----------|--------|------|-------|
| `/retailers/1219/branches/2585/products` | GET | Bearer token | Search products. Params: `query`, `size`, `from`, `isSearch`, `languageId`, `filters` |
| `/retailers/1219/branches/2585/products/autocomplete` | GET | Bearer token | Search autocomplete. Params: `query`, `size` |
| `/retailers/1219/branches/2585/carts/{cartId}` | POST | Bearer token | Create/update cart |
| `/retailers/1219/branches/2585/specials` | GET | Bearer token | Active promotions |
| `/retailers/1219/users/{userId}/coupons` | GET | Bearer token | User coupons |

#### Standard Search Filters

Always include this `filters` parameter (URL-encoded) when searching:
```json
{"must":{"exists":["family.id","family.categoriesPaths.id","branch.regularPrice"],"term":{"branch.isActive":true,"branch.isVisible":true}},"mustNot":{"term":{"branch.regularPrice":0}}}
```

#### Angular Service Access

Access Angular services from browser eval:
```javascript
var injector = angular.element(document.body).injector();
var Cart = injector.get('Cart');
var User = injector.get('User');
```

Key Cart methods:
- `Cart.addLine({product, quantity, isCase})` — add product to cart
- `Cart.removeLine(productId)` — remove from cart
- `Cart.lines` — object map of cart lines keyed by product ID
- `Cart.total` — total with `priceForView`, `lines` (count)
- `Cart.serverCartId` — server-side cart ID

Key User properties:
- `User.session.token` — Bearer token for API calls
- `User.session.userId` — user ID
- `User.isVerified()` — check if logged in

#### Product Data Structure

From search API responses, each product contains:
- `id` — unique product ID (use for cart operations)
- `localName` — Hebrew product name
- `brand.names.1` — Hebrew brand name
- `branch.regularPrice` / `branch.salePrice` — pricing
- `weight`, `unitOfMeasure`, `isWeighable` — size/weight info
- `family.name` — product category
- `image` — product image URL

---

## Known Issues

### Shufersal
- **WAF blocks special characters**: Product names with `'` (e.g., `תפוצ'יפס`) trigger "תווים לא חוקיים" (426) on the auto-save retry. The initial save usually succeeds. Close the error dialog and verify server-side.
- **List names**: Cannot contain `/`. Use hyphens for dates: DD-MM-YYYY.
- **Hebrew in CLI args**: `browser-use type` doesn't handle Hebrew. Use `eval` with `document.getElementById().value = '...'` instead.
- **Cart doesn't persist**: Items added to cart via browser automation are lost when the session closes. Always use wishlists.

### Keshet Teamim
- **SMS OTP login**: Cannot be fully automated — user must provide the SMS code.
- **Delivery dialog on first add**: A delivery options dialog appears the first time you add to cart. User must handle this manually in the browser.
- **Branch-specific pricing**: Prices and availability vary by branch. Default branch ID is `2585`. If the user needs a different branch, the branch ID must be updated in API calls.
- **Cart is session-bound**: Cart contents persist server-side within the session but may reset after extended inactivity.

---

## Session Management

- All commands use `--session superme` for persistent browser session
- Always use `--headed` on initial `open` so user can see the browser
- Login state persists within the session but is lost on `close`
- For Keshet Teamim, the Bearer token is saved separately and can be used for API calls even if the browser session is interrupted

## Supported Vendors

| Vendor | Domain | Platform | Login | Status |
|--------|--------|----------|-------|--------|
| Shufersal | shufersal.co.il | Custom (Vue.js) | Email + password | **Supported** |
| Keshet Teamim | keshet-teamim.co.il | Prutah (AngularJS) | Phone + SMS OTP | **Supported** |
| Rami Levy | rami-levy.co.il | — | — | Planned |
| Yochananof | yochananof.co.il | — | — | Planned |
| Victory | victory-online.co.il | — | — | Planned |

## Error Handling

- Login fails (Shufersal) → check credentials, try again
- Login fails (Keshet) → verify phone number, request new SMS code
- OTP expired → run `/superme login keshet` again to get a new code
- Session lost → run `/superme login` again
- WAF error on save (Shufersal) → close dialog, verify server-side count
- 500 on create wishlist (Shufersal) → check name for special characters
- Delivery dialog blocks cart add (Keshet) → user must complete delivery setup in browser
- browser-use not installed → `uv tool install browser-use --python 3.12`
