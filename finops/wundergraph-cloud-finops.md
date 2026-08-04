# WunderGraph Cloud FinOps

## Pricing Model

WunderGraph Cosmo uses a flat monthly subscription model with tiered pricing. Costs
are predictable and not metered per request beyond the plan's included monthly
request quota.

## Cost Tiers

| Plan       | Monthly Cost | Included Requests |
|------------|--------------|-------------------|
| Developer  | $0           | 10M               |
| Launch     | $499         | 250M              |
| Scale      | $3,499       | 1B                |
| Enterprise | Custom       | Custom            |

## Cost Drivers

- **Request volume:** The primary driver for plan selection. Exceeding the plan quota
  may trigger an upgrade to the next tier.
- **Team size:** Plans cap the number of users (1 / 10 / 25 / unlimited).
- **Federated graph count:** Plans limit the number of federated graphs
  (1 / 2 / 5 / custom).
- **Subgraph count:** Plans limit the number of subgraphs
  (10 / 20 / 100 / custom).
- **Data retention:** Longer retention (7 / 14 / 30 / custom days) maps to higher
  tiers.

## Cost Optimization Strategies

- Start on the free Developer plan for prototyping and single-developer projects.
- Use persisted queries (Scale+) to reduce redundant schema parsing and lower
  effective request overhead.
- Monitor request usage via Cosmo Studio analytics to right-size the plan before
  the billing cycle resets.
- Leverage SSO/RBAC (Scale+) to consolidate multiple team seats under a single
  managed identity provider rather than adding individual user seats.
- Negotiate startup or nonprofit discounts directly with WunderGraph for early-stage
  or mission-driven organizations.

## Enterprise Procurement

Enterprise pricing is negotiated directly and can include:
- Custom request volumes and data retention
- Dedicated cloud environment
- Custom MSA (Master Service Agreement)
- SLA up to 99.99% uptime

## Overage and Upgrade Policy

Specific overage fees are not publicly documented. Contact WunderGraph sales for
details on what happens when monthly request quotas are exceeded.

Source: https://wundergraph.com/pricing
