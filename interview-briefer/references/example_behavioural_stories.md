# Behavioural Story Bank — Jane Smith

## Quick Reference: Theme → Story Mapping

| Interview Theme | Best Story/Stories |
|---|---|
| Drove alignment across teams with competing priorities | North Star Architecture |
| Influenced without authority | Endpoint Conflict; Latency Observability |
| Technically difficult decision under constraints | Re-Architecture Decision |
| Took on ambiguity and drove clarity | Re-Architecture Decision |
| Had to push back or escalate | Endpoint Conflict |
| Time you failed | Scale Limit Miscalculation |
| Managed a critical incident | Ad Impression Drop |
| Delivered something under significant time pressure | Re-Architecture Decision |
| Improved a process or way of working | Latency Observability |

---

## Full Story Details

### Resolving Conflict with Partner Team
**Theme: Influenced without authority / navigating partner teams**
- **S:** Mid-build, a platform team wanted us to adopt their new endpoint process. The real risk wasn't technical — it was relational. We'd need their approvals throughout the project.
- **T:** Find a path that protected our timeline without burning a critical partnership.
- **A:** Rather than declining outright, met with their staff engineer, shared our constraints openly, and proposed a week-long direct collaboration so we could adopt their process without losing momentum.
- **R:** Both teams hit their goals. The platform team later added analytics to those endpoints that streamlined our monitoring — a friction point became a long-term alliance.

---

### Re-Architecture Decision Under Uncertainty
**Theme: Technically difficult decision under constraints / technical debt awareness**
- **S:** The retrieval layer we depended on was being rewritten by another team — integrating with the old system wasn't viable, but the new one wasn't ready.
- **T:** Design an approach that wouldn't block the MVP and could migrate cleanly when the new system was available.
- **A:** Invested time upfront to understand the target architecture, chose data models that would be easy to ingest later, used a lightweight interim approach, and clearly documented its scale limits and when it would need investment.
- **R:** Unblocked the MVP without hidden technical debt. The documented tradeoffs created shared understanding with leadership and set clear expectations for the follow-up work.

---

### Latency Observability
**Theme: Influenced without authority / using data to drive decisions**
- **S:** As we approached launch, endpoint latency was critical but we had limited visibility into where the bottlenecks were.
- **T:** Get enough observability in place to make fast, data-driven decisions rather than guessing under pressure.
- **A:** Instrumented each stage of the retrieval flow with granular latency metrics and built dashboards that made bottlenecks immediately visible. Used that data to escalate with leadership and get cross-team prioritisation.
- **R:** Drove p99 latency from over 1 second to 500ms. Dashboards also let us quantify parallelisation gains before committing to the work.

---

### Scale Limit Miscalculation (Failure Story)
**Theme: Time you failed / architectural humility**
- **S:** Designed a retrieval system with documented scale limits, with a plan to migrate to something more robust before hitting them.
- **T:** Deliver an architecture that could support launch and scale predictably into early growth.
- **A:** Built it as an intentional short-term tradeoff, flagged it to leadership, and instrumented dashboards to monitor performance.
- **R:** The limit was hit earlier than projected — real usage was more geographically concentrated than estimated. Dashboards caught it before it became user-facing, but the fix required reducing retrieval radius, which meant less content diversity than intended.
- **What I'd do differently:** Validate distribution assumptions earlier using real density data. When you document a scale limit, also define a concrete tripwire metric — a threshold that automatically triggers the follow-up investment rather than relying on projections.

---
