# AI Upsell, Cross-Sell & Next-Best-Action Engine (n8n)

An n8n workflow that gives every customer one next best action (upsell, cross-sell, retention or none) using their usage, tickets and a product catalog from the database. Strong opportunities are sent to the sales team on Slack.

▶ **Demo video:** https://drive.google.com/file/d/1FIJnoDPsV6wssw_Qp4QOCVsR_5eTg57O/view

## How it works

1. Mock customers arrive (plan, usage, products owned, months, open tickets)
2. The product catalog is read from the Postgres `products` table and combined into the prompt
3. The AI Agent (Gemini) picks one action and one product from the catalog, with a confidence score and reason
4. A Code node validates the answer. Confidence of 70 or more with a real action counts as `strong`
5. Strong opportunities go to Slack, every recommendation is saved to Postgres

```
Manual Trigger -> Mock Customers -> Get Products (Postgres) -> Combine Catalog
-> AI Agent (Gemini) -> Parse Recommendation -> Strong Opportunity?
   yes -> Slack Alert -> Save Recommendation
   no  -> Save Recommendation
```

## AI Agent

AI Agent node without tools (JSON answer). It may only recommend products that exist in the catalog.

## Tech stack

n8n, AI Agent, Gemini, Sales, Upsell, Slack, Postgres

## Setup

1. Copy the JSON from `workflow.md` into a file named `next-best-action-aiagent-cloud.json` and import it into n8n
2. Create credentials (all have free tiers):
   - Google Gemini (API key)
   - Slack (Bot token and signing secret)
   - Postgres (Supabase Session pooler, SSL: Require)
3. In the Slack node set channel `sales-opportunities`, then run `/invite @your-bot` in that channel
4. Create the database tables (below)
5. In the Postgres save node remove the `id` and `created_at` fields, the database fills them
6. Click **Execute workflow**: 4 mock inputs run through all branches

## Database

```sql
create table if not exists public.products (
  id bigint generated always as identity primary key,
  name text,
  price numeric,
  category text,
  best_for text
);

create table if not exists public.recommendations (
  id bigint generated always as identity primary key,
  customer text,
  plan text,
  action text,
  product text,
  confidence numeric,
  reason text,
  status text,
  created_at timestamptz default now()
);

insert into public.products (name, price, category, best_for) values
  ('CRM Starter', 29, 'crm', 'very small teams just starting out'),
  ('CRM Pro', 79, 'crm', 'growing teams that need automation and reports'),
  ('CRM Enterprise', 199, 'crm', 'large teams needing advanced security and SSO'),
  ('Email Add-on', 19, 'addon', 'customers who send many email campaigns'),
  ('Analytics Pack', 49, 'addon', 'customers with high usage who want dashboards'),
  ('Priority Support', 39, 'addon', 'customers with many open support tickets');
```

## Demo data

| Mock input | Expected result |
|---|---|
| Northwind Traders (Starter, 92% usage) | Likely upsell to a higher plan |
| Bluebird Clinics (Pro, 55% usage) | Likely cross-sell or none |
| Tailspin Logistics (6 open tickets) | Likely retention first |
| Fabrikam Studio (Enterprise, 88% usage) | Likely cross-sell |

## Notes

- The sample `products` rows are examples, replace them with your real catalog
- Replace Mock Customers with a Postgres select on your customer table plus a Schedule Trigger for real use
- Slack message text must be in **Expression** mode so `{{ }}` values are filled in
- Never commit API keys. Use n8n credentials only

## Author

Zarin Tasnim Shaily, AI Automation Engineer
[LinkedIn](https://linkedin.com/in/5ha1l72) | [GitHub](https://github.com/ztsprofessionalaiid7-alt)
