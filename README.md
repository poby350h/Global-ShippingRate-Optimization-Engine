# Portfolio Case Study : Multi-region, cache-first Shipping Rate Engine for Shopify Checkout (DB-free critical path)

1. Overview

This repository documents a production-grade Shipping Rate API designed, implemented, and operated for a Shopify store.

The goal was to calculate complex international shipping rates in real time, while keeping the checkout path fast and highly available.

> Source code is private due to security/IP concerns.  
> This repository focuses on architecture, design decisions, and operations.  
> Code can be reviewed privately during interviews if needed.

---

2. Problem

Shopify's built-in shipping settings were not flexible enough to handle:

- Country-specific rules (US, CA, EU,AU, etc.)
- Weight-based tiers and surcharges
- Bundle rules (main + accessory items)
- Temporary promotions and pre-order discounts

At the same time:

- The checkout flow is extremely latency-sensitive.
- Any blocking dependency (DB, external API, or slow network call) could directly impact revenue.
- Database outages or long queries must not block checkout.
- Non-developer staff needed to update shipping rules via google sheet without redeploying code.

---

3. Solution Summary

I built a cache-first Shipping Rate API behind Shopify’s CarrierService:

- Global HTTPS Load Balancer → Multi-region Cloud Run**  
  (us-central1 & us-east4) for high availability.
- In-memory cache(`shipping_cache`) inside each Cloud Run instance.  
  Checkout requests are served **entirely from cache**.
- Google Sheets as the source of truth** for shipping rules.  
  Operations team can edit rules directly in a spreadsheet.
- Cloud Scheduler triggers a `/force-reload` endpoint every morning  
  to refresh the cache from Google Sheets.
- Cloud Storage snapshots + Cloud SQL (PostgreSQL) used Only for
  DR and analytics — Never in the critical checkout path.
- Google IAM Service Account + Application Default Credentials + Secret Manager   
  for secure access to Sheets / Storage.
- Shopify HMAC verification to ensure all incoming requests are
  authentic and unmodified.

---

4. Architecture

![Architecture](https://github.com/Poby350H/-BigTent-shipping-api-portfolio/blob/main/docs/shipping_rate-architecture.png)


High-level flow:

A. Shopify Checkout calls the CarrierService REST API.

B. Request goes through the HTTPS Load Balancer to the nearest healthy
   Cloud Run region.
   
C. The Shipping Rate API:
   - Verifies the **Shopify HMAC** using the app secret.
   - Looks up rates from the **in-memory cache**.
   - Returns rates to Shopify in the expected format.
     
D. Shipping rules lifecycle:

- On service startup:
  - Load latest cache snapshot from GCS if available.
  - If missing, pull from Google Sheets and create a new snapshot.

- On scheduled reload (Cloud Scheduler → /force-reload):
  - Always pull from Google Sheets.
  - Overwrite in-memory cache.
  - Persist a fresh snapshot to GCS.

- On manual reload (admin → /force-reload):
  - Immediate rule update without redeploy.
  - Updates both in-memory cache and GCS snapshot.
    
E. The current cache is periodically saved to **Cloud Storage** as
   snapshots. Snapshots can be:
   - Used for **DR/restore**.
   - Exported asynchronously to Cloud SQL for reporting and analysis.

Checkout latency and availability do not depend on Cloud SQL.

---

5. Tech Stack

- Backend: Python(Flask) REST API integrated with Shopify CarrierService  
- Infra: Google Cloud (Google Cloud Run, Cloud Load Balancing, Cloud Scheduler)  
- Data: Google Sheets (gspread + Application Default Credentials),  
  Cloud Storage (cache snapshots), Cloud SQL (PostgreSQL – analytics only)  
- Security & Auth: IAM Service Account,Secret Manager,HMAC verification, environment-based secrets

---

6. Key Features

- Cache-first design
  - All rate calculations use in-memory data.
  - No database queries in the checkout request path.

- Multi-region deployment
  - Cloud Run in `us-central1` and `us-east4`
  - Global HTTPS Load Balancer routes traffic and provides failover.

- Rule-driven configuration via Google Sheets
  - Operations team can change rates, countries, tiers, and bundles
    directly in a spreadsheet.
  - No code changes or redeploys required for most business updates.

- Automated & manual cache reload
  - Daily reload via Cloud Scheduler.
  - Manual reload endpoint for urgent rule changes.

- Resilience & observability
  - Cache snapshots stored in Cloud Storage for DR.
  - Snapshots exported to Cloud SQL for analytics and debugging.
  - Structured logging around each shipping request.

- Security
  - Shopify HMAC verification for every request.
  - GCP IAM service account with least-privilege roles.
  - Runtime secrets (Shopify tokens, sheet IDs, etc.) stored outside code
    (e.g., environment variables / Secret Manager).

---

7. Impact

- Reliability: Since launch, there have been no user-visible incidents
  attributable to this shipping rate service in production.

- Performance: Internal logs show end-to-end response times consistently under 100 ms
  ( p95 < 120 ms), keeping checkout latency impact negligible.

- Operational agility: Shipping rules can be updated by the operations team
  directly in Google Sheets and applied immediately via `/force-reload`,
  without any code changes or redeployments.

- Resilience: Checkout does not depend on Cloud SQL, and cache snapshots
  in GCS allow the service to recover quickly from failures or restarts.

---

8.Further reading

- [API examples](docs/api_examples.md)  
- [Design decisions](docs/decisions.md)

****

