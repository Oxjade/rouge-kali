# Pay Wallet TG — Full Security Assessment (Maximum Verbosity)

**Target:** `pay.wallet.tg` — Wallet Merchant Payment Processing  
**Date:** 2026-07-27  
**Methodology:** Rouge Kali Track A — Web/App Security (Maximum)  
**Status:** Complete  

---

## Infrastructure

| Component | Value |
|-----------|-------|
| Origin IPs | 45.196.29.100, 45.196.29.101, 45.196.29.102 |
| CDN/WAF | Cloudflare (all 3 IPs proxied) |
| SSL | `CN=wallet.tg` wildcard (\*.wallet.tg), Google Trust Services (WE1) |
| HSTS | `max-age=2592000; includeSubDomains; preload` |
| Backend | Java/Spring Boot (error response: "No static resource ..."), Envoy sidecar |
| Frontend | React SPA (MUI, Emotion, Webpack) |
| Monitoring | Sentry (2 projects), No OTel collector on this host |
| Auth | Telegram OAuth 2.0 with PKCE + cookie-based sessions |
| API Prefixes | `/alectryon/public-api/wpay/` (auth), `/wpay/website-api/v1/` (business) |

---

## Subdomain Map

| Host | IP | Service |
|------|----|---------|
| `pay.wallet.tg` | 45.196.29.100-102 | Merchant payment portal (React SPA) |
| `www.wallet.tg` | 45.196.29.100-102 | Same SPA (alias) |
| `docs.wallet.tg` | 45.196.29.100-102 | Same SPA (alias) |
| `help.wallet.tg` | 18.210.189.28 | Wallet Help Center (Play Framework, AWS) |
| `*.walletbot.me` | 45.196.29.12 | Backend services (alectryon, events-gateway, p2p) |

---

## Open Ports

### pay.wallet.tg (45.196.29.100-102)
| Port | Status | Service |
|------|--------|---------|
| 80 | Open | HTTP → redirect to HTTPS |
| 443 | Open | HTTPS (Cloudflare proxied) |
| 8080 | Open | HTTP (direct origin, cert mismatch) |
| 8443 | Open | HTTPS (direct origin, Envoy, returns 403) |

### help.wallet.tg (18.210.189.28)
| Port | Status | Service |
|------|--------|---------|
| 80 | Open | HTTP |
| 443 | Open | HTTPS (Caddy + istio-envoy) |

---

## API Surface (REST)

### Unauthenticated Endpoints

| Method | Path | Status | Response |
|--------|------|--------|----------|
| GET | `/wpay/website-api/v1/auth/get-ip-country` | **200** | Country code (`NG`) — info leak |

### Authenticated Endpoints (return 401 without auth)

| Method | Path | Description |
|--------|------|-------------|
| POST | `/wpay/website-api/v1/auth/authorize-by-telegram` | WebApp initData login (500 error) |
| POST | `/wpay/website-api/v1/merchant` | Create merchant record |
| GET | `/wpay/website-api/v1/merchant` | Get merchant profile |
| POST | `/wpay/website-api/v1/merchant/application/company/create` | Company application |
| POST | `/wpay/website-api/v1/merchant/application/company/update` | Update company app |
| POST | `/wpay/website-api/v1/merchant/application/individual/create` | Individual application |
| POST | `/wpay/website-api/v1/merchant/application/individual/update` | Update individual app |
| POST | `/wpay/website-api/v1/merchant/application/submit` | Submit for review |
| POST | `/wpay/website-api/v1/merchant/application/revoke` | Revoke application |
| POST | `/wpay/website-api/v1/merchant/application/get-verification-token` | KYC token |
| POST | `/wpay/website-api/v1/merchant/get-verification-token` | Merchant KYC token |
| POST | `/wpay/website-api/v1/store/create` | Create store |
| POST | `/wpay/website-api/v1/store/rename` | Rename store |
| POST | `/wpay/website-api/v1/store/api-key/decrypt` | **Decrypt API key** |
| POST | `/wpay/website-api/v1/store/api-key/recreate` | Recreate API key |
| POST | `/wpay/website-api/v1/store/webhook-config` | Webhook configuration |
| GET | `/wpay/website-api/v1/store` | List stores |
| GET | `/wpay/website-api/v1/order/get-list` | List orders |
| GET | `/wpay/website-api/v1/order/get-list-as-file` | Export orders |
| POST | `/wpay/website-api/v1/payout/calculate` | Calculate payout |
| POST | `/wpay/website-api/v1/payout/proceed` | Execute payout |
| POST | `/wpay/website-api/v1/payout/proceed-all-accounts` | Payout all |
| POST | `/wpay/website-api/v1/payout/configure-autopayout` | Auto-payout config |
| GET | `/wpay/website-api/v1/payout/get-list` | List payouts |
| GET | `/wpay/website-api/v1/payout/get-list-as-file` | Export payouts |
| GET | `/wpay/website-api/v1/payout/status` | Payout status |
| GET/POST | `/wpay/website-api/v1/settings/app` | App settings |

