# IC Frontend Canister Manager

A template for the **deploy manager** of a client's websites on the Internet
Computer. One manager repo (`aaa-<client>-frontend-manager`) holds everything a
production deploy needs — `dfx.json`, canister ids, custom-domain files — for
one or more frontend canisters, and gives a one-command test deploy through
[`motoko-crafting-table`](https://github.com/JSM-Tooling/motoko-crafting-table).

The web projects stay free of deploy details. Each one only knows how to build
and how to copy its `dist` into the manager; the manager decides where the
build goes.

```
web project                     manager                                        target
───────────                     ───────                                        ──────
npm run build ──▶ dist/ ──copy──▶ ./dist ──┬─ deploy:test -- <slot> ──▶ ~/motoko-crafting-table/<slot> ──▶ shared test canister
                                           │                                                               (reinstall, .icp0.io URL)
                                           └─ deploy:prod ──▶ ./dist + custom-domain files ──▶ dfx deploy --ic ──▶ production canister
```

For a manager that serves several sites the production side is namespaced
per site — see [Several sites in one manager](#several-sites-in-one-manager).
Everything else is the same.

---

## Quick start: a new site

**Prerequisites:** `dfx` installed, an identity with cycles for the production
canister, `motoko-crafting-table` cloned at `~/motoko-crafting-table`, and a
web project that builds into `dist/`.

Is this the client's first site? Do steps A–E. Does the client already have a
manager? Skip A and follow [Adding a site](#adding-a-site) instead.

### A. Create the manager

1. Create a GitHub repo named `aaa-<client>-frontend-manager` and clone it as
   a **sibling of the web project** (the web project's scripts use
   `../aaa-<client>-frontend-manager`).
2. Copy the contents of this template into it (download the zip, or clone
   and copy — without `.git`).
3. Replace the placeholders:

   | Placeholder | Where | Value |
   |---|---|---|
   | `ic-frontend-canister-manager` | `package.json` → `name` | the repo name, e.g. `aaa-acme-frontend-manager` |
   | `<CANISTER_NAME>` | `package.json`, `dfx.json`, `canister_ids.json` | the canister name, e.g. `acme_frontend` |
   | `<CANISTER_ID>` | `canister_ids.json` | the existing canister's id — or **delete the file** if the canister does not exist yet; the first `deploy:prod` creates it and writes the file |
   | `www.<YOUR-DOMAIN>.com` | `custom-domain-files/.well-known/ic-domains` | the site's domain(s), one per line |

4. Commit and push.

### B. Wire the web project

Add these three scripts to the web project's `package.json` (replace
`<aaa-client-frontend-manager>` with the manager's folder name):

```json
"copy": "rm -rf ../<aaa-client-frontend-manager>/dist && cp -r dist ../<aaa-client-frontend-manager>/dist",
"craft:deploy": "f() { S=${1:-frontend3}; npm run build && npm run copy && cd ../<aaa-client-frontend-manager> && npm run deploy:test -- $S; }; f",
"prod:deploy": "npm run build && npm run copy && cd ../<aaa-client-frontend-manager> && npm run deploy:prod"
```

* `copy` is the only place the web project touches the manager: it replaces
  the manager's `./dist` with the fresh build.
* `craft:deploy` is the test deploy. `frontend3` is the site's **default
  slot** in the crafting table — this is the one place the default lives.
  `npm run craft:deploy -- frontend2` uses another slot for one run.
* `prod:deploy` is the production deploy.

### C. Test deploy

```
npm run craft:deploy
```

The build lands in `~/motoko-crafting-table/frontend3` and is deployed to that
slot's canister — open `https://<slot canister id>.icp0.io` (ids are in
`~/motoko-crafting-table/canister_ids.json`). The slots are **shared between
all projects** and every deploy is a reinstall: check what a slot holds
(`~/motoko-crafting-table/<slot>/index.html`) before you take it, and expect
your build to be replaced by the next project. `frontend3` has no custom
domain, it is served only from its `.icp0.io` URL.

### D. First production deploy

```
npm run prod:deploy
```

Without `canister_ids.json` this creates the canister on mainnet (costs
cycles — check `npm run dfx:cycles-balance` first) and writes the file:
**commit it**. Later runs redeploy into the same canister.

### E. Custom domain

Point the domain at the canister: DNS records, IC registration, verification —
all in [DOMAIN_SETUP.md](./DOMAIN_SETUP.md). The
`custom-domain-files/` you filled in step A are what `deploy:prod` ships next
to the build so the domain keeps working after every deploy.

---

## Scripts

### In the manager

| Script | What it does |
|---|---|
| `deploy:test -- <slot>` | `scripts/copy-to-crafting-table.sh <slot>` (copies `./dist` into `~/motoko-crafting-table/<slot>`, replacing it), then `npm run deploy:<slot>` in the crafting table. The slot is required and must be an `assets` canister in the table's `dfx.json`, otherwise the script stops before deleting anything. |
| `deploy:prod` | `task:copy-files` (custom-domain files into `./dist`), then `dfx deploy <CANISTER_NAME> --ic`. |
| `task:copy-to-crafting-table -- <slot>` | the copy step of `deploy:test` alone. |
| `task:copy-files` | the custom-domain copy step of `deploy:prod` alone. |
| `dfx:icp-wallet-address`, `dfx:icp-balance`, `dfx:cycles-balance`, `dfx:convert-icp-to-cycles -- <amount>` | ledger and cycles helpers for the current `dfx` identity. |

`./dist` is git-ignored: it is a staging area the web project overwrites on
every deploy, never a source of truth.

### In the web project

| Script | What it does |
|---|---|
| `copy` | replace the manager's `./dist` with this project's `dist`. |
| `craft:deploy [-- <slot>]` | build, copy, manager `deploy:test -- <slot>`; default slot set in the script. |
| `prod:deploy` | build, copy, manager `deploy:prod` (or `<slug>:deploy:prod` in a multi-site manager). |

---

## Several sites in one manager

When a client has more than one site, they share one manager instead of one
repo each. The test side does not change (`deploy:test` and the crafting-table
slots are shared anyway). The production side is namespaced by a short site
slug, so a deploy can never target the wrong site:

* `dist-<slug>/` — the site's production source, filled from `./dist` by
  `<slug>:task:copy-dist`; add it to `.gitignore`.
* `<slug>-custom-domain-files/` — the site's `.ic-assets.json5` and
  `.well-known/ic-domains`.
* one canister per site in `dfx.json` and `canister_ids.json`.
* three scripts per site in `package.json`.

```json
"canisters": {
  "<slug_a>_frontend": {
    "frontend": { "entrypoint": "dist-<slug-a>/index.html" },
    "source": ["dist-<slug-a>"],
    "type": "assets"
  }
}
```

```json
"<slug-a>:task:copy-dist": "rm -rf dist-<slug-a> && cp -r dist dist-<slug-a>",
"<slug-a>:task:copy-files": "cp -r <slug-a>-custom-domain-files/. dist-<slug-a>/",
"<slug-a>:deploy:prod": "npm run <slug-a>:task:copy-dist && npm run <slug-a>:task:copy-files && dfx deploy <slug_a>_frontend --ic"
```

The web project's scripts are the ones from [step B](#b-wire-the-web-project);
only `prod:deploy` ends with `npm run <slug-a>:deploy:prod`. A web project
never touches `dist-<slug>`.

Working example: [`aaa-gulf-frontend-manager`](https://github.com/JSM-Gulf/aaa-gulf-frontend-manager)
— two unrelated sites, one manager.

### Adding a site

1. **Web project:** builds into `dist/`, lives next to the manager, gets the
   three scripts from [step B](#b-wire-the-web-project) with
   `prod:deploy` → `npm run <slug>:deploy:prod`. Pick its default slot.
2. **Manager:** `<slug>-custom-domain-files/` (copy an existing site's, edit
   `ic-domains`), the canister in `dfx.json`, the three `<slug>:*` scripts,
   `dist-<slug>` in `.gitignore`. Commit.
3. **Test:** `npm run craft:deploy` in the web project.
4. **Production:** `npm run prod:deploy` in the web project — the first run
   creates the canister (cycles) and adds it to `canister_ids.json`: commit.
5. **Domain:** [DOMAIN_SETUP.md](./DOMAIN_SETUP.md) with the site's own
   `<slug>-custom-domain-files/` and canister.

Turning a single-site manager into a multi-site one is the same list applied
to the existing site first: rename `custom-domain-files/` and the scripts to
the slug form, switch `dfx.json` to `dist-<slug>`, and add
`<slug>:task:copy-dist`.

---

## Checklist before the first production deploy

* no `<…>` placeholder left in `package.json`, `dfx.json`,
  `canister_ids.json` (or the file deleted), `ic-domains`;
* the manager sits next to the web project and the web project's scripts
  point at its folder name;
* `npm run craft:deploy` works from the web project;
* the `dfx` identity has cycles for a new canister (`npm run dfx:cycles-balance`).
