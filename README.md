# Shopify Inventory Automation

Event-driven inventory sync between Shopify, Google Sheets, and Slack — built with n8n.

## The problem

Online stores like need real-time visibility into stock levels as
orders come in. Without automation, staff manually check Shopify orders, update
inventory spreadsheets by hand, and remember to flag low stock — a slow,
error-prone process that doesn't scale as order volume grows.

## What this does

1. Customer places an order on Shopify
2. A webhook fires instantly — no polling, no manual refresh
3. The workflow looks up current stock for the ordered product
4. Subtracts the quantity sold
5. Writes the new stock level back to the tracking sheet, along with a timestamp
6. If stock falls below threshold, posts a Slack alert automatically
7. If the ordered product isn't found in the sheet, posts a separate alert so a
   human can add it
8. If any step in the workflow fails (API error, auth issue, missing resource),
   a dedicated error-handling workflow catches it and posts details to a
   separate Slack channel — keeping technical alerts separate from business alerts

## Architecture

![Architecture diagram](./architecture-diagram.png)

Google Sheets stands in for a real ERP system. The architecture generalizes to
any REST or OData-based ERP by swapping the Sheets nodes for HTTP Request nodes
against the ERP's API — the rest of the flow (trigger, business logic, alerting)
stays the same. Shopify and Slack integrate via n8n's native nodes over their
respective REST APIs.

## Edge cases handled

- **Unknown product**: if an ordered item isn't found in the inventory sheet,
  the workflow branches to a distinct Slack alert instead of failing silently
  or crashing downstream nodes
- **Negative stock guard**: stock calculations are floored at zero
  (`Math.max(0, stock - quantity)`), so inventory never displays as negative
- **Field-name consistency**: all downstream nodes reference normalized field
  names set by a single Code node, avoiding brittle direct references to
  spreadsheet column headers
- **Workflow-level failures**: a dedicated Error Trigger workflow catches any
  node failure in the main automation and posts the failed node, error message,
  and a direct link to the execution to a separate `#automation-errors` Slack
  channel

## Known limitations

- **Multi-line-item orders**: currently processes the first line item only.
  Orders with multiple distinct products need further work to loop over the
  full `line_items` array (planned: a Split Out node per line item)
- **HubSpot contact sync**: attempted but blocked by Shopify's Protected
  Customer Data restriction on dev-store webhooks, which strips email and
  customer info from the order payload without explicit app-level approval.
  In production, this would be resolved by requesting Protected Customer Data
  access in the Shopify Partner Dashboard

## Tech stack

- **n8n** — workflow orchestration
- **Shopify** — order source, webhook trigger
- **Google Sheets API** — inventory data store (ERP stand-in)
- **Slack API** — notifications (business alerts + technical error alerts)

## Design decisions

- Chose Google Sheets over a database to keep the demo accessible and
  inspectable, while keeping the node structure directly swappable for a real
  ERP's API
- Split "product not found" and "low stock" into separate Slack messages
  rather than one generic alert, since they require different human actions
- Used a Code node (rather than chained Set nodes) for the stock calculation,
  keeping the math and the negative-stock guard in one auditable place
- Built a separate error-handling workflow rather than relying on n8n's default
  silent failure behavior, so operational issues surface immediately instead of
  going unnoticed

## Setup

1. Import `n8n-workflow-export.json` into your n8n instance
2. Create a Google Sheet with columns: `Product`, `Stock`, `Threshold`
3. Connect Shopify, Google Sheets, and Slack credentials in n8n
4. Set the main workflow's Error Workflow (Settings → Error Workflow) to the
   included error-handler workflow
5. Activate both workflows

## Repo structure

\`\`\`
shopify-inventory-sync-n8n/
├── README.md
├── LICENSE
├── workflow/
│   └── shopify-inventory-sync.json
└── docs/
    ├── architecture-diagram.png
    └── demo-screenshot.png
\`\`\`