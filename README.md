# -BigTent-shipping-api-portfolio
Multi-region Cloud Run shipping rate service for Shopify checkout (DB-free critical path)


# Shipping Rate API (bd-shipping-api) — Portfolio Case Study

1. Overview

This repository documents a production-grade Shipping Rate APII designed, implemented, and operated for a Shopify store.

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
  DR and analytics — not in the critical checkout path.
- Google IAM Service Account + Application Default Credentials + Secret Manager   
  for secure access to Sheets / Storage.
- Shopify HMAC verification to ensure all incoming requests are
  authentic and unmodified.

---

4. Architecture

![Architecture](https://github.com/Poby350H/-BigTent-shipping-api-portfolio/blob/main/docs/shipping_rate-architecture.png)


High-level flow:

1. Shopify Checkout calls the CarrierService REST API.
2. Request goes through the HTTPS Load Balancer to the nearest healthy
   Cloud Run region.
3. The Shipping Rate API:
   - Verifies the **Shopify HMAC** using the app secret.
   - Looks up rates from the **in-memory cache**.
   - Returns rates to Shopify in the expected format.
     
4. Shipping rules are loaded and refreshed as follows:

- On service startup (init):
  - The API first tries to load the latest cache snapshot from Google Cloud Storage (GCS).
  - If no snapshot exists, it falls back to Google Sheets as the source of truth,
    then saves a new snapshot back to GCS.
  
- On scheduled reload (Cloud Scheduler → `/force-reload`):
  - The API always pulls the latest rules from Google Sheets,
    overwrites the in-memory cache, and persists a fresh snapshot to GCS.

- On optional manual reload (admin → `/force-reload`):
  - When shipping prices or rules change and need to be applied immediately,
    an admin can trigger `/force-reload` to force-pull from Google Sheets and
    update both the in-memory cache and the GCS snapshot.
    
5. The current cache is periodically saved to **Cloud Storage** as
   snapshots. Snapshots can be:
   - Used for **DR/restore**.
   - Exported asynchronously to Cloud SQL for reporting and analysis.

Checkout latency and availability do not depend on Cloud SQL.


5. Key Features

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

6. Example API Interaction (simplified)

> Note: Request/response formats are simplified and anonymized.

Request (from Shopify CarrierService):

```http
POST /shopify_shipping
Content-Type: application/json
X-Shopify-Hmac-Sha256: <HMAC_SIGNATURE>
X-Shopify-Shop-Domain: example-shop.myshopify.com

{
  "destination": {
    "country": "US",
    "postal_code": "90001"
  },
  "items": [
    {"sku": "ABC-123", "quantity": 1, "grams": 1200},
    {"sku": "ACC-456", "quantity": 2, "grams": 300}
  ]
}


Response:

{
  "rates": [
    {
      "service_name": "DHL",
      "rate_version": "V1001",
      "currency": "USD",
      "total_price": "2499"
    }
  ]
}