### Auth Endpoints

| Method | Path | Status | Notes |
|--------|------|--------|-------|
| GET | `/alectryon/public-api/wpay/login?provider_name=telegram` | **302** → Telegram OAuth | PKCE + state cookie |
| POST | `/alectryon/public-api/wpay/auth-refresh` | **401** (needs cookie) | Uses `refresh_token_wpay` cookie |
| GET | `/alectryon/public-api/wpay/logout` | **302** → `/` | Clears session |
| GET | `/alectryon/public-api/wpay/callback` | **400/401** | OAuth callback (state+code validation) |

---

## Findings

### F-01: Unauthenticated IP Geolocation (Low)

**Endpoint:** `GET /wpay/website-api/v1/auth/get-ip-country`  
**Status:** Confirmed (PoC)  
**CVSS:** 3.7 (AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N)

Returns the 2-letter country code of the requesting IP without any authentication. Leaks approximate geographic location.

**Reproduction:**
```bash
curl -sk "https://pay.wallet.tg/wpay/website-api/v1/auth/get-ip-country"
# Response: NG
```

---

### F-02: Internal Server Error on Authorize Endpoint (Medium)

**Endpoint:** `POST /wpay/website-api/v1/auth/authorize-by-telegram`  
**Status:** Confirmed (PoC)  
**CVSS:** 5.3 (AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:L)

Returns `HTTP 500 {"status":"INTERNAL_ERROR"}` for any request body. All variations tested (empty, valid JSON, invalid JSON, plain string, nested objects) trigger the same error. Indicates an unhandled exception (likely NPE) in the Spring Boot controller.

While error details are not leaked, the endpoint can be used for:
- Denial of service (repeated 500s consume server resources)
- Potential for future exploitation if the error path has a vulnerability

**Reproduction:**
```bash
curl -sk -X POST "https://pay.wallet.tg/wpay/website-api/v1/auth/authorize-by-telegram" \
  -H "Content-Type: application/json" -d '{}'
# Response: {"status":"INTERNAL_ERROR"} (500)
```

---

### F-03: Sentry DSN Exposure (Informational)

**Location:** Main JS bundle (`main.eeb06d.js`)  
**Status:** Confirmed

Two Sentry DSNs hardcoded in the frontend JS:

| DSN URL | Project |
|---------|---------|
| `https://4e467c24a2a9458aaeed0cf20e424ebc@sentry.walletbot.me/41` | Wallet Pay |
| `https://f06bb85740784985908fda7572382d06@sentry.walletbot.me/40` | Unknown |

These are public DSNs (Sentry DSNs are designed to be public). However, they confirm Sentry is used for error monitoring and provide a potential attack surface for injecting fake error reports.

---

### F-04: Missing Content Security Policy (Medium)

**Status:** Confirmed  

No `Content-Security-Policy` header on any response. The SPA loads external resources from:
- `fonts.googleapis.com`, `fonts.gstatic.com`
- `www.googletagmanager.com` (Google Analytics)
- Sentry (dynamic)

Without CSP, if an XSS vulnerability is found, the attacker can exfiltrate data, load malicious scripts, or perform crypto-jacking.

**Observed security headers:**
```
strict-transport-security: max-age=2592000; includeSubDomains; preload ✓
x-content-type-options: nosniff ✓
x-frame-options: DENY ✓
x-xss-protection: 0 ✓
content-security-policy: MISSING ✗
```

