# Project Pioneer Submission

## Project title

Project Pioneer: FroyoOS Store Manager

## Tagline

An ADK-powered store operations agent that uses MCP Toolbox, AlloyDB, and BigQuery federation to answer product questions and process live orders.

## What it does

FroyoOS Store Manager lets a store manager interact with operational and analytical data through natural language. The agent can:

- Check product allergens from federated BigQuery product/ingredient/allergen tables.
- Place live customer orders into AlloyDB's `live_orders` table.
- Keep the application code decoupled from raw database credentials and SQL wiring through MCP Toolbox.

## Why it matters

Operational agents often need both transactional and analytical context. This project demonstrates an HTAP-style pattern where an agent inserts live order transactions and queries warehouse-grade product/allergen data through one AlloyDB endpoint.

## Built with

- Google ADK
- Gemini
- MCP Toolbox for Databases
- AlloyDB for PostgreSQL
- BigQuery federation
- Cloud Run
- Secret Manager
- Flask

## Architecture

1. The user interacts with a Flask web UI.
2. The Flask route sends the prompt to a Google ADK agent.
3. The ADK agent selects the correct tool.
4. MCP Toolbox exposes the database operation as a named tool.
5. AlloyDB executes the operation:
   - `check_allergens` scans federated BigQuery tables.
   - `place_order` inserts into the local `live_orders` table.

## Demo script

1. Open the web app.
2. Ask: `Does Midnight Swirl have any allergens?`
3. Expected behavior: the agent calls `check_allergens` and answers with the allergen from the data.
4. Ask: `Order 2 Midnight Swirl for Alice.`
5. Expected behavior: the agent calls `place_order`, inserts into `live_orders`, and confirms the order.

## Submission links

- GitHub repo: `https://github.com/gowtham66867/froyo-data`
- Demo video: add final video URL
- Cloud Run app URL: add final app URL if deployed
- MCP Toolbox URL: keep private or omit from public submission if not needed

## Current status

- Repo prepared for submission.
- Safe `.env.example` provided.
- Real `.env` removed from git tracking.
- Toolbox `tools.yaml` template configured for the project/region/cluster.
- Cloud deployment requires authenticated `gcloud` in Cloud Shell and a real AlloyDB password.

