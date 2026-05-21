---
name: meta-capi-dedup-setup
description: Wire up Meta Pixel + Direct Conversions API (CAPI) with browser↔server event_id dedup, server-side _fbp/_fbc cookie generation, external_id hashing, and PageView/ViewContent/AddToCart mirroring. Use when a Next.js (or any SSR) project needs production-grade Meta ads tracking with high Event Match Quality and resilience against ad-blockers / iOS ITP. Built from TWKCard implementation (verified by Meta Events Manager production reception).
---

# Meta Pixel + Direct CAPI with Dedup (Next.js)

> Pattern dari TWKCard. Production-ready. EMQ-optimized. No CAPI Gateway needed.

## Kapan dipakai

- Project punya Meta Pixel ID + CAPI Access Token (dari Events Manager)
- Mau bypass adblocker / iOS ITP tanpa pasang CAPI Gateway AWS/Birch
- Butuh tracking PageView / ViewContent / AddToCart / Lead / Purchase
- Mau **Event Match Quality (EMQ)** tinggi (≥ 7/10)

## Arsitektur (1 menit baca)

```
Browser              Your Next.js Server         Meta Graph API
  │                       │                        │
  │  GET /home            │                        │
  │ ────────────────────► │                        │
  │                       │  POST /events          │
  │                       │  (PageView+VC, eid=X)  │
  │                       │ ─────────────────────► │  (server fire)
  │                       │                        │
  │ ◄──────────────────── │ HTML+fbq snippet       │
  │   eventID: X embedded │                        │
  │                       │                        │
  │ fbq('track',...,{eventID:X}) ───────────────► │  (browser fire)
  │                       │                        │
  │                       │   Meta dedupes by      │
  │                       │   event_id within 60s  │
```

Both fires arrive at Meta with the **same `event_id`** → Meta deduplicates → counted as **1** event but with the **best signals from both sources** combined.

## File yang akan dibuat / diubah

```
packages/meta/src/index.ts                  # CAPI client + helpers
packages/tracking/src/eventId.ts            # UUID generator
packages/tracking/src/pixels.ts             # buildClickTrackerSnippet (browser)
apps/web/lib/capi-helpers.ts                # request → user_data builder
apps/web/app/[brand]/layout.tsx             # pixel snippet injection
apps/web/components/PageShell.tsx           # server-side CAPI PageView+VC fire
apps/web/components/ProductCardSSR.tsx      # AddToCart data-twk-event attrs
apps/web/app/api/click/[productId]/route.ts # AddToCart CAPI + cookie write
```

## Step-by-step

### 1. CAPI client (`packages/meta/src/index.ts`)

