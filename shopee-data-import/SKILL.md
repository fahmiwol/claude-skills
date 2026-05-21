---
name: shopee-data-import
description: Pragmatic decision tree for importing product data from a Shopee Indonesia (or any region) storefront into your own database, when you need product names, prices, images, descriptions for catalog mirroring, ad campaigns, or Meta Commerce. Documents what works, what fails, and gives ready-to-run code for the practical paths. Use when client/PM asks "fetch products from shopee.co.id/{shop}" and you need to choose between scraping, API, CSV export, or manual entry.
---

# Shopee Storefront → Your DB

> Brutal honesty: Shopee's anti-bot is among the most sophisticated in SEA e-commerce. This skill documents the **realistic** paths, ranked by reliability.

## Decision tree (read first)

```
Can you get seller-center credentials from the brand?
├─ YES → Path A: Seller Center CSV Export (5 min, 100% data)
└─ NO
   ├─ Brand has CPAS feed enabled? → Path B: Meta Catalog API (need catalog scope token)
   ├─ Budget $0.01/product OK?     → Path C: Apify xtracto/shopee-scraper actor
   ├─ Skilled engineer + time?     → Path D: Reverse-engineer af-ac-enc-dat (NOT recommended)
   └─ Need fast skeleton with real names+prices?
                                   → Path E: Manual seed from "Terlaris" screenshot (15 min, 10 products)
```

## What DOES NOT work (don't waste time)

❌ **Raw HTTP to `shopee.co.id/api/v4/search/search_items`**
- Returns `{"error": 90309999, "items": []}` even with full browser headers + cookies (SPC_F, csrftoken)
- Missing `af-ac-enc-dat` + `x-sap-sec` headers that are computed by obfuscated runtime JS

❌ **Bare Playwright headless**
- Page loads, 12 API calls fire, but `search_items` is **silently suppressed** by anti-bot
- Browser fingerprint detected via JS challenge before product feed fetches
- Title becomes generic "Shopee Indonesia" instead of shop name

❌ **Playwright + `navigator.webdriver` undefined + AutomationControlled disabled**
- Still fails. Detection is deeper (canvas fingerprint, TLS JA3, audio context).

❌ **Cookies from Playwright session reused in raw fetch**
- Cookies obtained but API still 403. Tokens are session+IP+UA bound and expire fast.

## What WORKS

### Path A: Seller Center CSV Export ⭐ Recommended

Ask client to:
1. Login `seller.shopee.co.id`
2. Produk Saya → Manajemen Produk → tombol **Ekspor**
3. Pilih "Semua Produk" → format Excel/CSV
4. Send file ke kita

CSV fields: `Kode Produk`, `Nama Produk`, `Deskripsi Produk`, `Variation Name 1/2`, `Harga`, `Stok`, `URL Gambar Utama`, `URL Gambar 1-8`, `Kategori`, `Berat`, etc. — **full data, no scraping needed**.

Then parse:
```ts
import { PrismaClient } from '@prisma/client';
import { parse } from 'csv-parse/sync';

const prisma = new PrismaClient();
const csv = await fs.readFile('aloha-products.csv', 'utf8');
const rows = parse(csv, { columns: true, skip_empty_lines: true });

for (const r of rows) {
  await prisma.product.upsert({
    where: { id: `shopee_${r['Kode Produk']}` },
    create: {
      id: `shopee_${r['Kode Produk']}`,
      brandId,
      name: r['Nama Produk'],
      description: r['Deskripsi Produk']?.slice(0, 500),
      imageUrl: r['URL Gambar Utama'],
      priceIdr: parseInt(r['Harga'], 10),
      shopeeUrl: `https://shopee.co.id/product/${r['Shop ID']}/${r['Kode Produk']}`,
      active: r['Status'] === 'Aktif',
      metadata: { sku: r['SKU Induk'], stock: parseInt(r['Stok'], 10), category: r['Kategori'] },
    },
    update: { /* same minus id */ },
  });
}
```

### Path B: Meta Catalog API (if CPAS active)

Aloha-style **Official Shop** dengan CPAS punya catalog yang sudah sync ke Meta. Generate token dengan `catalog_management` scope:

1. Business Settings → Users → **System Users** → Create
2. Tambahkan assets: pilih Catalog ID (terlihat di Commerce Manager URL: `commerce/{catalog_id}/...`)
3. Permission: `catalog_management` (full)
4. Generate Token (never expires kalau System User)

```bash
TOKEN="EAA..."
CATALOG_ID="4431641893749138"
curl -s "https://graph.facebook.com/v21.0/$CATALOG_ID/products?fields=id,name,description,price,image_url,url,brand,category,availability&limit=100&access_token=$TOKEN"
```

Returns Aloha catalog yang sudah Shopee push. Convert ke product schema mirip Path A.

### Path C: Apify xtracto/shopee-scraper

[apify.com/xtracto/shopee-scraper](https://apify.com/xtracto/shopee-scraper) — $10 per 1,000 products.

```bash
curl -X POST "https://api.apify.com/v2/acts/xtracto~shopee-scraper/run-sync-get-dataset-items?token=$APIFY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "country": "ID",
    "mode": "shop",
    "shopId": "1382911900",
    "sort": "sales",
    "maxProducts": 30,
    "fetchDetail": true
  }'
