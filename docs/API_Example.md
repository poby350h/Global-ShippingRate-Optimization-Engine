 Example API Interaction (simplified)

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


