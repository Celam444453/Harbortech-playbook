# HarborTech Operations Playbook — Week 5
## Riverside Goods Seasonal Promotion: Availability and Capacity Review

**Prepared by:** [Camden], HarborTech Intern
**Client:** Riverside Goods
**Date:** [10/07/26]
**Status:** Draft for recommendation approval

---

## 1. Summary

Riverside Goods runs its inventory application on **one healthy EC2 instance behind one public endpoint**. A seasonal promotion is expected to sharply increase traffic. The instance is healthy today, but "healthy" is not the same as "resilient" or "sized for peak."

**Recommendation:** A **combination** of controls is needed. No single control covers every risk:

| Priority | Control | Risk it solves |
|---|---|---|
| 1 | **Application Load Balancer (ALB)** across at least two Availability Zones | Single endpoint / single instance as a point of failure; unhealthy-target detection |
| 2 | **Auto Scaling group (ASG)** attached to the ALB target group, spanning at least two AZs | Capacity risk during traffic peaks; replacing failed instances |
| 3 | **Route 53** alias to the ALB, with failover routing **only if** a standby site or Region exists | Whole-endpoint or Region-level outage; not a capacity fix |

---

## 2. Evidence Reviewed

> **Note:** The figures below are illustrative sample values used to complete the analysis. Replace them with the actual numbers from the evidence packet if they differ.

| Evidence item | Observed value | What it tells us |
|---|---|---|
| Current request volume | 1,200 requests/min (average), 1,800 requests/min at daily peak | Baseline load the single instance handles today |
| Expected promotion traffic | 5× baseline (about 6,000 requests/min) | Size of the capacity gap |
| CPU utilization in previous peaks | 78% average, 94% peak | How close the instance already is to saturation |
| Architecture | 1 EC2 instance, 1 public endpoint | No redundancy at the compute or entry layer |
| Target-health expectations | Path `/health`, 30-second interval, 2 passes = healthy, 3 failures = unhealthy | Defines what "healthy" means for routing and replacement |
| Auto Scaling capacity range | min 2 / desired 2 / max 6 | Whether the range can absorb the promotion |
| Proposed Route 53 failover design | Primary = current instance IP; secondary = S3 maintenance page; 30-second health check; TTL 300 seconds | Whether DNS failover addresses the real risks |

---

## 3. Risk Analysis

### 3.1 Single points of failure

| # | Single point of failure | Impact if it fails | Likelihood during promotion |
|---|---|---|---|
| SPOF-1 | The one EC2 instance | Full application outage; no other instance to take traffic | Rises with load and CPU pressure |
| SPOF-2 | One Availability Zone (if the instance runs in one) | AZ-level event takes the app offline | Low, but high impact |
| SPOF-3 | One public endpoint (single IP / DNS name tied to the instance) | No way to route around a failed instance without manual DNS or IP changes | Tied to SPOF-1 |
| SPOF-4 | No automatic replacement mechanism | Recovery depends on a person noticing and acting | Increases mean time to recover |

### 3.2 Capacity risks

| # | Capacity risk | Evidence to cite |
|---|---|---|
| CAP-1 | Peak CPU on previous events was already 94%, leaving little headroom | CPU utilization data |
| CAP-2 | Promotion traffic of 5× baseline exceeds what one instance can serve | Request volume vs. instance capability |
| CAP-3 | Scaling by hand (resizing or launching instances) is too slow for a sharp spike | Single-instance architecture |
| CAP-4 | The Auto Scaling range of 2–6 may be too narrow or too slow to react | Auto Scaling capacity range |

### 3.3 Detection and response risks

- A health check that only tests "the instance is running" can miss an application that is up but failing. The health check path must exercise the application.
- Without target-health checks wired to a load balancer or ASG, a failing instance keeps receiving traffic or is never replaced.

---

## 4. Which Control Solves Which Risk

| Risk | Load balancing (ALB) | Auto Scaling | Route 53 failover |
|---|---|---|---|
| Single instance / single endpoint (SPOF-1, SPOF-3) | **Yes.** Distributes traffic across multiple targets behind one stable DNS name | Supports it by keeping the number of healthy instances at the minimum | Partial. Only helps if a separate standby exists |
| Single AZ (SPOF-2) | **Yes**, when enabled in two or more AZs | **Yes**, when the ASG spans multiple AZs | Partial |
| Unhealthy instance still receiving traffic | **Yes.** Stops routing to targets that fail health checks | Replaces them when ELB health checks are enabled | Not for individual instances |
| Traffic spike exceeds one instance (CAP-1 to CAP-3) | Spreads load but **does not add capacity** | **Yes.** Adds instances based on scaling policies | **No.** DNS failover adds no capacity |
| Slow manual recovery (SPOF-4) | Detects failure | **Yes.** Launches replacements automatically | Detects endpoint failure, does not repair it |
| Entire endpoint or Region unavailable | No (ALB is Regional) | No | **Yes**, if a secondary endpoint is ready |

