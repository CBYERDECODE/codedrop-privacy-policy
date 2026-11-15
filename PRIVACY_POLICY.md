# QuickDeal — Privacy Policy

_Last updated: this document describes the MVP build. If the code and this page
ever disagree, the code is authoritative — see the verification table at the end._

## Summary

QuickDeal is a local-only coupon finder. It has **no account, no server, no
analytics, and makes no network requests** in this build. Everything it stores
lives on your device and is limited to two small records.

## What QuickDeal stores

| Storage key | Where | Contents | Why |
|---|---|---|---|
| `qd_settings` | `chrome.storage.local` | `{ privacyMode: boolean, badgeEnabled: boolean, merchantToggles: { [merchantId]: boolean }, affiliateEnabled: boolean }` | Remembers your switches so they survive a restart. |
| `qd_dismissals` | `chrome.storage.local` **or** `chrome.storage.session` (see Privacy Mode) | An array of **merchant ids** (e.g. `["amazon-us"]`) for which you closed the badge. | So the badge does not reappear on a store you dismissed. |

Merchant ids come from the bundled `data/merchants.json`. They identify a *store*
(e.g. `amazon-us`), not a page, product, or search.

## What QuickDeal never stores

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

If `chrome.storage.session` is unavailable, QuickDeal falls back to an **in-memory
map** rather than writing session-only dismissals to disk. Privacy takes
precedence over durability. (See `core/storage-manager.js`.)

## Permissions and why they exist

See `docs/PERMISSIONS.md` for the full justification. In short:
`storage` (remember your settings/dismissals), `activeTab` (read the page you are
on only when you open the popup, and never automatically), `clipboardWrite` (the
"Copy code" button).

## Network activity

There is none. The coupon dataset (`data/coupons.json`) and merchant list
(`data/merchants.json`) ship inside the extension. Opening the popup, mounting the
badge, copying a code, and filling a coupon field all happen entirely offline.

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
- **Reset all QuickDeal data** — removes `qd_settings` and `qd_dismissals` from
  both areas and clears the in-memory fallback.

Uninstalling the extension removes all of it as well.

## Children, analytics, third parties

No analytics, no telemetry, no third-party SDKs, no remote code. Nothing is shared
with anyone, because nothing leaves your device.

## Code ↔ policy verification

| Policy claim | Where the code enforces it |
|---|---|
| Dismissals live in `local` when Privacy Mode is OFF and `session` when ON | `dismissalsAreaName()` in `core/storage-manager.js` |
| Session-only dismissals never leak to disk | `areaFor()` / `memory` fallback in `core/storage-manager.js` |
| No hostname/URL is ever persisted | `describeStorage().persistsHostnames === false`; no storage write path accepts a URL |
| Settings are the only durable record besides dismissals | `KEYS` in `core/storage-manager.js` |
| No network requests | No `fetch`/`XMLHttpRequest` outside the service worker's local `chrome.runtime.getURL()` reads; no host permissions |
| Affiliate is off with an empty config | `normalizeAffiliateConfig()` in `core/affiliate-manager.js` |
| Sensitive fields are never touched | `isForbiddenField()` in `core/coupon-applier.js` |

The unit tests in `tests/unit/storage-manager.test.mjs` and
`tests/unit/affiliate-manager.test.mjs` assert these behaviours directly.
