# Setup Guide — Self-Healing Error Monitor
## n8n Import + Configuration Checklist

---

## Step 1: Import the Workflow

1. Open n8n → **Workflows** → **Import from file**
2. Select `self-healing-monitor.json`
3. The workflow loads with all 16 nodes pre-wired

---

## Step 2: Set Environment Variables

In n8n → **Settings** → **Variables**, create these:

| Variable | Example Value | Notes |
|---|---|---|
| `LOG_API_URL` | `https://logs.yourcompany.com` | Your log ingestion endpoint |
| `LOG_API_KEY` | `sk-...` | Bearer token for log API |
| `INFRA_API_URL` | `https://infra-api.yourcompany.com` | Restart/scale API |
| `QUEUE_API_URL` | `https://queue.yourcompany.com` | RabbitMQ/SQS API |
| `SLACK_INCIDENTS_CHANNEL` | `C0123456789` | Slack channel ID |
| `GOOGLE_SHEET_ID` | `1BxiM...` | Audit log Sheet ID |
| `JIRA_PROJECT_ID` | `ENG` | Jira project key |
| `PAGERDUTY_API_URL` | `https://api.pagerduty.com` | PD API base |
| `PAGERDUTY_TOKEN` | `y_NbY...` | PD API token |
| `PAGERDUTY_FROM_EMAIL` | `alerts@you.com` | PD from address |
| `PAGERDUTY_SERVICE_ID` | `P1234AB` | PD service ID |

---

## Step 3: Connect Credentials

Go to **Credentials** in n8n and create:

- **OpenAI API** — API key from platform.openai.com
- **Jira API** — OAuth2 or API token (Atlassian account)
- **Slack API** — OAuth2 (Bot token with `chat:write` scope)
- **Google Sheets OAuth2** — Google Cloud OAuth2 credentials

Assign each credential to the corresponding node.

---

## Step 4: For Demo — Mock Infra API

If you don't have a real infra API, deploy this Express stub:

```javascript
// infra-stub.js
const express = require('express');
const app = express();
app.use(express.json());

app.post('/services/:name/restart', (req, res) => {
  console.log(`[MOCK] Restarting ${req.params.name}`);
  setTimeout(() => res.json({ status: 'ok', service: req.params.name, action: 'restarted' }), 1200);
});

app.patch('/services/:name/scale', (req, res) => {
  res.json({ status: 'ok', service: req.params.name, replicas: req.body.replicas });
});

app.delete('/cache/:name/flush', (req, res) => {
  res.json({ status: 'ok', cache: req.params.name, keys_flushed: 4821 });
});

app.get('/health/:service', (req, res) => {
  // Always return healthy after 10s (simulates recovery)
  res.json({ status: 'healthy', service: req.params.service, uptime_seconds: 23 });
});

app.listen(3001, () => console.log('Mock infra API running on :3001'));
```

Run with `node infra-stub.js` and set `INFRA_API_URL=http://localhost:3001`

---

## Step 5: Google Sheets Audit Log Setup

Create a Sheet with these columns in row 1:

```
error_id | timestamp | processed_at | source | service | environment | severity | severity_score | error_category | message | root_cause_hypothesis | blast_radius | ai_confidence | needs_human_review | remediation_attempted | remediation_action | remediation_result | jira_created | jira_key | ai_reasoning | logged_at
```

---

## Step 6: Test Payloads

**Critical — triggers auto-remediation:**
```bash
curl -X POST http://localhost:5678/webhook/error-ingest \
  -H "Content-Type: application/json" \
  -d '{"source":"webhook","service":"session-service","environment":"production","error_type":"OOMKilled","level":"critical","message":"Redis OOMKilled: used_memory 16.1gb > maxmemory 16gb","stack_trace":"","metadata":{"pod":"session-redis-7d9f","restarts":3}}'
```

**High — triggers Jira ticket only:**
```bash
curl -X POST http://localhost:5678/webhook/error-ingest \
  -H "Content-Type: application/json" \
  -d '{"source":"sentry","service":"checkout-service","environment":"production","error_type":"TypeError","level":"error","message":"Cannot read properties of undefined (reading stripe_token)","stack_trace":"at processPayment (checkout.js:234)\nat POST /api/checkout (router.js:89)","metadata":{"user_id":"usr_8f2a","order_id":"ord_99b1"}}'
```

**Ambiguous — triggers human review:**
```bash
curl -X POST http://localhost:5678/webhook/error-ingest \
  -H "Content-Type: application/json" \
  -d '{"source":"webhook","service":"payment-service","environment":"production","error_type":"SequelizeDatabaseError","level":"error","message":"deadlock detected on relation transactions","stack_trace":"at processPayment (payment.service.ts:234)","metadata":{"query":"UPDATE transactions SET status=completed WHERE id=...","duration_ms":8400}}'
```

