# Branch Deep Dive: Remediation + Jira Auto-Ticket
## Self-Healing Error Monitor · n8n Workflow

---

## BRANCH 1: Auto-Remediation Pipeline

### How It Flows

```
Switch (critical) → Code: Pick Action → IF: Can Remediate?
                                              │
                         ┌────────────────────┴────────────────────────┐
                       YES (true)                                    NO (false)
                         │                                               │
                  Code: Execute Remediation                    → Slack: Escalation
                         │                                               │
                  HTTP: Call Infra API                         → PagerDuty Incident
                         │
                  Wait: 10s delay
                         │
                  HTTP: Health Check
                         │
              ┌──────────┴──────────┐
           HEALTHY               UNHEALTHY
              │                      │
     Slack: Self-Healed       Slack: Escalation
                                     │
                              PagerDuty Incident
```

---

### Node 1 — Code: Pick Remediation Action

**What it does:** Selects the single best safe action from the AI's list.

```javascript
const actions = item.classification.remediation_actions || [];

// Filter: safe + not "investigate_only" + confidence threshold
const safeActions = actions
  .filter(a => a.safe_to_automate && a.action !== 'investigate_only')
  .sort((a, b) => b.confidence - a.confidence);

const chosenAction = safeActions[0] || null;

// Gate: must meet ALL three conditions
const willAttempt = (
  !!chosenAction &&
  item.confidence >= 0.7 &&          // AI must be confident
  !item.needs_human_review            // AI must not have flagged for human
);
```

**Why confidence gate at 0.7?** Below 0.7 the model is essentially guessing. A wrong remediation action (e.g. restarting the wrong service, flushing a wrong cache) can make incidents worse. 0.7 is conservative — in practice you may raise this to 0.8 after testing.

**Why filter out `investigate_only`?** That action is always in the list as a safe fallback — if it wasn't excluded, it would always "win" at confidence 1.0 and nothing would ever auto-remediate.

---

### Node 2 — Code: Execute Remediation

**What it does:** Builds the HTTP config for the infra API call dynamically based on action type.

```javascript
const actionMap = {
  restart_service: {
    url: `${INFRA_API_URL}/services/${service}/restart`,
    method: 'POST',
    body: { reason: 'auto-remediation', triggered_by: 'n8n-monitor', error_id }
  },
  scale_up: {
    url: `${INFRA_API_URL}/services/${service}/scale`,
    method: 'PATCH',
    body: { replicas: params.target_replicas || 2 }
  },
  flush_cache: {
    url: `${INFRA_API_URL}/cache/${params.cache_name || service}/flush`,
    method: 'DELETE',
    body: { triggered_by: 'n8n-monitor' }
  },
  clear_queue: {
    url: `${QUEUE_API_URL}/queues/${params.queue_name}/purge`,
    method: 'POST',
    body: { triggered_by: 'n8n-monitor' }
  }
};
```

**Infrastructure API adapters you need to build/connect:**
| Action | AWS | GCP | Kubernetes | Custom |
|--------|-----|-----|------------|--------|
| restart_service | ECS UpdateService | Cloud Run deploy | kubectl rollout restart | Your API |
| scale_up | ECS UpdateService desiredCount | Cloud Run --min-instances | kubectl scale | Your API |
| flush_cache | ElastiCache flush | Memorystore flush | Redis CLI via job | Your API |
| clear_queue | SQS PurgeQueue | Pub/Sub seek | RabbitMQ purge API | Your API |

**Pro tip for demo:** Build a mock `/infra-stub` Express server that accepts these routes and returns `{ status: "ok" }`. This lets you demo the full workflow without real infra.

---

### Node 3 — HTTP: Health Check

**What it does:** Polls a `/health` endpoint 10 seconds after remediation.

**Design the health endpoint to return:**
```json
{
  "status": "healthy",
  "service": "session-service",
  "uptime_seconds": 23,
  "checks": {
    "database": "ok",
    "cache": "ok",
    "dependencies": "ok"
  }
}
```

**What if the service doesn't have a /health endpoint?**
Adapt the health check node to poll any reliable signal:
- HTTP 200 on a known endpoint
- Metric value from Datadog/Prometheus
- Process count from your infra API
- Queue depth back below threshold

---

### Node 4 — Slack: Self-Healed Message Design

```
🟢 Self-Healed | `session-service` | `production`

What happened: Redis OOM due to no eviction policy on session keys
Action taken: `flush_cache` (redis-session)
Recovery time: 34s
AI confidence: 87%

Error ID: `err-8f2a1c` | Source: datadog | View in Datadog →
```

**Why this message works for a portfolio demo:**
- The "Recovery time: 34s" is the money shot — shows end-to-end closed loop timing
- Confidence % shows the system is self-aware about uncertainty
- The error ID creates a traceable audit trail interviewers can follow

---

## BRANCH 2: Jira Auto-Ticket (High Severity)

### Why auto-Jira matters more than people think

Manual Jira creation during incidents is:
- Slow (takes 3-5 min when every second counts)
- Inconsistent (different engineers write different levels of detail)
- Often skipped entirely ("I'll create it after we fix it" → never created)

