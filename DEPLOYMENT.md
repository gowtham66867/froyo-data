# Deployment Guide

## Prerequisites

```bash
gcloud auth login
gcloud config set project eastern-map-498917-i6
```

Enable APIs:

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

## Create live orders table

Run in AlloyDB Studio:

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

## Deploy MCP Toolbox

Create a real `tools.yaml` from the template and replace `REPLACE_WITH_ALLOYDB_PASSWORD`.

```bash
cd ~/froyo-agent
gcloud secrets create tools-froyo --data-file=tools.yaml || \
  gcloud secrets versions add tools-froyo --data-file=tools.yaml
```

Create a Cloud Run service account if needed:

```bash
gcloud iam service-accounts create toolbox-identity \
  --display-name="MCP Toolbox Identity" || true
```

Deploy Toolbox. Replace network/subnet if your Part 2 values differ:

```bash
export IMAGE=us-central1-docker.pkg.dev/database-toolbox/toolbox/toolbox:latest

gcloud run deploy toolbox-froyo \
  --image "$IMAGE" \
  --service-account toolbox-identity \
  --region us-east1 \
  --set-secrets "/app/tools.yaml=tools-froyo:latest" \
  --args="--config=/app/tools.yaml","--address=0.0.0.0","--port=8080" \
  --network easy-alloydb-vpc \
  --subnet easy-alloydb-subnet \
  --allow-unauthenticated \
  --vpc-egress private-ranges-only
```

Copy the resulting Cloud Run URL into `.env`:

```bash
MCP_TOOLBOX_SERVER_URL=https://...
```

## Run the agent app locally

```bash
cd ~/froyo-data
pip install -r requirements.txt
python app.py
```

## Run fallback mode

```bash
python app-nobill.py
```

