---
name: facebook-domain-verification
description: Verify a domain in Facebook Business Manager using BOTH the HTML file method AND the meta-tag method simultaneously, for maximum reliability. Includes Next.js middleware exemption to ensure verification files at /public/ root are served verbatim (not rewritten under subdomain). Use when adding a new domain to Meta Business Settings → Brand Safety → Domains, or fixing a "Not Verified" status caused by middleware swallowing the verification file.
---

# Facebook Business Manager — Domain Verification

> Pattern dari TWKCard. Dual-method untuk redundancy + middleware fix.

## Kapan dipakai

- Domain baru di-add ke Meta Business Settings → status "Not Verified"
- Punya 1 dari 4 verification methods: DNS TXT, HTML file, meta tag, Domain provider
- App pakai Next.js dengan middleware (kemungkinan rewrite path → file 404)

## 2 Method paling reliable

### A. Meta tag (paling sederhana untuk SSR app)

Di `app/layout.tsx` (root layout):

```ts
export const metadata: Metadata = {
  title: 'Your App',
  other: {
    'facebook-domain-verification': 'YOUR_TOKEN_HERE',
  },
};
```

Next.js akan emit `<meta name="facebook-domain-verification" content="YOUR_TOKEN_HERE"/>` di `<head>` setiap halaman.

Verify:
```bash
curl -s https://yourdomain.com/ | grep -oE '<meta name="facebook-domain-verification"[^>]+>'
# → <meta name="facebook-domain-verification" content="YOUR_TOKEN_HERE"/>
```

### B. HTML file di root

```bash
# Save token sebagai filename + content
echo -n "YOUR_TOKEN_HERE" > apps/web/public/YOUR_TOKEN_HERE.html
```

Verify:
```bash
curl -s https://yourdomain.com/YOUR_TOKEN_HERE.html
# → YOUR_TOKEN_HERE
```

## ⚠️ Critical: Middleware exemption

Kalau Next.js app pakai middleware untuk subdomain rewriting (atau pattern lain), **DEFAULT middleware akan rewrite `/YOUR_TOKEN.html` → `/[brand]/YOUR_TOKEN.html`** dan render 404 instead.

Fix di `middleware.ts`:

```ts
export function middleware(req: NextRequest) {
  const url = req.nextUrl;

  // Add this BEFORE any rewrite logic
  if (
    url.pathname.startsWith('/api') ||
    url.pathname.startsWith('/_next') ||
    url.pathname === '/favicon.ico' ||
    url.pathname === '/robots.txt' ||
    url.pathname === '/sitemap.xml' ||
    // ✅ Site verification files (FB, Google Search Console, Bing)
    /\.(html|txt|xml|png|jpg|jpeg|svg|webp|ico)$/i.test(url.pathname)
  ) {
    return NextResponse.next();
  }

  // ... rest of middleware
}
```

## Workflow lengkap

1. **Get token dari Meta**:
   - Business Settings → Brand Safety → Domains → Add → enter your domain
   - Pilih method "Tambahkan meta-tag" → copy token (string seperti `za12dlnm8iz889xup6ih82iw7pveay`)

2. **Tambahkan di code (kedua method)**:
   ```bash
   # File method
   echo -n "za12dlnm8iz889xup6ih82iw7pveay" > apps/web/public/za12dlnm8iz889xup6ih82iw7pveay.html

   # Meta tag method — edit layout.tsx (see above)

   # Middleware exemption (kalau ada middleware)
   # Edit middleware.ts add the regex check
   ```

3. **Deploy** → wait propagation 1-3 menit

4. **Verify locally**:
   ```bash
   curl -s https://yourdomain.com/za12dlnm8iz889xup6ih82iw7pveay.html
   # Must return: za12dlnm8iz889xup6ih82iw7pveay
   curl -s https://yourdomain.com/ | grep facebook-domain-verification
   # Must show the meta tag
   ```

5. **Klik "Verifikasi Domain" di Meta UI** → status berubah jadi ✅ Verified
   - Kalau gagal: Meta cache, retry dalam 5-15 menit
   - Maks 72 jam untuk propagasi

## Bonus: covers juga

Regex `/\.(html|txt|xml|png|jpg|jpeg|svg|webp|ico)$/i` ini juga handle:
- Google Search Console: `google<token>.html`
- Bing Webmaster Tools: `BingSiteAuth.xml`
- Apple App Site Association: `apple-app-site-association` (no ext — add separately)
- Open Graph image fallbacks: `og-*.png`
- ads.txt / app-ads.txt (add `.txt` already included)

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| HTML file returns 404 page | Middleware exemption missing. Check regex. |
| Meta tag missing dari HTML | Make sure `metadata` is in root `layout.tsx`, not page-level. |
| Verified tapi balik jadi "Not Verified" | Domain juga di-claim BM lain. Check ownership conflicts. |
| Subdomain verified tapi apex tidak (atau sebaliknya) | Meta verifies EXACT domain. `acme.com` ≠ `www.acme.com` ≠ `sub.acme.com`. |
| Need to verify wildcard subdomain | Tidak bisa wildcard. Tiap subdomain harus diverifikasi terpisah ATAU verify apex domain (which covers all subdomains for ads attribution). |

## Quick test command

```bash
DOMAIN=yourdomain.com
TOKEN=za12dlnm8iz889xup6ih82iw7pveay

echo "=== HTML file ===" && curl -s https://$DOMAIN/$TOKEN.html
echo "=== Meta tag ===" && curl -s https://$DOMAIN/ | grep -oE '<meta name="facebook-domain-verification"[^>]+>'
```

Expected output: token verbatim + meta tag in head.
