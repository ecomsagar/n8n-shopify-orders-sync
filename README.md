# n8n Shopify Orders Sync — HTTP Request, no Shopify node

Pulls new Shopify orders on a schedule into Google Sheets, with a Telegram
alert for each one. Built with the raw HTTP Request node so the same pattern
transfers to any REST API — Noon, Amazon SP-API, or a custom backend.

https://prnt.sc/UwV8QdJDaL3a

## The problem

Multi-channel sellers run Amazon, Noon and Shopify side by side — three
dashboards, three data formats, and somebody checking all of them by hand.

The real cost isn't the time spent copying. It's overselling: an item sells
on one channel, stock isn't updated on another, and the account takes a
health hit.

## The flow

```
Schedule (15 min) → HTTP Request (GET orders) → Split Out
                  → Set (normalize) → Google Sheets → Telegram
```

| Node | What it does |
|---|---|
| Schedule Trigger | Fires every 15 minutes — knows nothing else |
| HTTP Request | GETs orders, token in the header, filters in the query |
| Split Out | Breaks the `orders` array into one item per order |
| Set | Flattens ~80 API fields down to the 5 that matter |
| Google Sheets | Appends a row |
| Telegram | Sends the alert |

## Requirements

- n8n (self-hosted or cloud)
- A Shopify store with a custom app (see Auth below)
- Google Sheets credential
- Telegram bot token + chat ID

## Auth: this changed in 2026

Shopify stopped issuing long-lived Admin API access tokens for custom apps
created from January 2026 onward. New apps are created in the **Dev Dashboard**
and you request a token yourself using the client credentials grant.

**Step 1 — create the app**

Dev Dashboard → Create an app → scopes `read_orders,read_products` → Install
on your store. Copy the **Client ID** and **Client secret** from Overview →
Credentials.

**Step 2 — request a token**

A one-off HTTP Request node:

```
POST https://YOUR-STORE.myshopify.com/admin/oauth/access_token

Body (JSON):
  client_id     = your client id
  client_secret = your client secret
  grant_type    = client_credentials
```

Response:

```json
{
  "access_token": "shpat_...",
  "scope": "read_orders,read_products",
  "expires_in": 86399
}
```

**`expires_in: 86399` is 24 hours.** Hardcode that token into a scheduled
workflow and it dies silently the next day — no error, the orders just stop
arriving. For production, put the token request as the first node of the
workflow so every run fetches a fresh one.

## Setup

1. Import `workflow.json` into n8n
2. Replace `YOUR-STORE`, `YOUR_ACCESS_TOKEN`, `YOUR_SHEET_ID`, `YOUR_CHAT_ID`
3. Create a sheet with headers in row 1:
   `OrderID | Number | Amount | Currency | Date`
4. Add your Google Sheets and Telegram credentials
5. Activate

## Fetching the orders

```
GET https://YOUR-STORE.myshopify.com/admin/api/2026-07/orders.json

Header:
  X-Shopify-Access-Token: shpat_...

Query:
  status          = any
  limit           = 5
  created_at_min  = {{ new Date(Date.now() - 20*60*1000).toISOString() }}
```

Three parts of one request, each doing a different job: the header proves who
you are, the query narrows what you want, and there's no body because you're
asking rather than sending.

## Detecting what's "new"

A polling trigger will re-fetch the same orders forever unless something tells
it where it left off.

With Gmail you can label a message as processed. Orders aren't yours to mark,
so the filter has to be time-based. The rolling window above asks for anything
created in the last 20 minutes, while the schedule runs every 15 — the 5-minute
overlap means nothing slips through a delayed run.

**Trade-off:** orders landing in that overlap arrive twice. The production fix
is storing a last-run timestamp in a sheet and reading it back on each run,
which also survives n8n being down. Not implemented here yet.

## Gotchas worth knowing

**Timezones differ per API.**
Shopify sends `2026-07-31T08:07:41+04:00` — the store's local time. Tally sends
UTC with a `Z`. Never assume; check what each API actually returns before doing
date maths.

**Split Out removes a nesting level.**
Before it, an order is at `$json.orders[0].id`. After it, `$json.id`. Every
downstream expression gets shorter.

**Empty results aren't always a bug.**
A date filter returning nothing might mean the filter is broken, or it might
mean there are genuinely no orders. Tell them apart by running a query whose
answer you already know — a date range you're certain contains an order.

**Google Sheets strips trailing zeros.**
`89.00` lands as `89`. Format the column as a number with 2 decimals.

## Known limitations

- `limit` is 5 and pagination isn't handled — 50+ orders in a window will be
  truncated
- Only one channel. Adding Noon or Amazon means a second HTTP Request branch
  and a Set node that normalizes both into the same shape
- Date column is stored raw, not formatted for reading

## Scoping questions before building this for someone

1. How many channels?
2. Does each one have working API access? *(this sets the price — Shopify is
   easy, Amazon SP-API is not)*
3. How many orders a day?
4. Read-only, or does stock need syncing back?

Expect the permission screen to alarm them — it says the app can view customer
data. The honest answer: order data contains customer names and addresses, so
it can't be separated. Everything is read-only, and uninstalling the app ends
access immediately.

---

Part of a 26-week automation build series. This is week 3.
