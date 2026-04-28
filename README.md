# IC Frontend Canister Manager
This project provides a streamlined workflow for deploying frontend assets to the Internet Computer (IC) while ensuring that custom domain configurations and SSL certificates remain valid during the deployment process.

## Configuration Guide

* Clone this project
* Create branch for your web project
* Follow below instructions:

This project requires replacing placeholder values with environment-specific configuration before deployment. Follow the instructions below carefully.

---

## 1a. `package.json` Configuration

Update the following placeholders in the `package.json` file:

* `<MOTOKO_CRAFTING_TABLE_PATH>`
  Replace with the absolute or relative path to your Motoko project directory that contains the deployment scripts.

* `<CANISTER_NAME>`
  Replace with the exact name of the canister you intend to deploy.

### Example

```json
"deploy:test": "cd ./path-to-your-motoko-project && npm run deploy:frontend1",
"deploy:prod": "npm run task:copy-files && dfx deploy your_canister_name --ic"
```

---

## 1b `package.json` Configuration in your web project

Add this two scripts in your web project (pay attention to folder hierarchy):

```json
"prod:copy": "rm -rf ../ic-frontend-canister-manager/dist && cp -r dist ../ic-frontend-canister-manager/dist",
"prod:deploy": "npm run build && npm run prod:copy && cd ../ic-frontend-canister-manager && npm run deploy:prod"
```

---

## 2. Domain Configuration (`ic-domains` file)

Update the domain placeholder:

* `www.<YOUR-DOMAIN>.com`
  Replace `<YOUR-DOMAIN>` with your actual domain name.

---

## 3. Notes

* Ensure all placeholders are replaced before running any deployment commands.
* Incorrect or missing values may result in failed deployments or misconfigured environments.
* The `deploy:prod` script assumes that required files are present in the `custom-domain-files` directory.

---

## 4. Deployment Commands

* Test deployment:
  ```
  npm run deploy:test
  ```

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
