---
name: shopee-open-platform-integration
description: Build a Shopee Open Platform "Partner App" that lets ONE developer account onboard MANY seller shops via OAuth, with HMAC-SHA256 request signing, 4-hour access token + 30-day refresh token lifecycle, and a minimum-scope strategy that shortens production review. Use when an agency / SaaS platform needs legitimate API access to fetch products, manage orders, upload images, or sync inventory from multiple Shopee shops without scraping the public storefront (which is anti-bot protected). Covers ID/MY/SG/TH/PH/VN/BR/TW marketplaces — one app can serve all regions.
---

# Shopee Open Platform — Partner App Integration

> Production playbook. One Tiranyx-style developer account → unlimited client shops via OAuth. Replaces the storefront-scraping hacks (see `shopee-data-import` skill) with legitimate, supported API access.

## When to use this vs `shopee-data-import`

| Situation | Skill |
|---|---|
| Client grants you Seller Center login OR sends CSV | `shopee-data-import` (Path A) |
| You're an agency wanting **scalable, repeatable** integration across many clients | **THIS skill** |
| One-off scrape, no budget, no client cooperation | `shopee-data-import` (Path E manual) |

The Open Platform path takes **1-2 weeks one-time review**, then onboarding each new client is **2-click OAuth**.

## Account types — pick ONE

| Type | Use when | OAuth model |
|---|---|---|
| **Shop App** | Building a tool for YOUR OWN single Shopee shop | Self-only token |
| **Partner App** | Building agency/SaaS for many sellers | Multi-tenant OAuth |

For agency / biolink / catalog-management → **Partner App**.

## Account registration (one-time, ~30 min)

1. Go to <https://open.shopee.com> → **Sign Up** (use PT/company email)
2. Pick region — **Indonesia** if primary, but you can add others later from same account
3. Submit company docs:
   - Business name (e.g., PT Tiranyx Digitalis Nusantara)
   - NIB / NPWP
   - Brief description of integrator role
4. Wait approval (1-3 business days) → dashboard access
5. **Create your first app**:
   - App name: e.g. "TWKCard Biolink Platform"
   - App type: **Partner**
   - Region: ID (add more later if needed)
   - Description (used during review — see strategy below)
   - Logo (square ≥ 512px)

You get:
- `partner_id` (numeric, e.g. `2001234`)
- `partner_key` (HMAC signing secret — keep server-side, NEVER ship to browser)
- Sandbox credentials (separate from production)

## Architecture: 1 dev account → ∞ shops

```
Your Backend (e.g. PT Tiranyx)
├─ 1 Shopee Open Platform Developer Account
└─ 1 "Partner App" with partner_id + partner_key
   │
   ├─ OAuth scope: product.info_read, shop.info_read, image.upload
   │
   ├─ Client A authorizes → access_token_A + shop_id_A + refresh_token_A
   ├─ Client B authorizes → access_token_B + shop_id_B + refresh_token_B
   ├─ Client C authorizes → access_token_C + shop_id_C + refresh_token_C
   │
   └─ Per-shop API calls: signed with (partner_id, partner_key, shop_id, access_token)
```

Storage: a `ShopeeOAuthToken` table keyed by `(partnerId, shopId)` storing `accessToken`, `refreshToken`, `accessTokenExpiresAt`, `refreshTokenExpiresAt`, `region`. Refresh proactively when access token has < 30 min left.

## OAuth flow (the dance)

Shopee uses a non-standard OAuth-ish flow with HMAC-SHA256 signing. Pseudo-code:

### Step 1 — Generate authorization URL

```ts
const path = '/api/v2/shop/auth_partner';
const timestamp = Math.floor(Date.now() / 1000);
const baseString = `${partnerId}${path}${timestamp}`;
const sign = crypto.createHmac('sha256', partnerKey).update(baseString).digest('hex');
const redirect = encodeURIComponent('https://dev.yourapp.com/api/shopee/oauth/callback');
const authUrl = `https://partner.shopeemobile.com${path}?partner_id=${partnerId}&timestamp=${timestamp}&sign=${sign}&redirect=${redirect}`;
// Send user to authUrl. They login Shopee, click Authorize.
```

Sandbox host: `partner.test-stable.shopeemobile.com`
Production host: `partner.shopeemobile.com` (ID/MY/PH/SG/TW/TH/VN/BR — same hostname).

### Step 2 — Callback receives `code` + `shop_id`

```ts
// GET /api/shopee/oauth/callback?code=XXX&shop_id=123&main_account_id=456
```

### Step 3 — Exchange code for token

```ts
const path = '/api/v2/auth/token/get';
const timestamp = Math.floor(Date.now() / 1000);
const baseString = `${partnerId}${path}${timestamp}`;
const sign = crypto.createHmac('sha256', partnerKey).update(baseString).digest('hex');

const res = await fetch(`https://partner.shopeemobile.com${path}?partner_id=${partnerId}&timestamp=${timestamp}&sign=${sign}`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ code, shop_id: shopId, partner_id: partnerId }),
});
const { access_token, refresh_token, expire_in } = await res.json();
// expire_in = 14400 seconds (4 hours)
// refresh_token TTL = 30 days
```

Store `access_token`, `refresh_token`, `expiresAt = now + expire_in*1000`, `shop_id` in your DB.

### Step 4 — Signed API calls

Every subsequent call signs with the access_token included:

```ts
const path = '/api/v2/product/get_item_list';
const timestamp = Math.floor(Date.now() / 1000);
const baseString = `${partnerId}${path}${timestamp}${accessToken}${shopId}`;
const sign = crypto.createHmac('sha256', partnerKey).update(baseString).digest('hex');

