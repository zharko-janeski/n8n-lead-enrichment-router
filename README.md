🎯 Lead Enrichment & Smart Router — n8n Workflow
An end-to-end RevOps automation built with n8n.

A lead submits a form → the workflow enriches the company via the Apollo.io API → a Switch node routes by company size: enterprise leads (≥ 200 employees) trigger a Slack alert + a high-tier CRM record in Airtable, SMB leads get a drip record. Every lead receives a personalized email via SendGrid.

If the enrichment API fails, no lead is lost: the failure is logged to a Discord channel, safe fallback values are stamped, and the lead continues down the SMB path. Errors are handled, never fatal.

Portfolio project demonstrating: external API enrichment, credential management, conditional routing (Switch), error-output branching, retries with backoff, cross-node data references, multi-path convergence, and graceful degradation.

🔄 Architecture
flowchart TD    A["1 · Webhook: Lead Form"] --> B["2 · Edit Fields: Normalize Input"]    B --> C["3 · HTTP: Enrich via Apollo"]    C -- "🟢 Success" --> D["6 · Switch: Lead Router"]    C -- "🔴 Error" --> E["4 · HTTP: Log to Discord"]    E --> F["5 · Edit Fields: Fallback Defaults"]    F --> D    D -- "Enterprise (≥ 200 employees)" --> G["7 · Slack: Enterprise Alert"]    D -- "Fallback (SMB / no data)" --> H["9 · Airtable: Add SMB to Drip"]    G --> I["8 · Airtable: Create Enterprise Deal"]    I --> J["10 · SendGrid: Personalized Welcome Email"]    H --> J
Two entry paths into the Switch (normal data + error-recovery data), two exit paths (Enterprise / SMB), converging again into one email node. Every possible outcome ends with the lead processed and contacted.

🧩 Node-by-Node
#	Node	Type	Job
1	Webhook: Lead Form	Webhook	Receives form POST: name, email, company, domain
2	Edit Fields: Normalize Input	Edit Fields (Set)	Cleans input: extracts first name, trims/lowercases email, derives domain from email if missing
3	HTTP: Enrich via Apollo	HTTP Request	POST to Apollo organizations/enrich → employee count, industry. Retry: 2× / 3s · Timeout: 10s · On Error → dedicated error output
4	HTTP: Log to Discord	HTTP Request	Posts 🚨 failure alert to Discord. Best-effort: continue-on-error so a monitoring outage can never block leads
5	Edit Fields: Fallback Defaults	Edit Fields (Set)	Stamps safe defaults: 0 employees, unknown industry, enrichmentStatus: failed
6	Switch: Lead Router	Switch	Routes on employee count: ≥ 200 → Enterprise, everything else → Fallback (SMB)
7	Slack: Enterprise Alert	Slack	Formatted 🏆 hot-lead alert to a sales channel
8	Airtable: Create Enterprise Deal	Airtable	CRM record · Tier = Enterprise · Status = Deal Created
9	Airtable: Add SMB to Drip	Airtable	CRM record · Tier = SMB · Status = Drip Started
10	SendGrid: Personalized Welcome Email	SendGrid	Personalized email — merges both paths, so every lead gets a reply
✨ Design Decisions Worth Noting
1. Error-output branching (graceful degradation)
The Apollo node uses n8n's On Error → Continue (using error output) setting, which gives it two exits: 🟢 success and 🔴 error. On failure the item flows: Discord alert → default values → normal routing. The workflow never dies and never drops a lead — the exact question interviewers ask.

2. Retry with backoff for transient failures
Before erroring, the Apollo node retries 2× with 3s wait and a 10s timeout. Transient blips (rate limits, network hiccups) self-heal; only genuine failures reach the error branch.

3. Safe-default routing
Failed enrichment is stamped with 0 employees → routes to SMB. Logic: better to send a big company an automated email than to falsely promise an enterprise rep. Failing "down" is always safe.

4. One expression, two data shapes
The Switch reads:{{ $json.organization?.estimated_num_employees ?? $json.estimated_num_employees ?? 0 }}This handles both the nested Apollo response (organization.estimated_num_employees) and the flat fallback fields — one rule covering both entry paths.

