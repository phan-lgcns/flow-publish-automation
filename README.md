# Boomi Flow Automation Pipeline

This repository contains Jenkins pipeline scripts and configuration to automate the full Boomi Flow lifecycle:
1. **Import Flows** via Flow sharing package tokens.
2. **Refresh Connectors** by updating service elements in Boomi Flow.
3. **Update Flow Identity Providers (IdP)** for master flows and any connected subflows discovered via the Boomi Flow Graph API.
4. **Publish Master & Subflows** by activating the latest snapshot for all subflows first, followed by the master flow.

---

## 📁 Repository Structure

- **`Jenkinsfile`**: End-to-end Jenkins declarative pipeline executing the 4 automation stages.
- **`config/config.json`**: Central configuration containing tenant settings, tokens, connector definitions, IdP ID, and flow IDs.
- **`import_flow_command.txt`**: Reference curl command for flow import API.
- **`update_idp.txt`**: Reference curl command for flow IdP update API.
- **`get_flow_info.txt`**: Reference curl command for flow graph API (`/api/draw/2/graph/flow/{id}`).
- **`gotten_flow_info.txt`**: Sample graph API response illustrating map elements and subflows.

---

## ⚙️ Configuration Properties (`config/config.json`)

```json
{
  "tenantId": "85c2ac30-08cd-48d1-baf8-d66e05b9cb29",
  "flowBaseUrl": "https://us.flow-prod.boomi.com",
  "importTokens": [
    "wCcXd5HqLeY9MhFkvj6WFDbywLO6APYY7ECjMJU2/wP8m2vsU0CW6JunD8No0vD8"
  ],
  "overwriteExisting": false,
  "identityProviderId": "b9acfc4d-c9b0-43d6-a827-55139d365fb8",
  "masterFlowId": "43dedc28-aadc-44bc-a254-61402a2db5f7",
  "flowIds": [
    "43dedc28-aadc-44bc-a254-61402a2db5f7"
  ],
  "connectors": [
    {
      "id": "bbb6a4c7-c0a8-4323-a4ec-292001ed27fe",
      "uri": "https://mizuho-dev.boomi.cloud/fs/RegisterBeneficiary",
      "developerName": "Register Beneficiary Service",
      "developerSummary": null,
      "flowUsername": "mizuhobankltd-ECNYC6.V4O7OK",
      "flowPassword": "da028fc8-01a7-468e-b5ed-3d44438b50a8"
    }
  ]
}
```

---

## 🚀 Pipeline Execution & Stages

The pipeline executes the following 4 stages sequentially:

```
[Import Flows] ➔ [Refresh Connectors] ➔ [Update Flow Identity Providers (incl. Subflows)] ➔ [Publish Master & Subflows]
```

### 1. Import Flows
Imports flows from `config.importTokens` via `POST /api/package/1/shared/flow`.

### 2. Refresh Connectors
Re-installs and refreshes service connectors in `config.connectors` via `POST /api/draw/1/element/service/install`.

### 3. Update Flow Identity Providers
- Traverses the flow graph (`GET /api/draw/2/graph/flow/{id}`) starting from the root flow(s) to discover all referenced subflows (including nested subflows).
- For each subflow and master flow, retrieves the flow model (`GET /api/draw/1/flow/{id}`), updates `identityProvider.id = targetIdpId`, and saves via `POST /api/draw/1/flow`.

### 4. Publish Master & Subflows
- Traverses the graph to find all subflows attached to the master flow.
- Publishes and activates each subflow's latest snapshot first.
- Publishes and activates the master flow's latest snapshot as default and outputs the runnable Play URL.

 