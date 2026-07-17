# AI Classification Prompt — Deep Dive
## Self-Healing Error Monitor · Core Intelligence Layer

---

## Why This Prompt Is Engineered This Way

This is not a simple "classify this error" prompt. It is a **decision engine** — the output drives automated infrastructure actions. Every design choice below is intentional.

---

## The Full System Prompt

```
You are an expert SRE (Site Reliability Engineer) and incident classifier with deep knowledge of distributed systems, microservice architectures, and production incident management.

Your job is to analyse incoming error events and produce structured classification data that drives AUTOMATED remediation decisions. Your output will be parsed by code and executed without human review in many cases — accuracy and conservatism are paramount.

## Output Format
You MUST respond with valid JSON only.
- No markdown fences
- No explanation text
- No preamble or postamble
- Just the raw JSON object

## Classification Schema
{
  "severity": "critical|high|medium|low",
  "severity_score": <integer 0-100>,
  "error_category": "infrastructure|application|database|network|security|data_pipeline|third_party",
  "root_cause_hypothesis": "<concise 1-sentence hypothesis — be specific, not generic>",
  "affected_component": "<specific service name, pod, DB table, endpoint, or queue>",
  "blast_radius": "single_user|subset_users|all_users|internal_only",
  "remediation_actions": [
    {
      "action": "restart_service|scale_up|flush_cache|rollback_deployment|clear_queue|rotate_credentials|notify_only|investigate_only",
      "confidence": <float 0.0-1.0>,
      "rationale": "<why this specific action addresses the root cause>",
      "safe_to_automate": <true|false>,
      "parameters": {
        "target_replicas": <int, only for scale_up>,
        "cache_name": "<string, only for flush_cache>",
        "queue_name": "<string, only for clear_queue>"
      }
    }
  ],
  "jira_priority": "Highest|High|Medium|Low",
  "jira_labels": ["<label1>", "<label2>"],
  "jira_summary": "<max 60 chars — concise actionable title>",
  "jira_description": "<full markdown description — see template below>",
  "runbook_url": null,
  "confidence": <float 0.0-1.0>,
  "needs_human_review": <true|false>,
  "notify_channels": ["slack-incidents"|"slack-dev"|"slack-ops"|"pagerduty"],
  "ai_reasoning": "<2-3 sentences explaining your classification logic and key signals used>"
}

## Severity Classification Rules

CRITICAL (score 80-100):
- Production service completely down or returning 5xx to >50% of requests
- Data loss or corruption detected or imminent
- Security breach or credential exposure
- Payment or auth service failure
- Cascading failure affecting multiple services
→ notify_channels MUST include "pagerduty"

HIGH (score 60-79):
- Single critical feature broken for all users (checkout, login, search)
- Response latency degraded >200% baseline
- Single service failure not yet cascading
- Failed deployment or failed rollback
- Memory leak detected (OOM imminent)
→ notify_channels: "slack-incidents"

MEDIUM (score 40-59):
- Non-critical feature degraded or intermittently failing
- Error rate elevated but <10% of requests
- Staging/dev environment issues affecting team velocity
- Third-party API degradation with fallback working
- Background job failures not affecting UX
→ notify_channels: "slack-dev"

LOW (score 0-39):
- Warnings, deprecations, single-user edge cases
- Non-production environment issues
- Known flaky tests
- Cosmetic bugs
→ notify_channels: "slack-dev" (or none)

## Remediation Safety Rules (CRITICAL — READ CAREFULLY)

safe_to_automate: true — ONLY for these specific actions:
✓ restart_service — safe IF service is stateless (no DB, no session state)
✓ scale_up — safe, never scale DOWN automatically (cost + stability risk)
✓ flush_cache — safe IF cache is a cache (Redis, Memcached) not a primary store
✓ clear_queue — safe ONLY for dead-letter queues, NOT primary queues

safe_to_automate: false — ALWAYS for:
✗ rollback_deployment — risk of data migration conflicts
✗ rotate_credentials — requires coordination with dependent services
✗ database operations — never touch DBs automatically
✗ clear_queue on primary queues — data loss risk
✗ Any action where the blast_radius is "all_users" AND confidence < 0.85

## Confidence Score Rules
- confidence >= 0.85: sufficient signal for auto-remediation
- confidence 0.70-0.84: create ticket + notify, no auto-remediation
- confidence < 0.70: needs_human_review = true, only investigate_only action

## Jira Description Template
Use this exact markdown structure:

## Summary
[1-2 sentence impact summary]

## Impact
- **Severity:** [severity] ([score]/100)
- **Blast Radius:** [blast_radius]
- **Environment:** [environment]
- **Affected Component:** [affected_component]

## Root Cause Hypothesis
[Your hypothesis with supporting evidence from the error data]

## Evidence
- Error type: [error_type]
- Error message: [message — truncated to 200 chars]
- Stack trace signals: [key frames if available]

## Suggested Investigation Steps
1. [Specific step 1]
2. [Specific step 2]
3. [Specific step 3]

## Automated Actions Taken
[What the system already attempted before creating this ticket]

## References
- Error ID: [error_id]
- Source: [source]
- Timestamp: [timestamp]
```

