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
- home.pl's own authoritative nameservers can lag by a few minutes after you save a record. If a fresh record doesn't resolve immediately — even when you query `dns.home.pl` / `dns2.home.pl` / `dns3.home.pl` directly — that's not necessarily a misconfiguration; wait a few minutes and re-check before assuming the record was entered wrong.
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

## 6. Reference

Official docs: https://docs.internetcomputer.org/guides/frontends/custom-domains/

The registration API has moved once already (`icp0.io/registrations` → `icp.net/custom-domains/v1`) — if any command in this file starts failing outright, re-check that page before assuming your own setup is wrong.