This branch creates a fully-populated, investigation-ready ticket in under 2 seconds.

---

### The Jira Node Configuration

```javascript
// n8n Jira node - key fields
{
  resource: "issue",
  operation: "create",
  projectId: "{{ $env.JIRA_PROJECT_ID }}",
  summary: "={{ $json.classification.jira_summary }}",   // AI-generated, max 60 chars
  issueType: "Bug",
  priority: { name: "={{ $json.classification.jira_priority }}" },
  labels: "={{ $json.classification.jira_labels }}",
  description: {
    // Jira uses Atlassian Document Format (ADF), not plain markdown
    type: "doc",
    version: 1,
    content: [{
      type: "paragraph",
      content: [{ type: "text", text: "={{ $json.classification.jira_description }}" }]
    }]
  }
}
```

**Critical gotcha:** Jira Cloud uses ADF (Atlassian Document Format), NOT markdown. The n8n Jira node handles basic conversion but if you want rich formatting (headings, code blocks, tables), you need to build the ADF object directly. For a demo, plain text in a paragraph block works fine.

---

### What a Generated Jira Ticket Looks Like

**Summary:** `[HIGH] session-service: Redis OOM — sessions unavailable`

**Priority:** High | **Labels:** `infrastructure`, `redis`, `auto-detected`

**Description:**
```
## Summary
Redis session cache hit memory limit with noeviction policy, causing all new
session writes to fail. Users cannot log in or maintain authenticated sessions.

## Impact
- Severity: HIGH (74/100)
- Blast Radius: all_users
- Environment: production
- Affected Component: redis-session (pod: session-redis-7d9f)

## Root Cause Hypothesis
Redis maxmemory limit reached with noeviction policy configured, blocking all
writes and causing authentication failures across the session service.

## Evidence
- Error type: OOMKilled
- Message: Container OOMKilled: redis-session — used_memory 16.1gb > maxmemory 16gb
- Stack trace signals: N/A (infrastructure-level event)

## Suggested Investigation Steps
1. Check Redis memory usage with `redis-cli info memory` — look at used_memory vs maxmemory
2. Review eviction policy: `redis-cli config get maxmemory-policy` — should be allkeys-lru not noeviction
3. Audit session TTL settings — keys may not be expiring correctly
4. Check for cache key explosion — run `redis-cli --scan --pattern 'session:*' | wc -l`

## Automated Actions Taken
- flush_cache executed at 2025-08-14T09:23:11Z
- Health check passed at 2025-08-14T09:23:45Z (34s recovery)
- This ticket created for post-incident review and permanent fix tracking

## References
- Error ID: err-8f2a1c
- Source: datadog
- Timestamp: 2025-08-14T09:22:37Z
```

This is **dramatically better** than what engineers write manually under pressure.

---

### Post-Ticket Enhancements (show these in your demo)

**1. Link ticket to the Slack thread:**
After Jira ticket creation, use the returned `issue.key` to update the Slack message with a button linking to the ticket.

**2. Auto-assign based on service ownership:**
```javascript
// CODEOWNERS-style map — store in n8n workflow static data
const ownerMap = {
  'session-service': 'U0123SLACK',   // Slack user ID
  'payment-service': 'U0456SLACK',
  'api-gateway': 'U0789SLACK'
};
const assignee = ownerMap[item.service] || null;
```

**3. Add to a sprint automatically:**
Use the Jira `sprint` field to drop the ticket directly into the current sprint — no triage meeting required.

**4. GitHub Actions trigger on Jira creation:**
If jira_labels includes `deployment-related`, trigger a GitHub Actions workflow to prepare a rollback branch. This is the next level of automation most teams have never seen.

---

## MAKING IT STAND OUT IN A DEMO

### The 5-minute demo script

1. **Open n8n** — show the workflow canvas. Let them see the complexity.

2. **Send a test webhook** (Postman/curl):
```bash
curl -X POST https://your-n8n.com/webhook/error-ingest \
  -H "Content-Type: application/json" \
  -d '{
    "source": "webhook",
    "service": "session-service",
    "environment": "production",
    "error_type": "OOMKilled",
    "level": "error",
    "message": "Redis OOMKilled: used_memory 16.1gb > maxmemory 16gb. Eviction: noeviction",
    "metadata": { "pod": "session-redis-7d9f" }
  }'
```

3. **Switch to n8n execution view** — watch nodes light up in real time.

4. **Show Slack** — the self-healed message appears. Point to "Recovery time: 34s".

5. **Show Google Sheets** — the audit row appeared automatically.

6. **Send a different payload** (ambiguous DB error) — show how the system correctly routes to human review instead of auto-remediating. This shows the system knows its limits.

7. **Close with the architecture diagram** (the SVG from earlier) and explain each decision.

**The line that closes the room:**
> "Every other candidate will show you a chatbot. This system fixed a production incident, created the Jira ticket, notified the team, and logged the audit trail — before most engineers have even opened their laptop."