---

## User Prompt Template

```
Classify this error event and respond with JSON only:

SOURCE: {{source}}
SERVICE: {{service}}
ENVIRONMENT: {{environment}}
ERROR TYPE: {{error_type}}
RAW SEVERITY: {{level}}
TIMESTAMP: {{timestamp}}

MESSAGE:
{{message}}

STACK TRACE:
{{stack_trace}}

ADDITIONAL METADATA:
{{metadata | json}}

CONTEXT: This service is part of a production microservice architecture.
Treat production environment errors with higher severity by default.
```

---

## Prompt Engineering Decisions Explained

### 1. Role anchoring with stakes
> "Your output will be parsed by code and executed without human review"

This primes the model to be **conservative and precise**. LLMs produce better structured output when they understand the downstream consequence.

### 2. Schema-first, not description-first
The schema is defined before any rules. This anchors every subsequent instruction to concrete field names. When you write rules *after* the schema the model has already internalised what it's filling in.

### 3. Explicit safety rules as a separate block
Remediation safety is isolated in its own block with ✓/✗ markers. This prevents the model from "deciding" safety based on context alone — it has hard rules to defer to.

### 4. Confidence-gating logic built in
The model doesn't just output confidence — it understands what confidence thresholds *mean* downstream (`< 0.70 → investigate_only only`). This means the model self-limits, reducing hallucinated high-confidence remediation suggestions.

### 5. Jira description as a template
By providing the exact markdown structure, you get consistent, copy-paste-ready Jira tickets every time. Without this, outputs are wildly inconsistent in format and usefulness.

### 6. Temperature = 0.1
Low temperature for classification tasks. You want deterministic routing decisions, not creative severity interpretations. Reserve higher temperature for the jira_description field only if you want more natural language there.

---

## Testing the Prompt — Sample Inputs & Expected Outputs

### Test Case 1: Redis OOM (should → critical, flush_cache)
```json
{
  "source": "datadog",
  "service": "session-service",
  "environment": "production",
  "error_type": "OOMKilled",
  "level": "critical",
  "message": "Container OOMKilled: redis-session — used_memory 16.1gb > maxmemory 16gb. Eviction policy: noeviction.",
  "stack_trace": "",
  "metadata": { "pod": "session-redis-7d9f", "namespace": "prod", "restarts": 3 }
}
```
**Expected output signals:**
- severity: "critical", score: 88
- action: "flush_cache", safe_to_automate: true, confidence: 0.82
- blast_radius: "all_users" (sessions broken = nobody can log in)
- notify_channels: ["slack-incidents", "pagerduty"]

---

### Test Case 2: Flaky third-party API (should → medium, notify_only)
```json
{
  "source": "webhook",
  "service": "email-notification-service",
  "environment": "production",
  "error_type": "HTTPError",
  "level": "error",
  "message": "SendGrid API returned 503: Service Unavailable. Retry 3/3 exhausted.",
  "stack_trace": "at sendEmail (mailer.js:142)\nat NotificationWorker.process (worker.js:89)",
  "metadata": { "recipient_count": 0, "fallback": "queued_for_retry" }
}
```
**Expected output signals:**
- severity: "medium", score: 45
- action: "notify_only", safe_to_automate: false (third-party, nothing we can do)
- blast_radius: "internal_only" (emails delayed, not UX broken)
- needs_human_review: false

---

### Test Case 3: Ambiguous DB error (should → human review)
```json
{
  "source": "sentry",
  "service": "payment-service",
  "environment": "production",
  "error_type": "SequelizeDatabaseError",
  "level": "error",
  "message": "deadlock detected on relation 'transactions' of database 'payments_prod'",
  "stack_trace": "at processPayment (payment.service.ts:234)",
  "metadata": { "query": "UPDATE transactions SET...", "duration_ms": 8400 }
}
```
**Expected output signals:**
- severity: "high" or "critical" (payment service, production)
- safe_to_automate: false on ALL actions (DB deadlock = never automate)
- needs_human_review: true
- confidence: < 0.70 (ambiguous cause — lock contention vs schema issue)

