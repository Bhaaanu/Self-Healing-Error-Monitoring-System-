<div align="center">

# 🛡️ Self-Healing Error Monitor
### Autonomous Incident Detection · AI Classification · Auto-Remediation · Zero-Touch Recovery

![n8n](https://img.shields.io/badge/n8n-workflow-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)
![OpenAI](https://img.shields.io/badge/OpenAI-GPT--4o-412991?style=for-the-badge&logo=openai&logoColor=white)
![Slack](https://img.shields.io/badge/Slack-notifications-4A154B?style=for-the-badge&logo=slack&logoColor=white)
![Jira](https://img.shields.io/badge/Jira-auto--tickets-0052CC?style=for-the-badge&logo=jira&logoColor=white)
![PagerDuty](https://img.shields.io/badge/PagerDuty-escalation-06AC38?style=for-the-badge&logo=pagerduty&logoColor=white)

**A production-grade n8n workflow that detects, classifies, and autonomously resolves infrastructure incidents — before most engineers open their laptops.**

[View Workflow JSON](./self-healing-monitor.json) · [AI Prompt Design](./AI-CLASSIFICATION-PROMPT.md) · [Branch Deep Dive](./BRANCH-DEEP-DIVE.md) · [Setup Guide](./SETUP-GUIDE.md)

</div>

---

## What This Does

Most monitoring setups alert humans and stop there. This system goes further:

| Stage | What Happens |
|---|---|
| **Ingest** | Receives errors from Webhooks, Sentry, Datadog, PagerDuty, or any log API |
| **Normalise** | Unifies all source formats into a single schema regardless of origin |
| **Classify** | GPT-4o analyses the error, assigns severity, hypothesises root cause, and scores confidence |
| **Route** | Switch node branches into Critical / High / Medium / Low tracks |
| **Remediate** | For critical issues, executes the safest high-confidence action automatically |
| **Verify** | Polls a health endpoint 10 seconds post-remediation |
| **Self-heal** | Posts "🟢 Self-Healed in 34s" to Slack if recovered — zero human involvement |
| **Escalate** | Fires PagerDuty + enriched Slack alert only when auto-remediation fails |
| **Ticket** | Creates fully-populated Jira tickets for High severity with investigation steps pre-written |
| **Audit** | Logs every event, decision, and outcome to Google Sheets for trend analysis |
| **Digest** | Posts a weekly Slack summary: incidents resolved, MTTR, top failing services |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         INGESTION LAYER                             │
│  Webhook (apps)  │  Cron+HTTP (log APIs)  │  Sentry  │  Datadog    │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Normalise Schema   │  ← Unified event format
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  AI: Classify Error │  ← GPT-4o · temp 0.1
                    │  severity · cause   │
                    │  actions · confidence│
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Severity Router   │  ← Switch node
                    └────┬────┬────┬──────┘
                         │    │    │
              ┌──────────┘    │    └───────────┐
              │               │                │
     ┌────────▼───────┐  ┌────▼─────┐  ┌──────▼──────┐
     │  CRITICAL track│  │HIGH track│  │  MED/LOW    │
     │                │  │          │  │  Log + Trend│
     │ Pick Action    │  │Jira Auto │  └─────────────┘
     │ IF: Automate?  │  │  Ticket  │
     │ Execute Infra  │  └──────────┘
     │ Wait 10s       │
     │ Health Check   │
     │                │
     ├── HEALTHY ─────┼──→ Slack: 🟢 Self-Healed
     │                │
     └── UNHEALTHY ───┼──→ Slack: 🔴 Escalation
                      │         │
                      │         └──→ PagerDuty Incident
                      │
              ┌───────▼───────┐
              │  Audit Logger │  ← Every path logs here
              │ Google Sheets │
              └───────────────┘
                      │
              ┌───────▼───────┐
              │  Weekly Cron  │  ← Every Monday 09:00
              │ Digest → Slack│
              └───────────────┘
```

---

## Workflow Files

```
📁 self-healing-error-monitor/
├── 📄 self-healing-monitor.json      # n8n workflow — import directly
├── 📄 weekly-digest.json             # Weekly digest branch — import separately
├── 📄 AI-CLASSIFICATION-PROMPT.md   # Full prompt design + engineering rationale
├── 📄 BRANCH-DEEP-DIVE.md           # Remediation + Jira branch walkthrough
├── 📄 SETUP-GUIDE.md                # Credentials, env vars, test payloads
└── 📄 README.md                     # You are here
```

---

## Quick Start

### 1. Import into n8n

```bash
# Self-hosted n8n
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n

# Then open http://localhost:5678
# Workflows → Import from file → select self-healing-monitor.json
```

### 2. Configure Environment Variables

In n8n → Settings → Variables:

```env
LOG_API_URL=https://logs.yourcompany.com
LOG_API_KEY=your-log-api-key
INFRA_API_URL=https://infra-api.yourcompany.com
QUEUE_API_URL=https://queue.yourcompany.com
SLACK_INCIDENTS_CHANNEL=C0123456789
GOOGLE_SHEET_ID=your-sheet-id
JIRA_PROJECT_ID=ENG
PAGERDUTY_API_URL=https://api.pagerduty.com
PAGERDUTY_TOKEN=your-pd-token
PAGERDUTY_FROM_EMAIL=alerts@yourcompany.com
PAGERDUTY_SERVICE_ID=PXXXXXX
```

### 3. Wire Credentials

| Node | Credential Type |
|---|---|
| AI: Classify Error | OpenAI API Key |
| Jira: Create Ticket | Jira OAuth2 |
| Slack nodes | Slack OAuth2 Bot Token |
| Sheets: Audit Log | Google Sheets OAuth2 |

### 4. Test It

```bash
# Critical — triggers full auto-remediation loop
curl -X POST http://localhost:5678/webhook/error-ingest \
  -H "Content-Type: application/json" \
  -d '{
    "source": "webhook",
    "service": "session-service",
    "environment": "production",
    "error_type": "OOMKilled",
    "level": "critical",
    "message": "Redis OOMKilled: used_memory 16.1gb > maxmemory 16gb. Eviction: noeviction",
    "metadata": { "pod": "session-redis-7d9f", "restarts": 3 }
  }'

# High — triggers Jira ticket creation only
curl -X POST http://localhost:5678/webhook/error-ingest \
  -H "Content-Type: application/json" \
  -d '{
    "source": "sentry",
    "service": "checkout-service",
    "environment": "production",
    "error_type": "TypeError",
    "level": "error",
    "message": "Cannot read properties of undefined (reading stripe_token)",
    "stack_trace": "at processPayment (checkout.js:234)\nat POST /api/checkout (router.js:89)"
  }'

# Ambiguous — triggers human review (no auto-action)
curl -X POST http://localhost:5678/webhook/error-ingest \
  -H "Content-Type: application/json" \
  -d '{
    "source": "webhook",
    "service": "payment-service",
    "environment": "production",
    "error_type": "SequelizeDatabaseError",
    "level": "error",
    "message": "deadlock detected on relation transactions",
    "metadata": { "query": "UPDATE transactions SET status=completed", "duration_ms": 8400 }
  }'
```

---

## AI Classification Engine

The core intelligence is a **purpose-engineered GPT-4o prompt** designed as a decision engine, not a chatbot.

Key design decisions:

**Conservative by design** — The prompt tells the model its output drives automated infrastructure actions with no human review. This primes it for precision over creativity.

**Confidence-gated automation** — Actions are only executed when `confidence >= 0.7`. Below that, the system routes to human review regardless of severity.

**Explicit safety rules** — `safe_to_automate: true` is only permitted for: service restarts (stateless), cache flushes, scale-up, dead-letter queue clears. Database operations, rollbacks, and credential rotation are always `false`.

**Structured remediation array** — The AI outputs a ranked list of actions with individual confidence scores. The workflow picks the highest-confidence safe action, not the first in the list.

See [AI-CLASSIFICATION-PROMPT.md](./AI-CLASSIFICATION-PROMPT.md) for the full prompt, rationale, and test cases.

---

## Supported Remediation Actions

| Action | Trigger Condition | Safe to Automate | Notes |
|---|---|---|---|
| `restart_service` | OOM, crash loop, unresponsive | ✅ Yes (stateless only) | Calls infra restart API |
| `scale_up` | High load, memory pressure | ✅ Yes | Never auto-scales down |
| `flush_cache` | Cache corruption, OOM | ✅ Yes (cache layer only) | Not primary stores |
| `clear_queue` | Dead-letter queue overflow | ✅ Yes (DLQ only) | Not primary queues |
| `rollback_deployment` | Bad deploy detected | ❌ No | Migration conflict risk |
| `rotate_credentials` | Credential exposure | ❌ No | Requires coordination |
| `investigate_only` | Ambiguous / low confidence | ❌ No | Always the safe fallback |

---

## Weekly Digest — Sample Output

Every Monday at 09:00, a digest is posted to your incidents channel:

```
📊 Weekly Incident Digest — w/c 11 Aug 2025

🟢 Auto-resolved:     12 incidents  (71%)
🟡 Human-escalated:    4 incidents  (24%)
⚪ Logged only:         1 incident   (5%)

⚡ Mean Time to Recovery (auto): 41s
📋 Jira tickets created: 4

🔥 Top Failing Services
  1. session-service    — 5 incidents  (Redis OOM recurring)
  2. payment-service    — 3 incidents  (DB deadlock intermittent)
  3. email-service      — 2 incidents  (SendGrid 3rd party)

📌 Highest Severity: payment-service deadlock [CRITICAL · 88/100]
   → Escalated to human · PD-4821

View full audit log → [Google Sheets]
```

---

## Integrations

| Tool | Purpose | Node Type |
|---|---|---|
| **Webhook** | Receive app error events | n8n Webhook Trigger |
| **Cron + HTTP** | Poll log APIs every 5 min | Schedule Trigger + HTTP Request |
| **Sentry** | Error event ingestion | HTTP Request (Sentry API) |
| **Datadog** | Metric anomaly ingestion | HTTP Request (Datadog API) |
| **OpenAI GPT-4o** | Error classification + Jira writing | OpenAI node (LangChain) |
| **Jira Cloud** | Auto-ticket creation | Jira node (OAuth2) |
| **Slack** | Incident + digest notifications | Slack node (OAuth2) |
| **PagerDuty** | Human escalation | HTTP Request (PD API) |
| **Google Sheets** | Audit trail + digest source data | Google Sheets node |
| **Infra API** | Execute restart/scale/flush | HTTP Request (configurable) |

---

## Design Principles

**1. Graceful degradation** — Every branch has a fallback. If the AI parse fails, the event still gets routed (conservatively) and logged. If remediation fails, it escalates. Nothing falls silently.

**2. Explainability over black-box** — Every AI decision is logged with `ai_reasoning`. Every automated action is attributed with `triggered_by: n8n-monitor` in the infra API call. Full audit trail, always.

**3. Conservative automation** — The system asks "is it safe to act?" before "should I act?". Confidence thresholds, safety flags, and blast-radius checks all gate the automation independently.

**4. Schema-first normalisation** — All four ingestion sources are normalised before AI classification. This means the AI prompt never needs to handle format variations — it always receives the same clean structure.

**5. Closed-loop verification** — Auto-remediation is meaningless without verification. The health check 10 seconds post-action is what makes this "self-healing" rather than just "auto-acting".

---

## Built With

- [n8n](https://n8n.io) — Workflow automation platform
- [OpenAI GPT-4o](https://platform.openai.com) — AI classification engine
- [Jira Cloud API](https://developer.atlassian.com/cloud/jira/platform/rest/v3/) — Issue management
- [Slack API](https://api.slack.com) — Notifications
- [PagerDuty API](https://developer.pagerduty.com) — On-call escalation
- [Google Sheets API](https://developers.google.com/sheets/api) — Audit logging

---

## Author

Built as a proof-of-work demonstrating production-grade automation and AI infrastructure engineering — not a tutorial project, but a system designed to run in real environments with real consequences.

---

<div align="center">

**If this saved you from a 2am incident, consider starring the repo ⭐**

</div>
