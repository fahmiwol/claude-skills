---
name: nextjs-subdomain-multitenant
description: Set up subdomain → brand/tenant rewriting in Next.js 15 App Router using middleware, with public file exemptions for site verification (FB/Google/Bing), and proper /api + /_next pass-through. Use when building a multi-tenant SaaS where each customer gets their own subdomain (acme.app.com, beta.app.com, etc.) on a single Next.js deployment.
---

# Next.js Subdomain Multi-Tenant Pattern

> Battle-tested from TWKCard. Handles brand subdomain routing + site verification files + dev-mode localhost fallback.

## Pola URL

```
acme.app.com/        → [brand]/page.tsx       dengan brand=acme
acme.app.com/home    → [brand]/home/page.tsx
acme.app.com/store   → [brand]/store/page.tsx
acme.app.com/cms     → [brand]/cms/page.tsx
acme.app.com/api/*   → app/api/* (apex API)   NO rewrite
app.com/             → apex landing           NO rewrite
www.app.com/         → apex landing           NO rewrite
localhost:3000/      → fallback ke DEFAULT_BRAND (dev convenience)
acme.localhost:3000/ → [brand] dengan brand=acme (dev with subdomain)
```

## Struktur folder

```
apps/web/
  middleware.ts                   # the rewriter
  app/
    page.tsx                      # apex landing (only seen on app.com root)
    [brand]/
      layout.tsx                  # per-brand <head>, pixels, theme CSS vars
      page.tsx                    # brand home
      home/page.tsx               # /home variant (e.g. IG bio link)
      store/page.tsx              # /store variant (e.g. catalog)
      cms/page.tsx                # editor (gated)
    api/                          # all routes here STAY at apex
  public/                         # static files served verbatim
```

## middleware.ts

```ts
import { NextResponse, type NextRequest } from 'next/server';

const DEFAULT_BRAND = process.env.NEXT_PUBLIC_DEFAULT_BRAND ?? 'aloha';
const ROOT_DOMAIN = (process.env.NEXT_PUBLIC_ROOT_DOMAIN ?? 'app.com').toLowerCase();

export function middleware(req: NextRequest) {
  const host = (req.headers.get('host') ?? '').toLowerCase();
  const url = req.nextUrl;

  const requestHeaders = new Headers(req.headers);
  requestHeaders.set('x-twk-pathname', url.pathname);

  // Never rewrite these — apex-level + static + verification files.
  // The regex catches .html/.txt/.xml/image files for site verification
  // (Facebook, Google Search Console, Bing) that MUST serve verbatim.
  if (
    url.pathname.startsWith('/api') ||
    url.pathname.startsWith('/_next') ||
    url.pathname === '/favicon.ico' ||
    url.pathname === '/robots.txt' ||
    url.pathname === '/sitemap.xml' ||
    /\.(html|txt|xml|png|jpg|jpeg|svg|webp|ico)$/i.test(url.pathname)
  ) {
    return NextResponse.next({ request: { headers: requestHeaders } });
  }

  const hostNoPort = host.split(':')[0] ?? '';
  const isLocalhost = hostNoPort === 'localhost' || hostNoPort === '127.0.0.1';
  const isApexProd = hostNoPort === ROOT_DOMAIN || hostNoPort === `www.${ROOT_DOMAIN}`;

  // Apex production domain → serve apex pages (landing)
  if (isApexProd) {
    return NextResponse.next({ request: { headers: requestHeaders } });
  }

  // Extract subdomain
  let subdomain: string | null = null;
  if (isLocalhost) {
    const dotIndex = host.indexOf('.');
    if (dotIndex > 0) subdomain = host.slice(0, dotIndex);
  } else if (hostNoPort.endsWith(`.${ROOT_DOMAIN}`)) {
    subdomain = hostNoPort.slice(0, -ROOT_DOMAIN.length - 1);
  }

  const brand = subdomain || DEFAULT_BRAND;
  if (!brand) return NextResponse.next({ request: { headers: requestHeaders } });

  // Rewrite: /xyz → /[brand]/xyz
  url.pathname = `/${brand}${url.pathname === '/' ? '' : url.pathname}`;
  return NextResponse.rewrite(url, { request: { headers: requestHeaders } });
}

export const config = {
  // Match everything except internal next assets
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
};
```

## Why `x-twk-pathname` header?

In layouts we may want to know "what URL did the user actually request?" The rewrite changes `url.pathname` internally. We pass the ORIGINAL pathname via custom header so layouts can branch (e.g. show CMS chrome only when `/cms`).

```ts
// In layout.tsx
const h = await headers();
const pathname = h.get('x-twk-pathname') ?? '';
const isCms = pathname.startsWith('/cms');
return <>{!isCms && <Pixels />}{children}{!isCms && <ConsentBanner />}</>;
```

## DNS setup

For production multi-tenant on Cloudflare / your DNS:

```
A      app.com           → server_ip
A      *.app.com         → server_ip      (wildcard)
A      www.app.com       → server_ip
```

For development with subdomains: edit `C:\Windows\System32\drivers\etc\hosts` (or `/etc/hosts`):
```
127.0.0.1   acme.localhost
127.0.0.1   beta.localhost
```

## TLS for wildcard

Use Let's Encrypt DNS-01 challenge with Cloudflare DNS:
```bash
certbot certonly --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  -d app.com -d '*.app.com'
```

Auto-renew via cron / systemd timer.

## Brand resolution

```ts
// apps/web/lib/brand.ts
import { db } from '@your/db';
import { unstable_cache } from 'next/cache';

export const getBrandBySubdomain = unstable_cache(
  async (subdomain: string) => {
    return db.brand.findUnique({ where: { subdomain } });
  },
  ['brand-by-subdomain'],
  { revalidate: 60, tags: ['brand'] }
);
```

Invalidate on edit: `revalidateTag('brand')` di Server Action saat update.

## Gotchas

| Gotcha | Fix |
|--------|-----|
| FB domain verification HTML returns 404 | Middleware rewrote `/<token>.html` → `/[brand]/<token>.html`. Add `.html` to exempt regex. |
| Static images broken under subdomain | `/_next/image` & `/_next/static` already exempt — good. But favicon/icons should also bypass. |
| Localhost dev needs subdomain | Use `acme.localhost:3000` (modern browsers route this to 127.0.0.1 without hosts edit). |
| API route protected by brand check | Look up brand from request `?b=` param OR `Host` header inside the route handler. |
| Cookies set on subdomain vs apex confusion | If you need cross-subdomain session: `Domain=.app.com` on Set-Cookie. Subdomain-only: omit Domain. |
| Build-time vs runtime brand list | Brands typically in DB. Avoid `generateStaticParams` listing all brands (won't scale). Use `dynamic = 'force-dynamic'` or per-request cache. |

## Verify it's working

```bash
# Apex
curl -s https://app.com/ | grep -oE '<title>[^<]+</title>'
# → Apex landing

# Subdomain
curl -s https://acme.app.com/ | grep -oE 'data-brand="[^"]+"'
# → data-brand="acme"

# API still apex-level
curl -s https://acme.app.com/api/health
# → same response as app.com/api/health

# Verification file
curl -s https://acme.app.com/<some-fb-token>.html
# → file content verbatim, NOT 404 page
```