```ts
import crypto from 'node:crypto';

const META_GRAPH_VERSION = process.env.META_GRAPH_API_VERSION ?? 'v21.0';
const FBP_SUBDOMAIN_IDX = '1';

export function generateFbp(now = Date.now()): string {
  const rand = crypto.randomInt(1_000_000_000, 9_999_999_999);
  return `fb.${FBP_SUBDOMAIN_IDX}.${now}.${rand}`;
}

export function buildFbcFromFbclid(fbclid: string, now = Date.now()): string {
  return `fb.${FBP_SUBDOMAIN_IDX}.${now}.${fbclid}`;
}

export function sha256Hex(input: string): string {
  return crypto.createHash('sha256').update(input.trim().toLowerCase()).digest('hex');
}

export interface CapiUserData {
  emailHash?: string;
  phoneHash?: string;
  externalId?: string;   // sha256 of anon session id → EMQ boost
  ipAddress?: string;
  userAgent?: string;
  fbc?: string;
  fbp?: string;
}

export interface CapiEvent {
  eventName: 'PageView' | 'ViewContent' | 'AddToCart' | 'Lead' | 'Purchase' | string;
  eventId: string;
  eventTime?: number;
  sourceUrl: string;
  userData: CapiUserData;
  customData?: Record<string, unknown>;
}

export async function sendCapiEvents(input: {
  pixelId: string;
  accessToken: string;
  testEventCode?: string;
  events: CapiEvent[];
}) {
  const payload = {
    data: input.events.map((e) => ({
      event_name: e.eventName,
      event_time: e.eventTime ?? Math.floor(Date.now() / 1000),
      event_id: e.eventId,
      event_source_url: e.sourceUrl,
      action_source: 'website',
      user_data: {
        em: e.userData.emailHash,
        ph: e.userData.phoneHash,
        external_id: e.userData.externalId,
        client_ip_address: e.userData.ipAddress,
        client_user_agent: e.userData.userAgent,
        fbc: e.userData.fbc,
        fbp: e.userData.fbp,
      },
      custom_data: e.customData ?? {},
    })),
    ...(input.testEventCode ? { test_event_code: input.testEventCode } : {}),
    access_token: input.accessToken,
  };
  const res = await fetch(
    `https://graph.facebook.com/${META_GRAPH_VERSION}/${input.pixelId}/events`,
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload),
    }
  );
  return { ok: res.ok, status: res.status, body: await res.text() };
}
```

### 2. Browser tracker (`packages/tracking/src/pixels.ts`)

The trick: client passes `event_id` from server-rendered `data-twk-payload` to `fbq` as 3rd arg `{eventID}` — Meta uses this as dedup key.

```ts
export function buildClickTrackerSnippet(): string {
  return `(function(){
    function fire(eventName, payload){
      try {
        var eventId = payload && payload.event_id;
        if (window.fbq) {
          if (eventId) window.fbq('track', eventName, payload, {eventID: eventId});
          else window.fbq('track', eventName, payload);
        }
        if (window.ttq && typeof window.ttq.track === 'function') {
          var ttPayload = {
            value: payload.value,
            currency: payload.currency || 'IDR',
            contents: (payload.content_ids||[]).map(function(id){
              return {content_id:id, content_name:payload.content_name, content_type:payload.content_type||'product', price:payload.value, quantity:1};
            })
          };
          window.ttq.track(eventName, ttPayload);
        }
        if (window.gtag) window.gtag('event', eventName.toLowerCase(), {
          currency: payload.currency || 'IDR', value: payload.value,
          items: (payload.content_ids||[]).map(function(id){
            return {id:id, name:payload.content_name, price:payload.value, quantity:1};
          })
        });
      } catch(e){}
    }
    document.addEventListener('click', function(e){
      var el = e.target;
      while (el && el !== document.body) {
        if (el.dataset && el.dataset.twkEvent) {
          var payload = {};
          try { payload = JSON.parse(el.dataset.twkPayload || '{}'); } catch(_) {}
          fire(el.dataset.twkEvent, payload);
          return;
        }
        el = el.parentNode;
      }
    }, { capture: true, passive: true });
    window.twkTrack = fire;
  })();`;
}
```

### 3. Request → user_data helper (`apps/web/lib/capi-helpers.ts`)

```ts
import { generateFbp, buildFbcFromFbclid, sha256Hex, type CapiUserData } from '@your-meta-pkg';

export function buildCapiUserData(input: {
  ip?: string | null;
  userAgent?: string | null;
  fbpCookie?: string | null;
  fbcCookie?: string | null;
  fbclidParam?: string | null;
  sessionId?: string | null;
}): { userData: CapiUserData; fbpToSet?: string; fbcToSet?: string } {
  const now = Date.now();
  let fbp = input.fbpCookie ?? undefined;
  let fbpToSet: string | undefined;
  if (!fbp) { fbp = generateFbp(now); fbpToSet = fbp; }

  let fbc = input.fbcCookie ?? undefined;
  let fbcToSet: string | undefined;
  if (!fbc && input.fbclidParam) { fbc = buildFbcFromFbclid(input.fbclidParam, now); fbcToSet = fbc; }

  return {
    userData: {
      ipAddress: input.ip ?? undefined,
      userAgent: input.userAgent ?? undefined,
      fbp, fbc,
      externalId: input.sessionId ? sha256Hex(input.sessionId) : undefined,
    },
    fbpToSet, fbcToSet,
  };
}

export const FBP_COOKIE_OPTS = { name: '_fbp' as const, maxAge: 60*60*24*90, path: '/', sameSite: 'lax' as const, secure: true, httpOnly: false };
export const FBC_COOKIE_OPTS = { name: '_fbc' as const, maxAge: 60*60*24*90, path: '/', sameSite: 'lax' as const, secure: true, httpOnly: false };
```

### 4. Server-side fire in your page shell

```tsx
// In your main page Server Component:
const pageViewEid = generateEventId();
const viewContentEid = generateEventId();
const h = await headers();
const c = await cookies();
const { userData } = buildCapiUserData({
  ip: h.get('x-forwarded-for')?.split(',')[0]?.trim(),
  userAgent: h.get('user-agent'),
  fbpCookie: c.get('_fbp')?.value,
  fbcCookie: c.get('_fbc')?.value,
  sessionId: c.get('your_session_cookie')?.value,
});
void sendCapiEvents({
  pixelId: brand.metaPixelId,
  accessToken: brand.metaAccessToken,
  testEventCode: process.env.META_TEST_EVENT_CODE || undefined,
  events: [
    { eventName: 'PageView', eventId: pageViewEid, sourceUrl, userData },
    { eventName: 'ViewContent', eventId: viewContentEid, sourceUrl, userData, customData: { ... } },
  ],
}).catch(() => {});

