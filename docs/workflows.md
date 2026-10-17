# Workflow Details

All workflows use the Airtable base "warehouse". Blueprints are in [../make-blueprints](../make-blueprints). W1 to W4 were built in one Make.com account and W5 to W7 in another, so connection labels differ between the files. Re-create the connections after importing.

## W1 Inventory Intake

Trigger: custom webhook (instant), called from Postman.

Expected JSON keys: sku, product_name, quantity, category, location, unit_price, Reorder_Level.

Modules:

1. Custom Webhook.
2. Router with two routes.
3. Valid route, filter: sku, product_name and quantity exist, and quantity is greater than or equal to -1.
   1. Iterator over the SKU.
   2. Set Variable inventory_id = INV-YYYYMMDDHHmmss.
   3. Airtable Create Record in Inventory: SKU, Product Name, Category, Quantity, reorder level, Unit Price, Location.
4. Invalid route, filter: sku, product_name or quantity missing.
   1. Webhook Response: HTTP 400 with {"status":"error","message":"Invalid inventory record. Required fields: sku, product_name, quantity"}.

Known limitations:

1. The generated inventory_id variable is not mapped to any Airtable field, so it is never stored.
2. The quantity check allows 0 and -1. Use "greater than or equal to 0" for strict validation.
3. The valid route has no explicit success response.
4. In the invalid-route test the Webhook Response module shows a warning in Make and Postman received 200 "Accepted" instead of the intended 400 error. Open the execution details of that module to fix it.
5. The module reads Reorder_Level while the request body uses reorder_level, so the reorder level can end up empty. Use one spelling in both places.

## W2 Stock Monitoring

Modules:

1. Airtable Search Records (Inventory, limit 4): Inventory ID, SKU, Product Name, Category, Quantity, reorder_level, Unit Price, Date Added.
2. Router with two routes.
3. Low Stock route, filter: Quantity less than or equal to reorder_level. Airtable Update Record sets Status = Low Stock.
4. Overstock route, filter: Quantity greater than or equal to reorder_level. Airtable Update Record sets Status = Overstock.

Known limitations:

1. When Quantity equals reorder_level both routes match, so the record gets both updates.
2. Overstock starts at the reorder level, so every item above it is Overstock. Use a higher threshold, for example 3 times the reorder level, and add a Normal route.
3. The search limit is 4 records.

## W3 AI Inventory Analysis

Modules:

1. Airtable Search Records (Inventory, limit 4).
2. Array Aggregator: collects the inventory snapshot.
3. Groq Create a Chat Completion, model llama-3.3-70b-versatile. The system prompt demands JSON only, with fast_moving, slow_moving, forecast_notes and reorder_priorities. It forbids inventing sales data or SKUs and returns empty arrays when data is insufficient.
4. Parse JSON.
5. Iterator over reorder_priorities, then Text Aggregator building "SKU (reason)" text.
6. Airtable Create Record in AnalysisLog: Analysis Date = now, Fast Moving Items, Slow Moving Items, Forecast Notes, Reorder Priorities.

Known limitations:

1. The snapshot has no sales history, so fast and slow movers are often empty. Add a movement or sales field for real forecasting.
2. Parse JSON has no data structure set. Define one, or use the Groq JSON response format, to avoid parse errors.
3. The search limit is 4 records.

## W4 Purchase Request Automation

Trigger: Airtable Watch Records on Inventory, trigger field Last Updated, limit 5.

Modules:

1. Airtable Watch Records (Inventory).
2. Airtable Search Records (PurchaseRequests, limit 4). Filter: the inventory record Status equals Low Stock.
3. Airtable Create Record in PurchaseRequests. Filter: found request SKU equals inventory SKU and found request Status equals Pending. Values: SKU, Item Name, Quantity Requested = (reorder_level x 3) - Quantity, Status = Pending, Supplier Response = Awaiting Response.

Known limitations:

1. The create filter fires only when a Pending request for the same SKU already exists. For duplicate prevention it should fire when none exists. Review this before submitting.
2. The search is not filtered by SKU, so it checks only the first 4 requests.
3. SKU and Item Name are copied from the searched request instead of the inventory record.

## W5 Supplier Notification

Trigger: Airtable Watch Records on PurchaseRequests, trigger field Last Modified, limit 2.

Modules:

1. Airtable Watch Records (PurchaseRequests).
2. Gmail Send an Email. Filter: Status equals Pending. HTML table with Item Name, SKU, Quantity Requested and Status. Subject "Product Demand".
3. Airtable Update Record: Status = Sent to Supplier, Supplier Response = Awaiting Response.

Known limitations:

1. The recipient is a fixed address instead of the supplier email from the linked Suppliers record.
2. The step 3 update runs even when the Pending filter blocks the email, so the status can change without an email being sent. Move the filter before both modules, or add one on the update.

## W6A Scheduled Inventory Audit: inactive items

Modules:

1. Airtable Search Records (Inventory, limit 4): Inventory ID, SKU, Last Updated, Status.
2. Airtable Update Record. Filter: Status is not "inactive". Sets Status = Inactive and Date Added = now.
3. Text Aggregator over SKU, one per line.
4. Airtable Create Record in Audit, filter: aggregated text not empty. Date = now, Inactive Items Marked = SKU list.

Known limitations:

1. There is no inactivity rule. Every item that is not already Inactive is marked Inactive. Add a Last Updated age check, for example older than 30 days.
2. The update overwrites Date Added with the current time.
3. The aggregator lists all searched SKUs, including ones the filter skipped.

## W6B Scheduled Inventory Audit: overdue purchase requests

Modules:

1. Airtable Search Records (PurchaseRequests, limit 10).
2. Text Aggregator over SKU. Filter: Status = Pending, or Status = Sent to Supplier and Date Requested later than 3.
3. Airtable Create Record in Audit, filter: aggregated text not empty. Date = now, Overdue Purchase Requests = SKU list.

Known limitations:

1. The overdue condition compares Date Requested with the value 3, which is not a clear date threshold. Use formatDate and addDays, for example older than 3 days.
2. Every Pending request is reported as overdue, whatever its age.

## W7 Weekly Inventory KPI and Audit Digest

Schedule: weekly, Monday 08:00.

Modules:

1. Search Inventory (limit 1000) then Array Aggregator: total items.
2. Search Inventory with formula Status = "Active" then aggregator: Active count.
3. Same pattern for Low Stock, Overstock and Inactive.
4. Search AnalysisLog, newest first, limit 1: Fast Moving Items, Slow Moving Items, Forecast Notes, Reorder Priorities.
5. Search Audit, newest first, limit 1: Inactive Items Marked, Overdue Purchase Requests.
6. Gmail Send an Email: HTML report with the KPI table, AI analysis and audit findings. Subject "Weekly Warehouse Inventory Report".

Known limitations:

1. When a status search returns no records, the aggregator still reports 1, so an empty category shows as 1.
2. Inventory Status has no "Active" choice in the W1 blueprint (In Stock, Low Stock, Overstock, Normal, Inactive), while W7 counts Active. Keep the choice names consistent.
3. The recipient is a fixed address.
