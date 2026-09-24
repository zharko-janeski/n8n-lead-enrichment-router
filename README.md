<div align="center">

[![n8n](https://img.shields.io/badge/n8n-Workflow%20Automation-EA4B71?logo=n8n\&logoColor=white)](https://n8n.io/)
[![Apollo.io](https://img.shields.io/badge/Apollo.io-Lead%20Enrichment-2D6CDF?logoColor=white)](https://www.apollo.io/)
[![Slack](https://img.shields.io/badge/Slack-Notifications-4A154B?logo=slack\&logoColor=white)](https://slack.com/)
[![Airtable](https://img.shields.io/badge/Airtable-CRM-18BFFF?logo=airtable\&logoColor=white)](https://www.airtable.com/)
[![SendGrid](https://img.shields.io/badge/SendGrid-Email-1A82E2?logo=sendgrid\&logoColor=white)](https://sendgrid.com/)
[![Discord](https://img.shields.io/badge/Discord-Error%20Monitoring-5865F2?logo=discord\&logoColor=white)](https://discord.com/)

</div>

# 🎯 Lead Enrichment & Smart Router - n8n Workflow

<div align="center">
  <img src="workflow.gif" alt="Lead Enrichment & Smart Router workflow demo" width="800">
</div>

An end-to-end **RevOps automation** built with **n8n**.

A lead submits a form → the workflow enriches the company via the **Apollo.io API** → a **Switch** node routes the lead by company size.

* 🏢 **Enterprise leads (≥ 200 employees)** → Slack alert + Enterprise CRM record in Airtable
* 🌱 **SMB leads** → SMB drip record in Airtable
* ✉️ **Every lead** → Personalized email via SendGrid

If the enrichment API fails, **no lead is lost**. The failure is logged to Discord, safe fallback values are applied, and the lead continues through the SMB path.

> **Errors are handled, never fatal.**

---

## 🔄 Architecture


![Full workflow canvas in n8n](canvas.png)


```mermaid
flowchart TD
    A["1 · Webhook: Lead Form"] --> B["2 · Edit Fields: Normalize Input"]
    B --> C["3 · HTTP: Enrich via Apollo"]

    C -- "🟢 Success" --> D["6 · Switch: Lead Router"]
    C -- "🔴 Error" --> E["4 · HTTP: Log to Discord"]

    E --> F["5 · Edit Fields: Fallback Defaults"]
    F --> D

    D -- "Enterprise ≥ 200 employees" --> G["7 · Slack: Enterprise Alert"]
    D -- "Fallback / SMB" --> H["9 · Airtable: Add SMB to Drip"]

    G --> I["8 · Airtable: Create Enterprise Deal"]

    I --> J["10 · SendGrid: Personalized Welcome Email"]
    H --> J
```

### Flow Overview

The workflow has **two entry paths** into the Switch:

1. Normal enrichment data
2. Error-recovery data

The Switch creates two business paths:

* **Enterprise**
* **SMB / Fallback**

Both paths converge again into the final SendGrid email node.

**Every possible outcome ends with the lead processed and contacted.**

---

# 🧩 Node-by-Node

| #  | Node                                 | Type         | Job                                                                                |
| -- | ------------------------------------ | ------------ | ---------------------------------------------------------------------------------- |
| 1  | Webhook: Lead Form                   | Webhook      | Receives `name`, `email`, `company`, and `domain`                                  |
| 2  | Edit Fields: Normalize Input         | Edit Fields  | Cleans input, extracts first name, normalizes email, and derives domain if missing |
| 3  | HTTP: Enrich via Apollo              | HTTP Request | Calls Apollo Organizations Enrich API for employee count and industry              |
| 4  | HTTP: Log to Discord                 | HTTP Request | Sends enrichment failure alerts to Discord                                         |
| 5  | Edit Fields: Fallback Defaults       | Edit Fields  | Applies safe fallback values when enrichment fails                                 |
| 6  | Switch: Lead Router                  | Switch       | Routes by employee count                                                           |
| 7  | Slack: Enterprise Alert              | Slack        | Sends a hot-lead alert to the sales channel                                        |
| 8  | Airtable: Create Enterprise Deal     | Airtable     | Creates an Enterprise CRM record                                                   |
| 9  | Airtable: Add SMB to Drip            | Airtable     | Creates an SMB drip record                                                         |
| 10 | SendGrid: Personalized Welcome Email | SendGrid     | Sends a personalized email to every lead                                           |

---

# ✨ Design Decisions Worth Noting

## 1. Error-output branching — graceful degradation

The Apollo node uses n8n's:

**On Error → Continue (using error output)**

This gives the node two possible exits:

```text
🟢 Success → Switch

🔴 Error → Discord → Fallback Defaults → Switch
```

If Apollo fails, the workflow does **not** terminate.

Instead:

1. The failure is logged to Discord
2. Safe fallback values are applied
3. The lead continues through normal routing
4. The lead eventually receives the welcome email

This prevents an enrichment failure from becoming a lead-processing failure.

---

## 2. Retry with backoff for transient failures

The Apollo HTTP request is configured with:

* **2 retries**
* **3-second wait**
* **10-second timeout**

This gives the workflow a chance to recover from temporary network or provider failures before entering the error branch.

---

# 🚀 Setup

## Prerequisites

* n8n — self-hosted or cloud
* Apollo.io account
* Discord server/channel
* Slack workspace
* Airtable account
* SendGrid account

---

## 🔐 Credentials

Create the following credentials in n8n:

| Service       | Setup                                                                                                   |
| ------------- | ------------------------------------------------------------------------------------------------------- |
| **Apollo.io** | Create an API key and configure access for the enrichment endpoint                                      |
| **Discord**   | Create a channel webhook and add the URL to the Discord HTTP node                                       |
| **Slack**     | Create a Slack app, configure messaging permissions, install it to the workspace, and add the bot token |
| **Airtable**  | Create a Personal Access Token scoped to the required base                                              |
| **SendGrid**  | Create an API key and configure a verified sender                                                       |

### Apollo.io

Create an API key through Apollo's API/integration settings.

The workflow uses the Organizations Enrich endpoint.

The HTTP node sends the API key using:

```text
X-Api-Key: <YOUR_APOLLO_API_KEY>
```

### Discord

Create a webhook for the target Discord channel and paste the webhook URL into:

```text
HTTP: Log to Discord
```

### Slack

Create a Slack app and provide the required permissions for sending messages.

Install the app into your workspace and provide the bot token to n8n.

### Airtable

Create a base:

```text
n8n Lead Router
```

Create a table:

```text
Leads
```

Create a Personal Access Token with access limited to the required base.

### SendGrid

Configure a verified sender and create a SendGrid API key.

---

# 🗃️ Airtable Schema

Table: `Leads`

| Field     | Type                                           |
| --------- | ---------------------------------------------- |
| Name      | Single line text                               |
| Email     | Single line text                               |
| Company   | Single line text                               |
| Industry  | Single line text                               |
| Employees | Number (integer)                               |
| Tier      | Single select: `Enterprise` / `SMB`            |
| Status    | Single select: `Deal Created` / `Drip Started` |

---

# 📥 Import Workflow

1. Open n8n
2. Go to **Workflows → Import from File**
3. Import:

```text
Lead Enrichment & Smart Router.json
```

4. Re-create the required credentials
5. Open the Discord HTTP node
6. Replace the Discord webhook placeholder
7. Verify the Airtable base and table
8. Verify the Slack channel
9. Verify the SendGrid sender
10. Activate the workflow

The production webhook becomes:

```text
POST /webhook/lead-form
```
# 🧪 Testing

No external form is needed to test the workflow — the webhook can be simulated directly with PowerShell:

![Simulating the webhook with PowerShell](input-simulate-wehook-lead-incoming-viapowershell.png)


# 📊 Expected Results

| Test       | Slack | Discord | Airtable        | Tier       | Email |
| ---------- | ----: | ------: | --------------- | ---------- | ----: |
| Enterprise |     ✅ |       — | Enterprise Deal | Enterprise |     ✅ |
| SMB        |     — |       — | SMB Drip        | SMB        |     ✅ |
| Error      |     — |       ✅ | SMB Drip        | SMB        |     ✅ |

For successful enrichment, **Industry** and **Employees** come from Apollo.

For failed enrichment, the workflow uses the configured fallback values.

**Discord alert received after simulating an enrichment failure. To trigger it, temporarily add `xx` to the Apollo URL:**

![Discord error alert on enrichment failure](discord.png)

### Results in Action

| Slack Alert | Airtable Record | Welcome Email |
| :---: | :---: | :---: |
| ![Slack enterprise alert](slack.png) | ![Airtable lead record](airtable.png) | ![SendGrid welcome email](sendgrid.png) |


---

# ⚠️ Notes & Limitations

### Email Deliverability

Free SendGrid accounts may send emails without full domain authentication.

Test emails can therefore land in spam.

For production, configure proper domain authentication such as **SPF/DKIM**.

### Apollo Usage

Apollo enrichment availability depends on the account and plan.

The workflow retries failed enrichment requests according to the configured retry settings before using the fallback path.

### Enterprise Threshold

The current Enterprise threshold is:

```text
200 employees
```

This value is configured in the Switch node and can be changed according to business requirements.

### Production Improvements

A production implementation could additionally include:

* Webhook signature verification
* Form authentication
* Deduplication by email
* Provider-specific rate-limit handling
* More robust retry strategies
* HubSpot or Salesforce integration
* Persistent error logging
* Observability and monitoring
* Lead scoring
* Human approval before Enterprise CRM creation

---

# 🛠️ Built With

**n8n · Apollo.io · Slack · Airtable · SendGrid · Discord**

---

