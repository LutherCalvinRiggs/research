# GitHub.com Outage — August 17, 2026 (7h 47m)

**Source:** https://www.githubstatus.com/incidents/zkxwbgr0cnmx
**Saved:** 2026-08-19
**Tags:** technology, infrastructure, security, fundamentals

> Post-incident report from GitHub status page. Incident resolved. Useful as a systems design case study: cascade failure from misconfigured autoscaling → HAProxy exhaustion → retry storm amplification → prolonged recovery.

---

## TL;DR
7h 47m GitHub outage on August 17, 2026 (13:28–21:15 UTC). Root cause: Istio sidecar pod hit concurrency limit and couldn't auto-scale due to a misconfigured policy that watched the host service but not the sidecar limits. One failure cascaded to four HAProxy nodes exhausting flow limits → gateway auth degradation → optimistic retry logic overloaded internal load balancers. VS Code's retry behavior amplified Copilot Token Service traffic 10× (7–9K RPS → 70–100K RPS) and was the last piece to recover.

---

## Timeline

| Time UTC | Event |
|----------|-------|
| 13:28 | Incident begins |
| 13:40 | Investigating |
| 13:41–15:40 | Cascading degradation across API, Actions, PRs, Issues, Webhooks, Pages, Git Ops, Copilot |
| 14:04–16:16 | ~20% web/API error rate; ~50% archive/raw content error rate; SAML, OIDC, SCIM, Team Sync affected |
| 16:36 | "Strong signs of recovery" — Central US datacenter recovered |
| 16:59 | Most services mitigated |
| 18:03 | Actions fully recovered |
| 18:23 | Git Operations mitigated |
| 19:01 | API Requests normalized |
| 20:22 | Issues normalized |
| 21:02–21:15 | Copilot Token Service fully recovered; incident closed |

**Total duration:** 7 hours 47 minutes

---

## Root Cause Chain

```
NEW TRAFFIC PEAK
    ↓
Istio sidecar pod hits concurrency limit
    ↓
Misconfigured autoscaling policy (watches host service, not sidecar limits)
→ Cannot auto-scale → sidecar fails
    ↓
Cascade: one failure → more failures
→ Four HAProxy nodes exhaust flow limits
    ↓
Gateway auth path degraded
→ Widespread authentication latency and failures
    ↓
Optimistic retry logic fires
→ Overloads internal load balancers
    ↓
~20% web/API error rate, ~50% archive/raw content error rate
SAML, OIDC, SCIM, Team Sync affected
Actions workflows in GHEC with Data Residency affected
```

---

## The Retry Storm Pattern (Two Instances)

**Instance 1 (during incident):** Optimistic retry logic in the gateway amplified load on internal load balancers during recovery, slowing restoration.

**Instance 2 (prolonged Copilot recovery):** VS Code's retry behavior amplified Copilot Token Service traffic approximately 10× — from normal 7–9K RPS to 70–100K RPS. A failed token operation could generate many extra requests and enter a retry loop.

**The pattern:** Failed requests → retries → more load → more failures → more retries. A retry storm can take a service that's barely struggling and turn it into a full outage. Without circuit breaking or backoff, retry storms are self-reinforcing.

---

## Mitigations That Worked

**For Central US (primary recovery, ~16:36 UTC):**
- Moved failing traffic to Northern Virginia
- Paused HAProxy on the four exhausted nodes → immediate broad recovery

**For Northern Virginia retry storm:**
- Temporarily reduced gateway retry logic via PR
- Blocked inbound Copilot Token Service token requests at load balancers with 403 (stopped the amplification)
- Gradually ramped traffic back up per-site

**For Copilot Token Service (final recovery, ~21:02 UTC):**
- Reduced gateway authentication retries
- Blocked retry-triggering responses
- These together stabilized the 70–100K RPS surge back to normal

**Complicating factor:** Scraping attacks on codeload endpoints during the incident impeded recovery by adding noise to the signal.

---

## Remediation Actions (GitHub's Follow-Up)

