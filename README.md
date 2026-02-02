#!/usr/bin/env bash
# name: create-shopify-products.sh
# Usage:
#   export SHOP="your-store.myshopify.com"
#   export TOKEN="YOUR_ACCESS_TOKEN"
#   export MUT_QTY=50      # optional override
#   export REV_QTY=30      # optional override
#   chmod +x create-shopify-products.sh
#   ./create-shopify-products.sh

set -euo pipefail

: "${SHOP:?Set SHOP env var to your-store.myshopify.com}"
: "${TOKEN:?Set TOKEN env var to your Admin API access token}"

API_VER="2024-07"
BASE="https://${SHOP}/admin/api/${API_VER}"
HEADERS=( -H "X-Shopify-Access-Token: ${TOKEN}" -H "Content-Type: application/json" )

# Prices from you: Mutchum R50, Revlon R35
MUT_PRICE="${MUT_PRICE:-50.00}"
REV_PRICE="${REV_PRICE:-35.00}"

# Default inventory quantities (override with env vars)
MUT_QTY="${MUT_QTY:-50}"
REV_QTY="${REV_QTY:-30}"

# Placeholder public image URLs — replace with your URLs if available
MUT_IMG="${MUT_IMG:-https://example.com/images/mutchum-rollon.jpg}"
REV_IMG="${REV_IMG:-https://example.com/images/revlon-body-spray.jpg}"

require_cmd() {
  command -v "$1" >/dev/null 2>&1 || { echo "Missing required command: $1"; exit 1; }
}
require_cmd curl
require_cmd jq

echo "Fetching first location_id..."
LOCATION_ID=$(curl -s "${HEADERS[@]}" "${BASE}/locations.json" | jq -r '.locations[0].id')
if [[ -z "$LOCATION_ID" || "$LOCATION_ID" == "null" ]]; then
  echo "ERROR: No location found. Check your token and store."
  exit 1
fi
echo "Using location_id: $LOCATION_ID"

create_product() {
  local payload="$1"
  curl -s -X POST "${HEADERS[@]}" "${BASE}/products.json" -d "$payload"
}

echo "Creating Mutchum Roll On (price R${MUT_PRICE})..."
MUT_PAYLOAD=$(jq -n \
  --arg title "Mutchum Roll On" \
  --arg body "<p>Antiperspirant roll-on. Long-lasting protection.</p>" \
  --arg vendor "MUTCHUM" \
  --arg ptype "Deodorant" \
  --arg tags "deodorant,roll-on" \
  --arg img "$MUT_IMG" \
  --arg sku "MUT-RR-001" \
  --arg price "$MUT_PRICE" \
  '{
    product: {
      title: $title,
      body_html: $body,
      vendor: $vendor,
      product_type: $ptype,
      tags: $tags,
      images: [ { src: $img } ],
      options: [ { name: "Title" } ],
      variants: [ { option1: "Default", sku: $sku, price: $price, inventory_management: "shopify" } ]
    }
  }'
)
MUT_RESP=$(create_product "$MUT_PAYLOAD")
MUT_INV_ITEM_ID=$(echo "$MUT_RESP" | jq -r '.product.variants[0].inventory_item_id')
MUT_VARIANT_ID=$(echo "$MUT_RESP" | jq -r '.product.variants[0].id')
echo "Mutchum created. variant_id: $MUT_VARIANT_ID, inventory_item_id: $MUT_INV_ITEM_ID"

echo "Creating Revlon Body Spray (price R${REV_PRICE})..."
REV_PAYLOAD=$(jq -n \
  --arg title "Revlon Body Spray" \
  --arg body "<p>Refreshing body spray by Revlon. Light fragrance, all day freshness.</p>" \
  --arg vendor "REVLON" \
  --arg ptype "Body Spray" \
  --arg tags "fragrance,body spray" \
  --arg img "$REV_IMG" \
  --arg sku "REV-BS-001" \
  --arg price "$REV_PRICE" \
  '{
    product: {
      title: $title,
      body_html: $body,
      vendor: $vendor,
      product_type: $ptype,
      tags: $tags,
      images: [ { src: $img } ],
      options: [ { name: "Title" } ],
      variants: [ { option1: "Default", sku: $sku, price: $price, inventory_management: "shopify" } ]
    }
  }'
)
REV_RESP=$(create_product "$REV_PAYLOAD")
REV_INV_ITEM_ID=$(echo "$REV_RESP" | jq -r '.product.variants[0].inventory_item_id')
REV_VARIANT_ID=$(echo "$REV_RESP" | jq -r '.product.variants[0].id')
echo "Revlon created. variant_id: $REV_VARIANT_ID, inventory_item_id: $REV_INV_ITEM_ID"

set_inventory() {
  local inv_item_id="$1"
  local qty="$2"
  payload=$(jq -n --argjson loc "$LOCATION_ID" --argjson iid "$inv_item_id" --argjson av "$qty" '{ location_id: $loc, inventory_item_id: $iid, available: $av }')
  curl -s -X POST "${HEADERS[@]}" "${BASE}/inventory_levels/set.json" -d "$payload" >/dev/null
  echo "Set inventory for inventory_item_id $inv_item_id => $qty units"
}

echo "Setting inventory quantities..."
set_inventory "$MUT_INV_ITEM_ID" "$MUT_QTY"
set_inventory "$REV_INV_ITEM_ID" "$REV_QTY"

echo "Done. Products created and inventory set."
