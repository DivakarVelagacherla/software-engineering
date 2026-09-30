# Monitoring and Alerting

## The One-Line Summary

> Monitoring and alerting gives you visibility into your system so problems are detected automatically before users notice. Three pillars: Logs, Metrics, Traces.
> 

---

## The Three Pillars of Observability

### 1. Logs

Detailed text records of what happened.

```
2024-01-15 03:12:45 ERROR PaymentService - Stripe timeout after 5000ms
```

- **Good for:** Debugging, understanding exactly what happened
- **Bad for:** Spotting trends, aggregating across millions of lines

### 2. Metrics

Numerical measurements collected over time, stored in a time series database.

```
payment_error_rate: 0.1% → 0.2% → 5% → 15% → 45%
```

- **Good for:** Spotting trends, triggering alerts
- **Bad for:** Understanding why — tells you something is wrong, not what

### 3. Traces

Follows one request end-to-end across multiple services.

```
User request → API Gateway (2ms) → Order Service (5ms) → Payment Service (2800ms) → Stripe timeout
```

- **Good for:** Finding where latency comes from in distributed systems
- **Bad for:** High volume — can't trace every request at scale

### How They Work Together

- **Metrics** alert you something is wrong
- **Traces** show you where in the system it happened
- **Logs** tell you what happened and why

**When you get a 3am alert:** Traces first to find the bottleneck → Logs to understand the failure.

---

## Latency Percentiles — p50, p95, p99

Never use average latency — it hides outliers.

**Example:** 99 requests at 10ms, 1 request at 10,000ms

- Average: ~110ms — looks fine
- p99: 10,000ms — one user waiting 10 seconds

| Percentile | Meaning |
| --- | --- |
| **p50** | 50% of requests faster than this — typical user experience |
| **p95** | 95% of requests faster than this — most users' experience |
| **p99** | 99% of requests faster than this — worst-case for 1 in 100 users |

Always monitor p99. Average latency will never surface the worst user experiences.

---

## Alerting Best Practices

**Alert on symptoms, not causes**

Alert when users are affected (5% error rate) not when a server has high CPU (might not affect users).

**Set meaningful thresholds**

Don't alert on 0.1% error rate — noise. Alert on 5% sustained for 5 minutes.

**Severity levels**

- **P1** — page immediately, wake someone up
- **P2** — notify during business hours
- **P3** — create a ticket, fix next sprint

**Runbooks**

Every alert links to a runbook — documented steps to diagnose and fix the issue. Without runbooks, every engineer re-investigates from scratch every time.

**Avoid alert fatigue**

Too many alerts = engineers ignore them. Every alert must be actionable.

---

## Tool Landscape

### Logs

| Tool | Type | Notes |
| --- | --- | --- |
| Splunk | Managed | Powerful search, enterprise standard, expensive |
| ELK Stack | Self-managed | Elasticsearch + Logstash + Kibana, open source |
| CloudWatch Logs | AWS managed | Native AWS, free tier |
| Datadog Logs | Managed | Unified with metrics and traces |
| Grafana Loki | Self-managed | Lightweight, pairs with Grafana |

### Metrics

| Tool | Type | Notes |
| --- | --- | --- |
| Prometheus | Self-managed | Industry standard, open source |
| Grafana | Self-managed | Visualization layer on top of Prometheus |
| CloudWatch Metrics | AWS managed | Native AWS infrastructure metrics |
| Datadog Metrics | Managed | Unified platform, auto-discovery |
| New Relic | Managed | Full observability platform |

### Traces

| Tool | Type | Notes |
| --- | --- | --- |
| AWS X-Ray | AWS managed | Native AWS, integrates with Lambda, ECS, API Gateway |
| Jaeger | Self-managed | Open source, CNCF project, industry standard |
| Zipkin | Self-managed | Open source, simpler than Jaeger |
| Datadog APM | Managed | Unified with logs and metrics |
| Grafana Tempo | Self-managed | Pairs with Grafana |

### Alerting + On-Call

| Tool | Notes |
| --- | --- |
| PagerDuty | Industry standard — on-call schedules, escalation policies, runbook links |
| OpsGenie | Atlassian product, similar to PagerDuty |
| CloudWatch Alarms | Native AWS alerting, routes to SNS |
| Grafana Alerting | Built into Grafana, flexible conditions |

### All-in-One Platforms

| Tool | Notes |
| --- | --- |
| Datadog | Most popular managed platform, expensive but powerful |
| New Relic | Full observability, competitive with Datadog |
| Grafana Stack | Prometheus + Loki + Tempo + Grafana, open source |
| AWS Native | CloudWatch + X-Ray, free tier, AWS only |

---

## Standard AWS Production Stack

```
CloudWatch Metric → threshold breached → CloudWatch Alarm
→ SNS Topic → PagerDuty → pages on-call engineer
```

**Grafana + CloudWatch:** Grafana connects to CloudWatch as a data source. Best-in-class dashboards on top of AWS metrics. Common even when CloudWatch is the primary metrics store.

---

## Three-Question Ritual

**What problem does monitoring and alerting solve?**

Unknown failures. Without it, you find out your system is broken when users complain. With it, you detect problems automatically — often before users notice — and have the data to diagnose and fix quickly.

**What breaks without the three pillars?**

Blind to both what and why. Metrics tell you something is wrong. Logs tell you what happened. Traces tell you where. Missing any one leaves a gap in diagnosing problems.

**When NOT to invest heavily in observability?**

Early stage, low traffic, small team. CloudWatch native tools are sufficient and essentially free at small scale. Invest in full observability as complexity and traffic grow.

---

## AWS Equivalents (Vanguard Stack)

| Pillar | Tool |
| --- | --- |
| Logs | CloudWatch Logs + Splunk |
| Metrics | CloudWatch Metrics |
| Traces | AWS X-Ray |
| Alerting | CloudWatch Alarms → SNS → PagerDuty |