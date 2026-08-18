# IC Frontend Canister Manager

This tool is for central management of production deploys of frontend applications. It's a starting point for further customization depending on your project's needs. The minimal configuration handles a single production canister, but for larger projects it can be turned into a full-blown deploy combine, covering multiple environments.

As a rule, deploys to test canisters should go through the `motoko-crafting-table` project. Nothing stops the frontend manager from also managing scripts for deploys to dedicated test canisters, though — motoko-crafting-table keeps reusing the same shared canisters, whereas here you can define your own project-specific test canisters.

This project provides a streamlined workflow for deploying frontend assets to the Internet Computer (IC) while ensuring that custom domain configurations and SSL certificates remain valid during the deployment process.

## Configuration Guide

* Create a new git repository named "aaa-<your-project-name>-frontend-manager", e.g. "aaa-nazwa-projektu-frontend-manager"
* Clone it locally
* Download this `ic-frontend-canister-manager` project as a zip from GitHub and unpack it, or clone it separately, then copy its contents into the repo you just created
* Follow below instructions:

This project requires replacing placeholder values with environment-specific configuration before deployment. Follow the instructions below carefully.

---

## 1a. `package.json` Configuration

Update the following placeholders in the `package.json` file:

* replace "ic-frontend-canister-manager" with your repo's name following the pattern "aaa-<your-project-name>-frontend-manager" i.e.: aaa-nazwa-projektu-frontend-manager

* `<CANISTER_NAME>`
  Replace with the exact name of the canister you intend to deploy.

### Example

```json
"deploy:prod": "npm run task:copy-files && dfx deploy your_canister_name --ic"
```
---

## 1b. `canister_ids.json` and `dfx.json` Configuration

* If your project doesn't have a canister created yet, delete `canister_ids.json`.
* If your project already has a canister — or you plan to reuse an existing one — keep `canister_ids.json` and replace `<CANISTER_ID>` with that canister's id.
* In both `canister_ids.json` (if kept) and `dfx.json`, replace `<CANISTER_NAME>` with the name of your canister.

---

## 1c `package.json` Configuration in your web project

Add this two scripts in your web project (pay attention to folder hierarchy and don't forget to replace <aaa-project-name> with your actual folder name):

```json
"prod:copy": "rm -rf ../<aaa-project-name>/dist && cp -r dist ../<aaa-project-name>/dist",
"prod:deploy": "npm run build && npm run prod:copy && cd ../<aaa-project-name> && npm run deploy:prod"
```

---

## 2. Domain Configuration (`ic-domains` file)

Update the domain placeholder:

* `www.<YOUR-DOMAIN>.com`
  Replace `<YOUR-DOMAIN>` with your actual domain name.

For the full custom-domain walkthrough — DNS records, apex vs. subdomain, IC registration API, verification — see **[DOMAIN_SETUP.md](./DOMAIN_SETUP.md)**.

---

## 3. Notes

* Ensure all placeholders are replaced before running any deployment commands.
* Incorrect or missing values may result in failed deployments or misconfigured environments.
* The `deploy:prod` script assumes that required files are present in the `custom-domain-files` directory.

---

## 4. Deployment Commands

* Production deployment:
  ```
  npm run deploy:prod
  ```

---

## 5. Summary

Before proceeding, verify that:

* All placeholders have been replaced with valid values.
* Paths are correct and accessible.
* Domain configuration matches your intended production setup.

Failure to properly configure these values may prevent successful deployment.

---

## 6. Managing Multiple Projects From One Manager

The minimal setup above assumes one canister, one `dist/`, one
`custom-domain-files/`. When several independent frontend projects should
share a single deploy manager (e.g. a family of related sites for the same
client), namespace everything by a short project slug instead of running
one manager per project. Concrete working example:
`aaa-gulf-frontend-manager`, which manages two unrelated sites
(`gulf-enterprise-solutions` and `gulfbusinesssupport`) from one repo.

### 6a. Per-project directories, not shared ones

* `dist/` → `dist-<project-slug>/`, one per project.
* `custom-domain-files/` → `<project-slug>-custom-domain-files/`, one per
  project, each with its own `.ic-assets.json5` and
  `.well-known/ic-domains`.

### 6b. `dfx.json` / `canister_ids.json` — one canister entry per project

```json
{
  "canisters": {
    "<project_a>_frontend": {
      "frontend": { "entrypoint": "dist-<project-a-slug>/index.html" },
      "source": ["dist-<project-a-slug>"],
      "type": "assets"
    },
    "<project_b>_frontend": {
      "frontend": { "entrypoint": "dist-<project-b-slug>/index.html" },
      "source": ["dist-<project-b-slug>"],
      "type": "assets"
    }
  },
  "version": 1
}
```

```json
{
  "<project_a>_frontend": { "ic": "<CANISTER_ID_A>" },
  "<project_b>_frontend": { "ic": "<CANISTER_ID_B>" }
}
```

### 6c. `package.json` — namespace every script by project slug

```json
"scripts": {
  "<project-a-slug>:task:copy-files": "cp -r <project-a-slug>-custom-domain-files/. dist-<project-a-slug>/",
  "<project-a-slug>:deploy:prod": "npm run <project-a-slug>:task:copy-files && dfx deploy <project_a>_frontend --ic",

  "<project-b-slug>:task:copy-files": "cp -r <project-b-slug>-custom-domain-files/. dist-<project-b-slug>/",
  "<project-b-slug>:deploy:prod": "npm run <project-b-slug>:task:copy-files && dfx deploy <project_b>_frontend --ic"
}
```

There's no single unqualified `deploy:prod` anymore — every command names
the project it acts on, so a deploy can never accidentally target the
wrong site.

### 6d. Each web project's own `package.json` still points at its own dist

Same pattern as [§1c](#1c-packagejson-configuration-in-your-web-project),
just pointed at the project-specific folder inside the manager:

```json
"prod:copy": "rm -rf ../aaa-<client>-frontend-manager/dist-<project-slug> && cp -r dist ../aaa-<client>-frontend-manager/dist-<project-slug>",
"prod:deploy": "npm run build && npm run prod:copy && cd ../aaa-<client>-frontend-manager && npm run <project-slug>:deploy:prod"
```

### 6e. Adding a new project to an existing multi-project manager

1. Create `dist-<new-slug>/` (or let the web project's `prod:copy` create it).
2. Create `<new-slug>-custom-domain-files/` with `.ic-assets.json5` +
   `.well-known/ic-domains` (copy an existing project's, then replace the
   domain — see [DOMAIN_SETUP.md](./DOMAIN_SETUP.md)).
3. Add the new canister to `dfx.json` (and `canister_ids.json` once it
   exists on mainnet).
4. Add the `<new-slug>:task:copy-files` / `<new-slug>:deploy:prod` script
   pair to `package.json`.
5. First `<new-slug>:deploy:prod` run creates the canister on mainnet if it
   doesn't already exist in `canister_ids.json` — this costs cycles.

### 6f. Custom domains stay per-project

Everything in [DOMAIN_SETUP.md](./DOMAIN_SETUP.md) applies unchanged per
project — just substitute the project's own `<project-slug>-custom-domain-files/`
directory and canister for the generic `custom-domain-files/` paths used
there. Each project registers its own domain independently through the
same `custom-domains/v1` API; nothing is shared between projects except
this one repo and its `dfx.json`/`package.json`.
