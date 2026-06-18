# Project Pioneer Submission

## Project title

FroyoOS Store Manager — an HTAP conversational ops agent on Google Cloud

## Tagline

A Google ADK agent that uses MCP Toolbox to read analytical data from **BigQuery** and write live
transactions to **AlloyDB** — natural-language store operations over a real HTAP backend.

## What it does

FroyoOS Store Manager lets a store manager interact with operational and analytical data through
natural language. The agent can:

- **Check product allergens** by querying the `froyo_data` tables in **BigQuery** (`check_allergens`).
- **Place live customer orders** by inserting into AlloyDB's `live_orders` table (`place_order`).
- Stay fully decoupled from raw SQL and credentials — every database operation is a declarative,
  named tool served by **MCP Toolbox for Databases**.

## Why it matters

Operational agents need both *analytical* context (what's in the catalog) and *transactional*
action (record this order). This project demonstrates a clean **HTAP split**: heavy analytical
reads go to BigQuery, low-latency transactional writes go to AlloyDB, and a single ADK agent
orchestrates both through one MCP tool layer.

## Built with

- Google ADK
- Gemini 2.5 Flash via **Vertex AI** (ADC auth, no API key)
- MCP Toolbox for Databases (`bigquery` + `alloydb-postgres` sources)
- Cloud Run (Toolbox hosting, **Direct VPC egress** to AlloyDB private IP)
- Secret Manager (Toolbox config)
- AlloyDB for PostgreSQL, BigQuery, Flask

## Architecture

1. The user interacts with a Flask web UI.
2. The Flask route forwards the prompt to a Google ADK agent (Gemini 2.5 Flash on Vertex AI).
3. The agent selects a tool exposed by MCP Toolbox running on Cloud Run.
4. MCP Toolbox executes the operation against the right store:
   - `check_allergens` → **BigQuery** `froyo_data` (analytical read).
   - `place_order` → **AlloyDB** `live_orders` (transactional write).
5. The Toolbox reaches AlloyDB's private IP via Direct VPC egress; it is a private Cloud Run
   service accessed through an authenticated proxy.

## Demo script (verified)

1. Open the web app (Cloud Shell Web Preview on port 8080).
2. Ask: `Does Pure Hazelnut Halo have any allergens?`
   → "Yes, Pure Hazelnut Halo contains the following allergens: Soy, Tree Nuts." (BigQuery)
3. Ask: `Order 2 Pure Hazelnut Halo for Alice.`
   → "Alice, your order for 2 Pure Hazelnut Halo has been placed. Your order ID is 2." (AlloyDB)
4. Ask: `Does Midnight Swirl have any allergens?`
   → "Midnight Swirl does not contain any allergens." (correct negative case)

## Submission links

- GitHub repo: https://github.com/gowtham66867/froyo-data
- Blog post: see `BLOG.md` (publish to Medium/dev.to or link the GitHub copy)
- Demo video / screenshots: add final URL

## Current status

- Backend fully deployed and verified: MCP Toolbox on Cloud Run (`toolbox`, us-east1),
  BigQuery read path and AlloyDB write path both working through the live agent.
- `tools.yaml` ships with a placeholder password; the real password is injected at deploy time
  and stored only in Secret Manager.
- Model runs on Vertex AI via Application Default Credentials — no API key in the repo.