// Inline script: client fbq with SAME event_id
const script = `(function(){function fire(){
  if (window.fbq) window.fbq('track', 'ViewContent', ${JSON.stringify(payload)}, {eventID: ${JSON.stringify(viewContentEid)}});
}; if (document.readyState === 'complete') setTimeout(fire, 50); else window.addEventListener('load', function(){setTimeout(fire,50)});})();`;
return <main><script dangerouslySetInnerHTML={{__html: script}} />...</main>;
```

### 5. Click route (AddToCart)

```ts
// /api/click/[id]/route.ts
const eventId = url.searchParams.get('eid') ?? generateEventId(); // SSR may have pre-generated
const { userData, fbpToSet, fbcToSet } = buildCapiUserData({
  ip, userAgent, fbpCookie, fbcCookie,
  fbclidParam: url.searchParams.get('fbclid'),
  sessionId,
});
void sendCapiEvents({ pixelId, accessToken, events: [{ eventName: 'AddToCart', eventId, sourceUrl, userData, customData: { content_ids:[id], value: priceIdr, currency:'IDR' } }] });

const res = NextResponse.redirect(destination, 302);
if (fbpToSet) res.cookies.set({ ...FBP_COOKIE_OPTS, value: fbpToSet });
if (fbcToSet) res.cookies.set({ ...FBC_COOKIE_OPTS, value: fbcToSet });
return res;
```

### 6. Product card (browser-side fire)

```tsx
const eventId = generateEventId(); // SSR
const href = `/api/click/${product.id}?target=shopee&b=${brand.id}&eid=${eventId}`;
const payload = JSON.stringify({
  content_ids: [product.id], content_name: product.name, content_type: 'product',
  value: product.priceIdr, currency: 'IDR', event_id: eventId, // shared!
});
return <a href={href} data-twk-event="AddToCart" data-twk-payload={payload}>Beli</a>;
```

## Verification

1. **Test mode**: set `META_TEST_EVENT_CODE=TESTxxxxx` in `.env`. Open Events Manager → Test Events → trigger flow. Should see PageView + ViewContent dengan badge "Dideduplikasi" (Browser + Server tick).

2. **Direct CAPI smoke test**:
```bash
curl -X POST "https://graph.facebook.com/v21.0/{PIXEL_ID}/events" \
  -H "Content-Type: application/json" \
  -d '{"data":[{"event_name":"PageView","event_time":'$(date +%s)',"event_id":"smoke-1","action_source":"website","event_source_url":"https://example.com","user_data":{"client_ip_address":"1.2.3.4","client_user_agent":"curl"}}],"access_token":"YOUR_TOKEN"}'
# Should return: {"events_received":1,"messages":[],"fbtrace_id":"..."}
```

3. **REMOVE `META_TEST_EVENT_CODE` from .env after verification**, else production traffic stays stuck in Test mode.

## Common gotchas

| Gotcha | Fix |
|--------|-----|
| `value: priceIdr / 100` divides by cents | priceIdr in IDR is integer rupiah, NOT cents. Send as-is. |
| Browser fbq + server CAPI fire BOTH but Meta shows 2× events | Missing dedup. Browser fbq's 3rd arg MUST be `{eventID: same_uuid}`. |
| EMQ score stuck at 4-5/10 | Add `external_id` (hashed session id), `_fbp`, `_fbc`, `client_ip_address`, `client_user_agent`. |
| Server-rendered `_fbp` differs from what browser SDK sets | Write the server-generated `_fbp` to response cookie so SDK reuses it. |
| `events_received:1` but Events Manager "Setup events" milestone won't tick | Token has `test_event_code` set → events stuck in Test tab. Remove env var. |
| "Klaim akun Birch / CAPI Gateway" warning di Events Manager | Direct CAPI doesn't need Gateway. Biarkan warning expire — won't break anything. |

## When NOT to use this pattern

- Stack tidak SSR (pure SPA) → use Tag Manager + CAPI Gateway instead.
- High traffic > 10k events/sec → consider CAPI Gateway for backpressure handling.
- Tim non-eng yang nggak comfort dengan code → pakai Birch SaaS Gateway.

## Reference

- https://developers.facebook.com/docs/marketing-api/conversions-api
- https://developers.facebook.com/docs/marketing-api/conversions-api/parameters/fbp-and-fbc
- https://developers.facebook.com/docs/marketing-api/conversions-api/deduplicate-pixel-and-server-events/