5. Cross-node references
Intermediate nodes (Slack, Airtable) return their own response data, overwriting the lead fields in the item. Downstream nodes reach back by name: $('Edit Fields: Normalize Input').first().json.email — keeping lead data available at every step regardless of what nodes in between returned.

6. Least-privilege credentials
Every credential is scoped to the minimum: Slack bot with only chat:write scopes, Airtable PAT limited to one base and write-only records, Apollo key scoped to a single endpoint. All secrets live in n8n's encrypted credential store — the exported workflow JSON contains no credentials (the Discord webhook URL is replaced with a placeholder in this repo).

🚀 Setup
Prerequisites
n8n (self-hosted or cloud)
Credentials to create in n8n
Service	How
Apollo.io	Free account → Settings → Integrations → API → create key scoped to organizations/enrich → in n8n: Header Auth credential, name X-Api-Key
Discord	Channel → Edit → Integrations → Webhooks → copy URL → paste directly into the HTTP node
Slack	api.slack.com/apps → Blank app → Bot Token Scopes: chat:write + chat:write.public → Install to Workspace → paste xoxb- token
Airtable	Create base n8n Lead Router, table Leads → Developer Hub → Personal Access Token with data.records:write + schema.bases:read, scoped to that base
SendGrid	Settings → Sender Authentication → Verify a Single Sender → create API key (Full Access)
Airtable table schema (Leads)
Field	Type
Name	Single line text
Email	Single line text
Company	Single line text
Industry	Single line text
Employees	Number (integer)
Tier	Single select: Enterprise / SMB
Status	Single select: Deal Created / Drip Started
Import
n8n → Workflows → Import from File → Lead Enrichment & Smart Router.json
Re-create the 5 credentials (they are intentionally not in the export)
Open the Discord node → paste your webhook URL
Activate the workflow → production URL becomes POST /webhook/lead-form
🧪 Testing
Click Execute workflow in n8n (test webhook listens once per click), then trigger:

Enterprise path — real company, ≥ 200 employees:

Invoke-RestMethod -Uri "http://localhost:5678/webhook-test/lead-form" -Method Post -ContentType "application/json" -Body '{"name":"Ana Smith","email":"ana@shopify.com","company":"Shopify","domain":"shopify.com"}'
SMB path — small company (enriches, but under threshold):

Invoke-RestMethod -Uri "http://localhost:5678/webhook-test/lead-form" -Method Post -ContentType "application/json" -Body '{"name":"John Doe","email":"john@techstartup.io","company":"Tech Startup","domain":"techstartup.io"}'
Error path — unknown domain (Apollo fails → Discord alert → fallback → SMB):

Invoke-RestMethod -Uri "http://localhost:5678/webhook-test/lead-form" -Method Post -ContentType "application/json" -Body '{"name":"Walt Small","email":"walt@waltsmallcoxx.example","company":"Walt Small Co","domain":"waltsmallcoxx.example"}'
Expected results:

Test	Slack 🏆	Discord 🚨	Airtable Tier	Email
Enterprise (shopify)	✅	—	Enterprise	industry from Apollo
SMB (techstartup)	—	—	SMB	industry from Apollo or "your industry"
Error (garbled domain)	—	✅	SMB	"your industry" fallback
⚠️ Notes & Limitations
Email deliverability: free SendGrid accounts send "via sendgrid.net" without domain authentication (SPF/DKIM), so test emails typically land in spam. Fine for a demo; production setups authenticate their domain.
Apollo free tier: ~75 enrichment credits. The workflow burns credits only on successful lookups; failures retry, then fall back.
Enterprise threshold (200 employees) is one number in the Switch rule — tune per market.
In production you'd add: form authentication / webhook signature verification, dedup by email before Airtable create, rate-limit handling per provider, and a real CRM (HubSpot/Salesforce) behind the same node pattern.
🛠️ Built With
n8n · Apollo.io · Slack · Airtable · SendGrid · Discord