1. Correct autoscaling policies to account for service-mesh sidecar concurrency and capacity
2. Audit Istio request, concurrency, and scaling limits across affected services
3. Review retry limits and backoff behavior across gateways and clients
4. Address VS Code retry behavior that amplified Copilot token traffic
5. Improve load-balancer capacity monitoring and regional failover safeguards

---

## Systems Design Lessons

**The misconfigured autoscaling policy was the initiating cause, not the root cause.** The actual root cause is architectural: a system where one component's autoscaling failure can cascade to HAProxy exhaustion across four nodes, which then degrades a global auth path. Defense in depth would require each layer to fail gracefully rather than propagating load upward.

**Retry logic without circuit breakers is a liability.** Both instances of retry amplification (gateway retries overloading internal LBs, VS Code retries overloading Copilot Token Service) show the same pattern. Exponential backoff with jitter and circuit breakers at the client layer would have contained these to self-limiting rather than self-amplifying failures.

**Traffic shifting as a mitigation is powerful but not instant.** Moving traffic from Central US to Northern Virginia was the primary recovery mechanism — but the retry storm in Northern VA then required separate mitigation. Regional failover is not "free" from a load perspective.

**"Blocking with 403" as a deliberate mitigation.** Temporarily returning 403 to all Copilot Token Service requests stopped the retry loop — callers stopped retrying on 403 (non-retryable error) vs. 5xx (retryable). This is a known technique: sending a definitive error to break retry loops is sometimes more effective than trying to serve traffic through degraded systems.

**The sidecar concurrency gap.** Istio service mesh sidecars are not just passthrough proxies — they have their own concurrency limits that are independent of the host service's limits. Autoscaling policies that watch host service metrics but not sidecar metrics will fail to scale the sidecar, creating a hidden bottleneck. This is a well-known operational pitfall in service mesh deployments.

---

## Questions & Gaps
- What was the "new peak in traffic" that triggered the initial Istio saturation? Organic growth, a specific event, or an unusual usage pattern?
- How many total requests failed during the 7h 47m window? The 20% error rate across GitHub's scale represents an enormous number of failed operations.
- The VS Code retry behavior was a latent bug that manifested only under load — was this a known issue or newly discovered? Fixing client retry behavior in a distributed IDE is a hard coordination problem.
- The scraping attacks during the incident — were these opportunistic or correlated with the outage somehow?

## Related Notes
- [Cursor Git at Scale — Continuity Storage](https://github.com/LutherCalvinRiggs/research/blob/main/technology/infrastructure/cursor-git-at-scale-continuity.md) — published the day after this outage. Cursor's post specifically mentions agents increasing Git load ("more code, more PRs, more CI runs"). This incident is the direct context for why Cursor is building an alternative Git hosting platform. The HAProxy exhaustion pattern here is exactly what Continuity's stateless WAL-based design is designed to avoid — no shared consensus bottleneck means no single point to exhaust.
- [NEEDLE Production Safety Guide](https://github.com/LutherCalvinRiggs/research/blob/main/repos/needle/NEEDLE-Production-Safety-Guide.md) — NEEDLE's retry threshold (stop dispatching after N failures) is the circuit-breaker pattern this incident lacked. The "retry storm" pattern here (failed requests generating more requests) is the headless fleet failure mode the safety guide's failure count threshold prevents.
- [Deterministic State Machines for Non-Deterministic Agents](https://github.com/LutherCalvinRiggs/research/blob/main/ai/productivity/deterministic-state-machines-for-agents.md) — the exhaustive outcome table in NEEDLE maps every possible outcome to a defined handling. This incident is what happens without one: "failed token operation could generate many extra requests and enter a retry loop" — no defined outcome for failure, so the system improvised with retries.
- [System Design Playbook — Neo Kim](https://github.com/LutherCalvinRiggs/research/blob/main/technology/fundamentals/system-design-playbook-neo-kim.md) — the cascade failure pattern here (one misconfigured pod → HAProxy exhaustion → auth degradation → retry storm) is the classic availability failure mode that system design at scale is designed to prevent. Rate limiting, circuit breaking, and bulkhead isolation are the standard mitigations — all absent or misconfigured here.
