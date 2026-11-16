# CodeDrop — Privacy Policy

_Last updated: this document describes the MVP build. If the code and this page
ever disagree, the code is authoritative — see the verification table at the end._

## Summary

CodeDrop is a privacy-first coupon finder. It has **no account, no analytics and
no server of its own**, and makes **no network request** until you connect a
coupon service — a single HTTPS endpoint *you* choose on the options page.
Everything CodeDrop stores lives on your device: your settings, the merchant ids
you dismissed, and a short-lived session cache of offers.

## What CodeDrop stores

| Storage key | Where | Contents | Why |
|---|---|---|---|
| `qd_settings` | `chrome.storage.local` | `{ privacyMode, badgeEnabled, merchantToggles, affiliateEnabled, regionOverride, couponServiceUrl }` | Remembers your switches, your region choice, and the coupon service you connected, so they survive a restart. |
| `qd_dismissals` | `chrome.storage.local` **or** `chrome.storage.session` (see Privacy Mode) | An array of **merchant ids** (e.g. `["amazon-us"]`) for which you closed the badge. | So the badge does not reappear on a store you dismissed. |
| `qd_coupon_cache` | `chrome.storage.session` **only** | `{ [merchantId]: { at, source, coupons } }` — the offers most recently fetched from your connected service, for up to 30 minutes. | So the popup and the badge do not re-query your service on every page load. Never written to disk. |

Merchant ids come from the bundled `data/merchants.json`. They identify a *store*
(e.g. `amazon-us`), not a page, product, or search.

## What CodeDrop never stores

- Page URLs, hostnames, or browsing history.
- Page content, HTML, cart contents, prices, or order details.
- Coupon codes you copy, clipboard contents, or form input.
- Any personal, payment, address, or account information. The applier refuses to
  touch password, payment, and address fields (see `core/coupon-applier.js`).

## Privacy Mode

Privacy Mode changes **where dismissals live**. It never changes *what* is stored.

| | Privacy Mode OFF (default) | Privacy Mode ON |
|---|---|---|
| Dismissals area | `chrome.storage.local` | `chrome.storage.session` |
| Dismissals survive browser restart | Yes | **No** — cleared when the session ends |
| Hostnames / URLs stored | Never | Never |
| Settings stored | Yes (`qd_settings`) | Yes (`qd_settings`) |

If `chrome.storage.session` is unavailable, CodeDrop falls back to an **in-memory
map** rather than writing session-only dismissals to disk. Privacy takes
precedence over durability. (See `core/storage-manager.js`.)

## Permissions and why they exist

See `docs/PERMISSIONS.md` for the full justification. In short:
`storage` (remember your settings/dismissals), `activeTab` (read the page you are
on only when you open the popup, and never automatically), `clipboardWrite` (the
"Copy code" button).

## Network activity

CodeDrop makes **no network request at all** until you connect a coupon service
on the options page. When you do — and only then:

- The **service worker** sends one `GET` to the single HTTPS endpoint you
  configured, carrying your **merchant id** (e.g. `amazon-us`) as the only
  parameter, e.g. `https://coupons.example.com/api?merchant=amazon-us`. Nothing
  else is sent.
- The request carries **no cookies and no credentials** (`credentials: "omit"`),
  no referrer, and no page content. Only `https:` endpoints are accepted.
- A malformed or unreachable response is reported as an error — CodeDrop never
  renders a made-up coupon.
- The result is cached in **session storage only** for up to 30 minutes (see the
  table above) and is discarded when the browser session ends.

The extension asks for access to **just that one origin** (via
`chrome.permissions.request`) and gives it back when you disconnect. If you
decline, the request can still succeed when the service sends permissive CORS
headers — the connection test tells you truthfully which happened.

There is **no CodeDrop server, no analytics, no telemetry and no third-party
SDK**. The bundled merchant list (`data/merchants.json`) is the only data that
ships inside the extension, and it is read locally via
`chrome.runtime.getURL()`.

Region and currency are detected entirely offline. CodeDrop derives your region
(US, UK, EU or India) from your explicit setting, the store's own domain, or your
browser's language, in that order. There is **no IP lookup and no geolocation
request**, and amounts are always shown in the store's native currency —
CodeDrop does **not** convert between currencies.

## Affiliate links

The MVP ships with an empty affiliate configuration
(`data/affiliate-config.json`), so the feature is **fully disabled** and no
affiliate UI appears anywhere. If a future build enables it: links would be opened
**only** on an explicit click, in a new tab, from config-defined URLs (never
invented, never rewritten), and the disclosure would be shown wherever a partner
link is offered.

## Deleting your data

The options page provides two controls:

- **Clear badge dismissals** — removes `qd_dismissals` from both storage areas.
- **Reset all CodeDrop data** — removes `qd_settings` and `qd_dismissals` from
  both areas and clears the in-memory fallback.

Uninstalling the extension removes all of it as well.

## Contact

Questions about this policy or about the extension:

- Email: `codedrop.support@gmail.com`
- Canonical URL for this policy: `https://cbyerdecode.github.io/codedrop-privacy-policy/`

## Children, analytics, third parties

No analytics, no telemetry, no third-party SDKs, no remote code. The only
outbound request is the one described above, to the coupon service **you** wired
up yourself; CodeDrop never contacts anything you did not choose.

## Code ↔ policy verification

| Policy claim | Where the code enforces it |
|---|---|
| Dismissals live in `local` when Privacy Mode is OFF and `session` when ON | `dismissalsAreaName()` in `core/storage-manager.js` |
| Session-only dismissals never leak to disk | `areaFor()` / `memory` fallback in `core/storage-manager.js` |
| No hostname/URL is ever persisted | `describeStorage().persistsHostnames === false`; the offer cache stores merchant ids only |
| Settings are the only durable record besides dismissals | `KEYS` / `DEFAULT_SETTINGS` in `core/storage-manager.js` |
| The offer cache never touches disk | `getCouponCache()` / `setCouponCache()` write only to `chrome.storage.session` |
| Network is opt-in, single-origin and credential-free | `sanitizeEndpoint()` + `credentials: 'omit'` in `core/remote-provider.js`; origin requested/released in `options/options.js` (`requestOrigin` / `releaseOrigin`) |
| Affiliate is off with an empty config | `normalizeAffiliateConfig()` in `core/affiliate-manager.js` |
| Sensitive fields are never touched | `isForbiddenField()` in `core/coupon-applier.js` |

The unit tests in `tests/unit/storage-manager.test.mjs`,
`tests/unit/remote-provider.test.mjs` and `tests/unit/affiliate-manager.test.mjs`
assert these behaviours directly.
