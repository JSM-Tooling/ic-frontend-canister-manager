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
