# Project Pioneer Setup

This repo contains the FroyoOS Store Manager agent app for the AlloyDB HTAP codelab.

## Current target environment

- Google Cloud project: `eastern-map-498917-i6`
- AlloyDB region: `us-east1`
- AlloyDB cluster: `froyo-cluster`
- AlloyDB instance: `froyo-instance`
- Database: `postgres`
- User: `postgres`
- GitHub repo: `gowtham66867/froyo-data`

## Files

- `app.py`: AlloyDB + MCP Toolbox app
- `app-nobill.py`: local CSV fallback app
- `templates/index.html`: web UI
- `requirements.txt`: Python dependencies
- `.env.example`: safe template for local environment values
- `tools.yaml`: template for MCP Toolbox tool definitions

Do not commit real `.env` values or a real database password.

## Required cloud steps

1. Reauthenticate locally if needed:

```bash
gcloud auth login
gcloud config set project eastern-map-498917-i6
```

2. Enable required APIs:

```bash
gcloud services enable \
  alloydb.googleapis.com \
  bigquery.googleapis.com \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  iam.googleapis.com \
  secretmanager.googleapis.com \
  compute.googleapis.com \
  servicenetworking.googleapis.com
```

3. Create `live_orders` in AlloyDB Studio:

```sql
CREATE TABLE IF NOT EXISTS live_orders (
    order_id SERIAL PRIMARY KEY,
    customer_name VARCHAR(100),
    product_id VARCHAR(100),
    quantity INT,
    order_status VARCHAR(50) DEFAULT 'Pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

4. Create a real local Toolbox config in `~/froyo-agent/tools.yaml` using your AlloyDB values and database password.

5. Save that config to Secret Manager and deploy MCP Toolbox to Cloud Run.

6. Put the Cloud Run URL into `.env` as `MCP_TOOLBOX_SERVER_URL`.

7. Run the app:

```bash
pip install -r requirements.txt
python app.py
```

For a no-cloud fallback:

```bash
python app-nobill.py
```

