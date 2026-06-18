# Deployment Guide (verified)

These are the exact steps used to bring the FroyoOS Store Manager backend up on Google Cloud.

## 0. Prerequisites

```bash
gcloud auth login
gcloud config set project eastern-map-498917-i6
gcloud config set run/region us-east1

gcloud services enable \
  run.googleapis.com alloydb.googleapis.com bigquery.googleapis.com \
  secretmanager.googleapis.com vpcaccess.googleapis.com \
  artifactregistry.googleapis.com aiplatform.googleapis.com
```

Confirm the AlloyDB cluster/instance and note the VPC network + subnet:

```bash
gcloud alloydb clusters  list --region=us-east1
gcloud alloydb instances list --cluster=froyo-cluster --region=us-east1
gcloud compute networks subnets list --filter="region:us-east1"
```

## 1. Service account + IAM

```bash
SA="toolbox-identity@eastern-map-498917-i6.iam.gserviceaccount.com"
gcloud iam service-accounts create toolbox-identity --display-name="MCP Toolbox" || true
for ROLE in roles/secretmanager.secretAccessor roles/alloydb.client \
            roles/serviceusage.serviceUsageConsumer \
            roles/bigquery.dataViewer roles/bigquery.jobUser; do
  gcloud projects add-iam-policy-binding eastern-map-498917-i6 \
    --member="serviceAccount:$SA" --role="$ROLE" --condition=None
done
```

## 2. tools.yaml + Secret Manager

Inject the real AlloyDB password into `tools.yaml` (the template ships with `__ALLOYDB_PW__`)
and store it as a secret — never commit the real password:

```bash
read -s -p "AlloyDB password: " ALLOYDB_PASSWORD; echo
sed "s|__ALLOYDB_PW__|$ALLOYDB_PASSWORD|" tools.yaml > /tmp/tools.yaml
gcloud secrets create tools --data-file=/tmp/tools.yaml \
  || gcloud secrets versions add tools --data-file=/tmp/tools.yaml
```

## 3. Deploy the MCP Toolbox to Cloud Run

Direct VPC egress lets the Toolbox reach AlloyDB's **private IP**. `--enable-api` turns on the
REST `/api` endpoints that the `toolbox-core` client uses (they are off by default in v1.4.0+).

```bash
gcloud run deploy toolbox \
  --image=us-central1-docker.pkg.dev/database-toolbox/toolbox/toolbox:latest \
  --region=us-east1 \
  --service-account="$SA" \
  --set-secrets=/app/tools.yaml=tools:latest \
  --args=--tools-file=/app/tools.yaml,--address=0.0.0.0,--port=8080,--enable-api \
  --port=8080 \
  --network=default --subnet=default --vpc-egress=private-ranges-only \
  --no-allow-unauthenticated
```

> Note: this org enforces Domain Restricted Sharing, so public (`allUsers`) Cloud Run access is
> blocked. The service is deployed **private** and accessed via an authenticated proxy (below).

## 4. Create the live_orders table

The Toolbox config includes a one-off `init_orders` tool. Through the proxy (step 5):

```bash
curl -s -X POST http://127.0.0.1:5000/api/tool/init_orders/invoke \
  -H "Content-Type: application/json" -d '{}'
```

(Alternatively run the equivalent `CREATE TABLE IF NOT EXISTS live_orders (...)` in AlloyDB Studio.)

## 5. Authenticated proxy + run the agent

```bash
# Terminal A — keep this running
gcloud run services proxy toolbox --region=us-east1 --port=5000

# Terminal B
cp .env.example .env            # Vertex AI defaults, MCP_TOOLBOX_SERVER_URL=http://127.0.0.1:5000
pip install -r requirements.txt
python app.py                   # http://localhost:8080  (use Cloud Shell Web Preview on 8080)
```

## 6. Smoke tests

```bash
curl -s -X POST http://127.0.0.1:5000/api/tool/check_allergens/invoke \
  -H "Content-Type: application/json" -d '{"product_name":"%Hazelnut%"}'
# -> {"result":"[{\"allergen_name\":\"Soy\"}, ...]"}

curl -s -X POST http://127.0.0.1:8080/chat -H "Content-Type: application/json" \
  -d '{"message":"Does Pure Hazelnut Halo have any allergens?"}'
```

## Teardown (after the demo)

```bash
gcloud run services delete toolbox --region=us-east1 --quiet
gcloud secrets delete tools --quiet
```

## Local fallback (no cloud)

```bash
pip install -r requirements.txt
python app-nobill.py
```