const url = `https://partner.shopeemobile.com${path}` +
  `?partner_id=${partnerId}&timestamp=${timestamp}&access_token=${accessToken}` +
  `&shop_id=${shopId}&sign=${sign}` +
  `&offset=0&page_size=50&item_status=NORMAL`;

const res = await fetch(url);
const { response } = await res.json();
// response.item has [{item_id, item_status}, ...]
```

### Step 5 — Refresh access token (before expiry)

```ts
const path = '/api/v2/auth/access_token/get';
const baseString = `${partnerId}${path}${timestamp}`;
const sign = crypto.createHmac('sha256', partnerKey).update(baseString).digest('hex');
const res = await fetch(`https://partner.shopeemobile.com${path}?...`, {
  method: 'POST',
  body: JSON.stringify({
    partner_id: partnerId, shop_id: shopId, refresh_token: refreshToken,
  }),
});
const { access_token, refresh_token } = await res.json();
// Replace both tokens — refresh tokens ARE rotated.
```

## Minimum scope strategy (faster review)

Don't request scopes you don't use yet. Shopee review checks for "scope necessity." Start with this minimal set and add as features land:

| Scope | When to request | Use case |
|---|---|---|
| `product.info_read` | Always | List + detail products → seed your catalog |
| `shop.info_read` | Always | Verify shop identity on OAuth callback |
| `image.upload` | When uploading photos | Replace local placeholder with Shopee CDN URL |
| `order.info_read` | Phase 2 — Purchase event attribution | CAPI Purchase events with real order value |
| `logistics.info_read` | Phase 3 — Order tracking widget | Auto-update buyer with shipment status |
| `product.info_update` | Phase 4 — Two-way sync | Bidirectional product edit. **Avoid unless feature ready** — review scrutiny is much higher. |

## Review submission strategy (1-2 weeks → cut to 1 week)

Shopee reviewers look for:

1. **App description with concrete use case** — not "management tools" but:
   > "TWKCard is a biolink + flip-card SaaS platform serving Indonesian Shopee Official Mall sellers. We help sellers showcase their Shopee catalog on a branded card linked from Instagram bio + TikTok. Our use of the Open Platform is limited to fetching product details (`product.info_read`) and shop information (`shop.info_read`) to display the seller's catalog accurately. Marketing pixel events fire when buyers click through to Shopee, helping sellers measure ad ROI."

2. **Working sandbox demo** — record 60-90s screen video:
   - Open OAuth link → login to Shopee sandbox account → authorize
   - Show product list rendered from API in your UI
   - One click-through to Shopee from your card → tracking pixel fires
   - Upload it to YouTube unlisted or Loom, paste link in submission

3. **Privacy policy** — must reference Shopee data:
   > "Data fetched via Shopee Open Platform (product details, shop info) is stored only in our application database, used only for displaying the merchant's catalog on their TWKCard biolink page, and never sold or shared with third parties. Sellers can revoke access at any time via Shopee Seller Center → Authorized Apps."

4. **Logo + screenshots** — square logo + 2-3 screenshots of the integration in action

5. **Error handling demo** — show what happens when token expires (refresh flow), when API returns 4xx, when seller revokes access

6. **Scope justification** — for each requested scope, write 1 sentence why it's needed. Don't request `order.info_read` unless you're ready to demo the Purchase pixel fire.

### Common review rejections (avoid these)

- ❌ Requesting `product.info_update` without showing the edit UI exists → "scope unnecessary"
- ❌ Vague description: "platform for sellers" → "what platform doing what?"
- ❌ No working sandbox demo → "cannot verify integration"
- ❌ Production callback URL doesn't HTTPS (Shopee rejects HTTP)
- ❌ Logo low-res or generic placeholder

## Common pitfalls (after going production)

| Pitfall | Fix |
|---|---|
| Rate limit 429 | Default 1000 req/min per shop. Cache aggressively. Use webhooks for order updates instead of polling. |
| `error_auth` mid-session | Access token expired but you didn't refresh. Add proactive refresh when `expiresAt - now < 30 min`. |
| Refresh token expired (30 days inactive) | Force re-authorization. Email seller "please reconnect Shopee in TWKCard." |
| Image upload 1MB limit | Resize client-side before upload. Shopee compresses to webp anyway. |
| Timestamp drift > 5 min | NTP sync your server. Shopee rejects signed requests with drifted timestamps. |
| Sandbox data ≠ production behavior | Shopee sandbox has limited product data, slower. Always do final QA against real shop test mode. |

## Webhooks (push instead of poll — Phase 2)

Once stable, register webhook URLs for:
- `order_status_update` → fire CAPI Purchase event on `READY_TO_SHIP`
- `item_promotion_update` → re-sync product when seller updates Shopee price
- `shop_authorization_partner_status` → handle seller revoking your app

Webhook signature: `Authorization: <hex>` header is HMAC-SHA256 of `(url + body)` using `partner_key`.

## Multi-region

Same `partner_id` works across regions. To add a new region (e.g. expand from ID to MY):
1. Open Platform → App → Region Settings → Add Malaysia
2. Re-submit for MY review (faster — same docs, just region change)
3. New OAuth authorize URL: same hostname, but seller must login MY Shopee account

Storage: tag each token row with `region` field so you call the right host (production hostname is same, but rate limits + data are isolated per region).

## References

- Open Platform: <https://open.shopee.com>
- Docs: <https://open.shopee.com/documents>
- OAuth flow: <https://open.shopee.com/documents/v2/OpenAPI%202.0%20Overview>
- API reference: <https://open.shopee.com/documents/v2/api-reference>
- Sandbox: <https://partner.test-stable.shopeemobile.com>
- Sample apps: <https://github.com/shopee-open-platform>
