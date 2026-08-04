# WunderGraph Cloud Rate Limits

## Platform-Level Request Quotas

WunderGraph Cosmo enforces monthly request quotas per plan tier. These are soft limits
on the total number of GraphQL requests routed through the Cosmo Router per billing
period:

| Plan       | Monthly Request Limit |
|------------|-----------------------|
| Developer  | 10 million            |
| Launch     | 250 million           |
| Scale      | 1 billion             |
| Enterprise | Custom                |

## Router-Level Rate Limiting

WunderGraph Cosmo Router supports configurable per-request rate limiting using a
Redis-backed simple strategy. Rate limiting is available on the Scale and Enterprise
plans. Configuration is done at the router level via YAML and supports:

- **Global rate limits** applied to all incoming requests
- **Per-key overrides** based on request attributes (e.g., by client ID or IP)
- **Redis storage** for distributed rate limit state across router instances

Example configuration pattern:

```yaml
rate_limit:
  storage:
    url: redis://localhost:6379
    key_prefix: cosmo_rate_limit
  simple_strategy:
    rate: 100
    burst: 200
    period: 1s
```

The first matching override wins; unmatched requests fall back to global defaults.

## Control Plane API

Specific rate limits for the Cosmo Control Plane API (cosmo-cp.wundergraph.com) are
not publicly documented. Enterprise customers may negotiate custom limits with dedicated
support.

## Notes

- Rate limiting as a feature (for protecting your own subgraphs) is only available
  on the Scale and Enterprise plans.
- Data retention windows (7–30 days depending on plan) affect how long analytics and
  tracing data is queryable but do not constitute a rate limit.

Source: https://wundergraph.com/pricing, https://cosmo-docs.wundergraph.com/
