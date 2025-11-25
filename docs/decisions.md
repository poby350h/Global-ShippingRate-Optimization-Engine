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
- Operations can edit shipping rules directly in a spreadsheet.
- No redeploy is needed for most business changes.
- Scheduler and manual `/force-reload` keep cache in sync.

4. Snapshots and analytics
- Cache snapshots are stored in GCS for DR.
- Snapshots can be exported asynchronously to Cloud SQL for analysis.
- Checkout latency does not depend on Cloud SQL.