---

### F-05: OAuth Implementation Review (Informational)

**Status:** Observed — no vulnerability confirmed

The Telegram OAuth 2.0 flow uses:
- **PKCE** (S256 code challenge method) ✓
- **State parameter** with HttpOnly cookie (`state_wpay`) ✓
- **Redirect URI** restricted to `/alectryon/public-api/wpay/callback` (server-side) ✓
- **Nonce** for OIDC ✓
- **Scope:** `openid profile phone` (requests phone number)

The state cookie has `Path=/alectryon/public-api/wpay` and `SameSite=Lax`, preventing CSRF on the callback endpoint.

---

### F-06: Help Center — Separate Infrastructure (Informational)

**Status:** Observed

`help.wallet.tg` runs on a completely different stack:
- **Server:** Caddy + Istio-Envoy
- **Framework:** Play Framework (Scala)
- **Hosting:** AWS (18.210.189.28)
- **Static assets:** CloudFront + S3

No vulnerabilities confirmed on the help center, but its different security posture warrants separate assessment.

---

### F-07: Merchant API Key Decryption Endpoint (High Risk)

**Endpoint:** `POST /wpay/website-api/v1/store/api-key/decrypt`  
**Status:** Requires auth — cannot verify

If accessible to merchants, this endpoint allows decryption of stored API keys. If abused via IDOR (lack of authorization checks), a malicious merchant could decrypt another merchant's API key. Requires authentication to test.

---

### F-08: Refresh Token in Cookie (Medium)

**Endpoint:** `POST /alectryon/public-api/wpay/auth-refresh`  
**Status:** Confirmed (responds to cookie presence)

Uses `refresh_token_wpay` cookie with the following properties:
- `HttpOnly` (not client-readable) ✓
- `Secure` (HTTPS only) ✓
- `SameSite=Lax` (CSRF protection) ✓
- Path: not observed (defaults to `/`)

Returns `"Invalid wpay refresh token"` for invalid tokens and `"refresh_token_wpay is missing in cookie"` when absent. The error messages differentiate between "missing token" and "invalid token," which enables a minor enumeration attack.

---

## Exploitation Routes (Requires Auth)

The following attack chains require a valid merchant session:

1. **API Key Theft:** `POST /store/api-key/decrypt` — decrypt stored API keys for payment processing
2. **Store Takeover:** `POST /store/rename`, `POST /store/api-key/recreate` — modify store configuration
3. **Fund Theft:** `POST /payout/proceed`, `POST /payout/proceed-all-accounts` — initiate unauthorized payouts
4. **IDOR Testing:** Store IDs appear to be UUIDs — test if merchant A can access merchant B's store

---

## CVSS Summary

| ID | Finding | CVSS | Severity |
|----|---------|------|----------|
| F-01 | Unauthenticated IP Geolocation | 3.7 | Low |
| F-02 | Internal Server Error on Auth Endpoint | 5.3 | Medium |
| F-03 | Sentry DSN Exposure | 0.0 | Info |
| F-04 | Missing CSP Header | 5.0 | Medium |
| F-05 | OAuth Implementation | 0.0 | Info |
| F-06 | Help Center Infrastructure | 0.0 | Info |
| F-07 | API Key Decryption Endpoint | — | Requires Auth |
| F-08 | Refresh Token Cookie Analysis | 3.1 | Low |

---

## Recommendations

1. **Fix `authorize-by-telegram` internal error** — add proper input validation and error handling
2. **Add CSP header** — restrict script sources, block inline scripts
3. **Rate-limit auth endpoints** — prevent DoS via repeated 500 errors
4. **Audit IDOR on store/merchant endpoints** — ensure proper authorization on all API key and payout operations
5. **Rate-limit refresh token validation** — prevent enumeration of valid vs invalid tokens
6. **Separate assessment for help.wallet.tg** — different tech stack needs its own review
7. **Consolidate Sentry projects** — reduce monitoring fragmentation

---

## References
- `wallettg-net/full-assessment.md` — previous assessment of main wallet backend
- `wallettg-net/otlp-operational-damage.md` — OTel collector analysis (shared infra)
