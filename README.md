# Boomi Automation Pipeline

This repository contains multi-pipeline Jenkins scripts and modular configuration files to automate Boomi Flow and Integration processes.

---

## 📁 Repository Structure

### ⚙️ Jenkins Pipelines
- **`Jenkinsfile.connector.refresh`**: Installs/refreshes Boomi Flow connectors in the specified tenant environment.
- **`Jenkinsfile.flow.publish`**: Publishes the latest snapshot version of specified Boomi Flows.
- **`Jenkinsfile.integration.deployment`**: Deploys Boomi Integration process packages to a target Boomi Atom/Environment. (Automatically skips execution if no component parameters are provided).

### 📄 Modular Configurations (`config/`)
- **`config/config.connector.json`**: Config for connector installation/refresh.
- **`config/config.flow.publish.json`**: Config for flow publishing.
- **`config/config.integration.deployment.json`**: Config for integration deployments.

---

## ⚙️ Configuration Files & Properties

### 1. `config/config.connector.json`
```json
{
  "tenantId": "85c2ac30-08cd-48d1-baf8-d66e05b9cb29",
  "flowBaseUrl": "https://us.flow-prod.boomi.com",
  "flowUsername": "mizuhobankltd-ECNYC6.V4O7OK",
  "flowPassword": "da028fc8-01a7-468e-b5ed-3d44438b50a8",
  "connectors": [
    {
      "id": "bbb6a4c7-c0a8-4323-a4ec-292001ed27fe",
      "uri": "https://mizuho-dev.boomi.cloud/fs/RegisterBeneficiary",
      "developerName": "Register Beneficiary Service",
      "developerSummary": null
    }
  ]
}
```

### 2. `config/config.flow.publish.json`
```json
{
  "tenantId": "85c2ac30-08cd-48d1-baf8-d66e05b9cb29",
  "flowBaseUrl": "https://us.flow-prod.boomi.com",
  "flowIds": [
    "43dedc28-aadc-44bc-a254-61402a2db5f7"
  ]
}
```

### 3. `config/config.integration.deployment.json`
```json
{
  "boomiAccountId": "mizuhobankltd-ECNYC6",
  "environmentName": "MIZUHO_DEV_MCS",
  "componentNames": [],
  "packageVersion": ""
}
```
> **Note**: If `componentNames` and `packageVersion` are left empty (and no parameters are passed when triggering the Jenkins job), the `Jenkinsfile.integration.deployment` pipeline will safely skip execution without failing.

---

## 🔑 Jenkins Requirements & Credentials

1. **Required Jenkins Plugins**:
   - **Pipeline Utility Steps Plugin**: Provides `readJSON`.
   - **HTTP Request Plugin**: Provides `httpRequest`.

2. **Required Credentials**:
   - **`boomi-flow-api-key`** (*Secret Text*): API Key for Boomi Flow access.
   - **`boomi-platform-credentials`** (*Username with Password*): Boomi Platform user credentials for REST API interaction.

---

## 🚀 Execution & Setup in Jenkins

To run 3 independent pipelines on every Git push:
1. Create 3 distinct Pipeline Jobs in Jenkins pointing to this Git repository:
   - **Job 1 (Connector Refresh)**: Script Path set to `Jenkinsfile.connector.refresh`
   - **Job 2 (Flow Publish)**: Script Path set to `Jenkinsfile.flow.publish`
   - **Job 3 (Integration Deployment)**: Script Path set to `Jenkinsfile.integration.deployment`
2. Enable **GitHub hook trigger for GPRC polling** (or SCM Webhook) on all 3 jobs.
