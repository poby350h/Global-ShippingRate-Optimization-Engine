##Design Decisions

1. Cache-first, DB-free checkout path
- All shipping rates are served from in-memory cache.
- No database calls in the critical checkout path.
- This avoids DB becoming a single point of failure.

2. Multi-region Cloud Run + Global HTTPS Load Balancer
- Deployed to us-central1 and us-east4.
- Global LB routes traffic to the nearest healthy region.
- Improves availability and resilience to regional incidents.

3. Google Sheets as the source of truth

- Shipping rules are managed by the operations team directly in Google Sheets.
- Each rule category is separated into multiple tabs
  (e.g., country rules, weight tiers, bundle mappings, promotions, fixed pricing, etc.).
- The API parses and loads all tabs into structured in-memory objects at startup or reload.
- No redeploy is required for most business rule changes.
- Scheduler and manual `/force-reload` keep the runtime cache fully synchronized with Sheets.

4. Snapshots, DR, and analytics decoupling

- Cache snapshots are stored in GCS for fast restore and disaster recovery.
- Snapshots can be exported asynchronously to Cloud SQL for analytics and reporting.
- Checkout latency and availability never depend on Cloud SQL.