**Key conclusion:** Load balancing and Auto Scaling together address the identified risks. Route 53 failover is a **disaster-recovery layer**, not a substitute for either, and it does nothing for a traffic spike.

---

## 5. Assessment of the Proposed Route 53 Failover Design

Route 53 failover routing sends traffic to a secondary resource when the primary fails its health check. For this case:

1. **It does not add capacity.** If the primary is overloaded but still passing health checks, DNS will not shift traffic.
2. **It needs somewhere to fail over to.** A secondary pointing at the same single instance protects nothing. The secondary must be an independent endpoint (another ALB/Region, or a static maintenance page in S3 as a degraded-mode fallback).
3. **DNS caching delays recovery.** Failover time is bounded by health-check interval, failure threshold, and record TTL. A long TTL can leave users on a dead endpoint.
4. **Better primary target:** Point the primary record at the **ALB** (Route 53 alias record) rather than at an instance IP.

**Verdict:** Keep Route 53 in the design only as the entry point (alias to the ALB) and as optional failover to a real standby. Do not rely on it as the answer to the promotion risk.

---

## 6. Recommended Architecture

```
Users
  │
Route 53 (alias record → ALB; optional failover to standby)
  │
Application Load Balancer (subnets in ≥ 2 AZs)
  │  health checks on application path
Target group
  ├── EC2 instance (AZ-a) ┐
  └── EC2 instance (AZ-b) ├── Auto Scaling group
          (more as needed) ┘   min ≥ 2, max sized for peak
```

**Configuration expectations**

- ASG **minimum of 2** instances in different AZs so one failure never means zero capacity.
- ASG **maximum** sized from peak load testing with margin (not left at a default).
- ASG health check type set to **ELB**, so unhealthy instances are replaced, not just removed from rotation.
- Target tracking policy on average CPU (for example around 50–60%) and, if available, requests per target.
- Health check path that verifies the application and its dependencies appropriately, with sensible interval and thresholds.
- Warm-up and cooldown values set so scaling reacts before saturation.

---

## 7. Verification Checklist (before approval)

### Capacity
- [ ] Load test reaches or exceeds the expected promotion traffic of 5×.
- [ ] CPU, memory, latency, and error rate stay within agreed limits during the test.
- [ ] ASG max capacity covers peak with headroom.
- [ ] Instance launch-to-healthy time is measured and is fast enough for the ramp rate.
- [ ] Account service quotas (EC2 vCPU limits, ALB limits) allow the max capacity.
- [ ] Downstream dependencies (database, APIs) can handle the added load.

### Availability
- [ ] ALB and ASG both span at least two AZs.
- [ ] Terminating one instance causes no user-visible outage in a test.
- [ ] Simulating an AZ loss still leaves enough capacity.
- [ ] Health check path and thresholds confirmed with the application owner.
- [ ] Security groups allow ALB-to-instance traffic on the health check port.

### Application readiness
- [ ] Application is stateless, or session state is stored outside the instance (sticky sessions understood if used).
- [ ] Launch template/AMI produces a ready instance with no manual steps.
- [ ] Logs and metrics are centralized so scaled-in instances do not lose data.

### Route 53 (if failover is kept)
- [ ] Primary record is an alias to the ALB.
- [ ] Secondary target is independent and tested.
- [ ] Health check, failure threshold, and TTL give an acceptable failover time.
- [ ] Failover and failback were exercised in a test.

### Operations
- [ ] CloudWatch alarms for CPU, 5xx rate, unhealthy host count, and ASG at max capacity.
- [ ] On-call owner and escalation path assigned for the promotion window.
- [ ] Rollback plan documented.

---

## 8. Evidence Gaps and Open Questions

| Question | Why it matters | Owner |
|---|---|---|
| What AZ is the current instance in? | Confirms SPOF-2 | Riverside Goods application owner |
| Is the application stateless? | Determines whether scaling out is safe | Riverside Goods application owner |
| What is the database's capacity? | The database can become the next bottleneck | Riverside Goods application owner |
| What is the Route 53 secondary target? | Determines whether failover has value | Riverside Goods application owner |
| What is the actual max Auto Scaling capacity needed? | Prevents under- or over-provisioning | Riverside Goods application owner |

---

## 9. Decision

**Recommended response:** ALB + multi-AZ Auto Scaling as the required controls; Route 53 alias to the ALB, with failover only to a validated standby.

**Approval condition:** All items in Section 7 are checked and the load test evidence is attached.

**Approved by:** ____________________  **Date:** ____________