```

Output: array dengan title, price, image, description, models, attributes, ratings.

### Path E: Manual seed from screenshot (best fallback)

Kalau client tidak kasih akses dan budget tidak ada untuk Apify, scrape **top 10 by visual inspection**. User send screenshot Terlaris tab, kita extract:

```cjs
const PRODUCTS = [
  { kind: 'Sprei King',         price: 127000, soldMonthly: 56, discount: 68 },
  { kind: 'Bed Cover Set King', price: 344000, soldMonthly: 28, discount: 65 },
  // ... 8 more from screenshot
];
// upsert ke Prisma dengan placeholder image, deep-link ke storefront category filter
```

URL fallback per produk: `https://shopee.co.id/{shop_username}?categoryName={category_name_url_encoded}`

Image: leave `null`, user upload via editor. Atau pakai placeholder logo brand.

## Anti-bot bypass techniques (for the curious)

Kalau ada engineering time untuk reverse-engineer:

1. **af-ac-enc-dat header** — encrypted blob computed by Shopee's WhiteList JS:
   - Use Playwright untuk capture network request setelah scroll → extract header value
   - Replay tidak bisa: token includes IP + UA + timestamp + signature
   - Some open-source attempts exist tapi all stale within weeks

2. **TLS JA3 fingerprint** — Playwright headless TLS handshake berbeda dari real Chrome
   - Bypass: pakai [curl-impersonate](https://github.com/lwthiker/curl-impersonate) yang spoof Chrome TLS
   - Limited: only works for raw HTTP, not browser automation

3. **Residential proxy + datacenter rotation**
   - BrightData / Oxylabs / Smartproxy: ~$15-50/GB
   - May still hit JS challenges, just from clean IPs

4. **Playwright-extra + stealth plugin**
   - GitHub: [berstend/puppeteer-extra-plugin-stealth](https://github.com/berstend/puppeteer-extra/tree/master/packages/puppeteer-extra-plugin-stealth)
   - Works for many sites; Shopee still detects through canvas + audio fingerprint
   - Diminishing returns vs maintenance burden

**Bottom line**: unless you're building a scraper-as-a-service, Path A/B/E.

## Shopee item URL format (for reference)

```
https://shopee.co.id/{slug}-i.{shopid}.{itemid}
  slug    = SEO-friendly URL-encoded product name (lowercase, hyphens)
  shopid  = numeric, e.g. 1382911900
  itemid  = numeric, unique per product
```

Affiliate redirect (if CPAS active):
```
https://shp.ee/{short_token}   ← obfuscated affiliate link
https://shope.ee/{short_token} ← deeplink to app
```

Price format in API responses:
```
price_min = price * 100000   (5 decimal places fixed-point)
Rp 127.000 ↔ 12700000000
```

Image URL format:
```
https://down-id.img.susercontent.com/file/{hash}              ← full res
https://cf.shopee.co.id/file/{hash}_tn                        ← thumbnail
https://cf.shopee.co.id/file/sg-11134201-7rai0-m{hash}@resize_w400_nl ← resized
```

## Lessons learned (TWKCard project)

- **Don't promise client "we'll scrape Shopee automatically"** — anti-bot stops 95% of attempts
- **Always ask for seller credentials or CSV export first** — even if client says "ribet"
- **Manual seed from screenshot is faster than 4 hours of bypass attempts**
- **Image hosting matters**: Shopee CDN URLs are publicly hotlinkable but can break (cache invalidation). For production, download + re-host on your CDN (Cloudflare R2 / Bunny / S3).
