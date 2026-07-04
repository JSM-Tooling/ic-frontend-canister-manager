# Custom Domain Setup for IC Frontend Canisters

This guide walks through connecting a custom domain to a frontend asset canister deployed with this repo. It's written to be registrar-agnostic, with call-outs for **home.pl** where its panel behaves differently from a "standard" DNS provider.

Replace `<YOUR-DOMAIN>` and `<CANISTER_ID>` below with your actual values throughout.

---

## 0. Before you start

- The frontend canister must already be deployed to mainnet (`dfx deploy <CANISTER_NAME> --ic ...`). Custom domain registration only works against a live canister.
- You need write access to your domain's DNS zone (registrar panel, or a DNS provider you've delegated to).
- Decide up front: **apex domain or `www` subdomain?** See [Apex vs. subdomain](#3-apex-vs-subdomain) before touching DNS — it changes which records are even possible.

---

## 1. Domain-ownership file (`.well-known/ic-domains`)

The canister must declare which domain(s) it's willing to serve, via a file at:

```
/.well-known/ic-domains
```

containing one domain per line, e.g.:

```
www.<YOUR-DOMAIN>
```

This repo already has a template for this at `custom-domain-files/.well-known/ic-domains` — replace the placeholder domain there. `custom-domain-files/.ic-assets.json5` is also included and required: by default the asset canister ignores dotfiles, so without it `.well-known/` silently won't be served.

```json5
// custom-domain-files/.ic-assets.json5
[
  { "match": ".well-known", "ignore": false },
  { "match": "**/*", "security_policy": "standard" }
]
```

**Deploy, then verify the file is actually live before touching DNS:**

```bash
curl https://<CANISTER_ID>.icp0.io/.well-known/ic-domains
# should echo back your domain(s)
```

> **Gotcha:** if your deploy pipeline rebuilds/overwrites the asset source directory from scratch on every deploy (e.g. copying a fresh `dist/`), make sure `.well-known/ic-domains` and `.ic-assets.json5` are re-injected on every deploy rather than living inside the directory that gets wiped. In this repo that's what `task:copy-files` (`cp -r custom-domain-files/. dist/`) is for — run it as part of every deploy, not just once.

---

## 2. DNS records

Three records are required, all typically set on the `www` (or chosen) subdomain — **not** on the bare apex (see [§3](#3-apex-vs-subdomain) for why):

| Type  | Host                          | Value                                        |
|-------|-------------------------------|-----------------------------------------------|
| CNAME | `www`                         | `www.<YOUR-DOMAIN>.icp1.io`                    |
| TXT   | `_canister-id.www`            | `<CANISTER_ID>`                                |
| CNAME | `_acme-challenge.www`         | `_acme-challenge.www.<YOUR-DOMAIN>.icp2.io`    |

**Important:** the CNAME target genuinely embeds your full domain (`<YOUR-DOMAIN>.icp1.io`, not a bare `icp1.io`). This is the official IC format, not a registrar quirk — don't "simplify" it to just `icp1.io`.

### Notes for home.pl

- home.pl's DNS form has two fields per record: **Host** and **nazwa kanoniczna** (canonical name / target). Put the subdomain (`www`, `_acme-challenge.www`, `_canister-id.www`) in **Host**, and the full target value (including the trailing dot IC docs show) in **canonical name**.
- home.pl's own authoritative nameservers can lag by a few minutes after you save a record. If a fresh record doesn't resolve immediately — even when you query `dns.home.pl` / `dns2.home.pl` / `dns3.home.pl` directly — that's not necessarily a misconfiguration; wait a few minutes and re-check before assuming the record was entered wrong. That said, if sibling records added in the same batch already resolve and one specific record still comes back empty after several minutes, stop waiting and re-check that record's exact values — at that point it's more likely a real input mistake than propagation lag.
- **Sub-view Host suffix trap:** home.pl's panel can drop you into a DNS view scoped to a specific subdomain (e.g. you navigated into settings *for* `biz.<YOUR-DOMAIN>`). On that scoped page, the "Host" field silently appends `.biz.<YOUR-DOMAIN>` to whatever you type — visible as literal suffix text next to the input. Typing nothing sensible there, or assuming it behaves like the top-level zone's Host field, creates a record one level too deep (e.g. `www.biz.<YOUR-DOMAIN>` instead of `biz.<YOUR-DOMAIN>` itself). **Fix:** to create a record at the bare subdomain name itself, add it from the **parent zone's** DNS page (the top-level `<YOUR-DOMAIN>` view, Host: `biz`), not from a page already scoped into that subdomain.
- Verify records against the authoritative servers directly, not just your local resolver:
  ```bash
  dig NS <YOUR-DOMAIN>          # find dns.home.pl / dns2.home.pl / dns3.home.pl
  dig @dns2.home.pl +short CNAME www.<YOUR-DOMAIN>
  dig @dns2.home.pl +short TXT  _canister-id.www.<YOUR-DOMAIN>
  ```

---

## 3. Apex vs. subdomain

A `CNAME` record cannot exist on the bare apex (`<YOUR-DOMAIN>` with no subdomain) — the apex must carry `SOA`/`NS` records, and DNS forbids mixing those with a `CNAME` at the same name. So:

- **Subdomain (`www.<YOUR-DOMAIN>`, `app.<YOUR-DOMAIN>`, etc.)** — always works, no special DNS provider features needed. Recommended default.
- **Bare apex (`<YOUR-DOMAIN>`)** — requires one of:
  - An `ALIAS`/`ANAME` record at the apex, if your DNS provider supports it (many don't — **home.pl does not**).
  - A registrar-level domain-forwarding/redirect feature that 301-redirects the apex to `www.<YOUR-DOMAIN>` (**home.pl does not offer this either**).
  - Migrating DNS to a provider that supports `ALIAS`/`ANAME` and/or redirect rules (e.g. Cloudflare's free plan) while keeping the domain registered elsewhere.

**If you're on home.pl and don't want to migrate DNS elsewhere:** the pragmatic default is to only serve the app on the `www` subdomain and accept that the bare apex won't resolve to your app (it'll show the registrar's default parking page instead). This is a cosmetic/branding tradeoff, not a functional one — decide based on whether you need the bare domain to work.

---

## 4. Register the domain with IC

Once DNS is live (double-check with `dig` against the authoritative NS, not just your resolver), register via the custom-domains API:

```bash
# 1. Validate — confirms DNS + ownership file are correctly set up
curl -sL -X GET "https://icp.net/custom-domains/v1/www.<YOUR-DOMAIN>/validate"

# 2. Register
curl -sL -X POST "https://icp.net/custom-domains/v1/www.<YOUR-DOMAIN>"

# 3. Check status (registering → registered)
curl -sL -X GET "https://icp.net/custom-domains/v1/www.<YOUR-DOMAIN>"
```

Other operations on the same endpoint:

```bash
# Repoint to a different canister
curl -sL -X PATCH "https://icp.net/custom-domains/v1/www.<YOUR-DOMAIN>"

# Remove
curl -sL -X DELETE "https://icp.net/custom-domains/v1/www.<YOUR-DOMAIN>"
```

> **Deprecated endpoint warning:** older guides/scripts may reference `https://icp0.io/registrations` — that endpoint is gone and returns a `400 canister_id_not_resolved` error. Use `icp.net/custom-domains/v1/...` as shown above.

---

## 5. Verify

After `registration_status` reaches `registered`, allow a few minutes for the TLS certificate to propagate to all boundary nodes before testing.

```bash
curl -sL -o /dev/null -w "%{http_code}\n" https://www.<YOUR-DOMAIN>/
# expect 200

echo | openssl s_client -connect www.<YOUR-DOMAIN>:443 -servername www.<YOUR-DOMAIN> 2>/dev/null \
  | openssl x509 -noout -subject
# expect: subject=CN = www.<YOUR-DOMAIN>
```

**Expected transient behavior right after registering:** the TLS certificate check above may briefly show an unrelated domain's certificate (a boundary node's fallback cert) before your certificate has propagated. This is normal — recheck a few minutes later rather than assuming registration failed.

---

## 6. Sharing an Internet Identity principal across multiple domains

By default, Internet Identity derives a **different principal per origin/hostname** — a user logging into `https://a.<YOUR-DOMAIN>` gets a different principal than on `https://b.<YOUR-DOMAIN>`, even with the same passkey. If you run the same user base across multiple custom domains (e.g. two frontend canisters that should share one identity), you need the **alternative origins** mechanism.

This is purely origin-based — it works across two completely different canisters, not just two domains on the same canister, since principal derivation only depends on the hostname string used, never on which canister happens to serve the assets at that hostname.

### 6.1 Pick a canonical origin

Choose one domain as canonical (the one whose principal gets reused everywhere). Every other domain is an "alternate."

### 6.2 Host the alternate-origins file — on the canonical domain only

```
/.well-known/ii-alternative-origins
```

```json
{
  "alternativeOrigins": ["https://alternate.<YOUR-DOMAIN>"]
}
```

- Max **10** alternative origins.
- Full `https://` URLs, no trailing slash, no path.
- Add matching `.ic-assets.json5` rules so it's served with the right content type and is fetchable cross-origin (the II frontend fetches this file via browser `fetch()` from its own origin):

```json5
[
  { "match": ".well-known", "ignore": false },
  {
    "match": ".well-known/ii-alternative-origins",
    "headers": {
      "Access-Control-Allow-Origin": "*",
      "Content-Type": "application/json"
    },
    "ignore": false
  },
  { "match": "**/*", "security_policy": "standard" }
]
```

Deploy the canonical domain's canister, then verify:

```bash
curl -sI https://<CANONICAL_DOMAIN>/.well-known/ii-alternative-origins
# expect: content-type: application/json, access-control-allow-origin: *
```

### 6.3 Pass `derivationOrigin` — only in the alternate domain's frontend code

In the app that's served on the **alternate** domain (not the canonical one), pass `derivationOrigin` pointing at the canonical origin when calling `AuthClient.login()`:

```ts
const CANONICAL_ORIGIN = 'https://<CANONICAL_DOMAIN>';
const ALTERNATE_HOSTNAMES = ['<alternate-domain>'];

function getDerivationOrigin() {
  return ALTERNATE_HOSTNAMES.includes(window.location.hostname) ? CANONICAL_ORIGIN : undefined;
}

client.login({
  identityProvider: II_URL,
  derivationOrigin: getDerivationOrigin(),
  // ...
});
```

Gate it on `window.location.hostname` like this rather than hardcoding it unconditionally — otherwise local dev (`localhost`) and any other deploy of the same codebase would also try to derive against the canonical origin and fail, since the canonical's `ii-alternative-origins` file won't list `localhost` as an alternate. The canonical domain's own deploy needs no `derivationOrigin` at all.

### 6.4 Verify

Log in from the alternate domain, then from the canonical domain, and confirm both produce the same principal (e.g. print `identity.getPrincipal().toText()` in your app, or compare whatever your backend sees as `caller`).

---

## 7. Reference

Official docs: https://docs.internetcomputer.org/guides/frontends/custom-domains/

The registration API has moved once already (`icp0.io/registrations` → `icp.net/custom-domains/v1`) — if any command in this file starts failing outright, re-check that page before assuming your own setup is wrong.
