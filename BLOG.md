# Building an HTAP Conversational Agent on Google Cloud: FroyoOS Store Manager

**Project Pioneer submission — evolving the AlloyDB + MCP Toolbox codelab into a real
hybrid transactional/analytical (HTAP) agent.**

Repo: https://github.com/gowtham66867/froyo-data

---

## TL;DR

I built **FroyoOS Store Manager**, a natural-language operations assistant for a frozen-yogurt
business. A store manager can ask *"Does Pure Hazelnut Halo have any allergens?"* or say
*"Order 2 Pure Hazelnut Halo for Alice."* A **Google ADK** agent (Gemini 2.5 Flash on **Vertex AI**)
turns those into tool calls served by **MCP Toolbox for Databases** running on **Cloud Run**, which
reads analytical data from **BigQuery** and writes transactions to **AlloyDB for PostgreSQL**.

The interesting part isn't the happy path — it's the five real problems I had to solve to make the
backend actually work. This post is about those.

## The use case

A frozen-yogurt operator has two very different data needs:

- **Analytical:** "What's in this product? Which allergens does it contain?" — joins across a
  product/ingredient/allergen catalog. This is warehouse-shaped work.
- **Transactional:** "Place this order." — a low-latency insert that must be durable and queryable
  immediately.

That's a textbook **HTAP** problem. Instead of forcing both into one engine, I let each store do
what it's best at and put a single conversational agent on top:

```
User → Flask UI → ADK Agent (Gemini 2.5 Flash / Vertex AI)
                     │
                     ▼
              MCP Toolbox (Cloud Run, private, Direct VPC egress)
                ├── check_allergens → BigQuery  (analytical read)
                └── place_order     → AlloyDB    (transactional write)
```

## How this evolves the original codelab

The reference codelab pairs an agent with **AlloyDB** through MCP Toolbox. My version is the same
**tech stack and architecture, applied to a new use case and extended into a multi-store design**:

1. **Two data sources instead of one.** I added a `bigquery` source to MCP Toolbox so the
   *analytical* read path (`check_allergens`) runs on BigQuery, while the *transactional* write
   path (`place_order`) stays on AlloyDB. One agent, two engines, chosen per tool.
2. **Vertex AI instead of an API key.** The agent authenticates to Gemini using Application Default
   Credentials (`GOOGLE_GENAI_USE_VERTEXAI=TRUE`) — no API key in the repo or environment.
3. **A correctness fix in the allergen query** (see below).
4. **A name-based ordering tool** so the agent can place an order without knowing internal SKUs.
5. **Private-by-default deployment** that works under an org policy which forbids public Cloud Run.

## Five problems I had to solve

### 1. The allergen join was silently wrong

The starting `check_allergens` query joined an **ID to a name**:

```sql
INNER JOIN ingredient i ON c.ingredient_id = i.ingredient_name   -- 🐛
```

`ingredient_id` is an identifier; `ingredient_name` is text. The join matched nothing, so the tool
returned empty results and the agent confidently reported "no allergens" for everything. The fix:

```sql
INNER JOIN ingredient i ON c.ingredient_id = i.ingredient_id     -- ✅
```

I verified the keys actually join (832 of 834 `consistsof` rows match `ingredient`) before trusting it.

### 2. MCP Toolbox v1.4.0 disables the REST API by default

The `toolbox-core` Python client calls `/api/...` REST endpoints. Recent Toolbox images return:

```
{"status":"Gone","error":"/api native endpoints are disabled by default. Please use /mcp ..."}
```

The fix is a server flag, discoverable from the binary's own help:

```bash
docker run --rm us-central1-docker.pkg.dev/database-toolbox/toolbox/toolbox:latest --help
#   --enable-api   Enable the /api endpoint.
```

Adding `--enable-api` to the Cloud Run container args restored compatibility with the SDK.

### 3. Reaching AlloyDB's private IP from Cloud Run

AlloyDB only had a private IP (`10.22.1.7`). The Toolbox on Cloud Run can't see it by default. The
clean solution is **Direct VPC egress** — no connector to manage:

```bash
gcloud run deploy toolbox ... \
  --network=default --subnet=default --vpc-egress=private-ranges-only
```

I confirmed connectivity by invoking a tool and watching it execute a query (a `relation "x" does
not exist` error is *progress* — it means the database was reached).

### 4. Org policy blocks public Cloud Run

`--allow-unauthenticated` was rejected because the org enforces Domain Restricted Sharing. Rather
than fight it, I left the service **private** and connected through an authenticated proxy:

```bash
gcloud run services proxy toolbox --region=us-east1 --port=5000
```

The proxy mints correctly-audienced identity tokens from my credentials. No public DB-writing
endpoint is ever exposed — strictly better than the original public deployment.

### 5. The agent didn't know SKUs

`place_order` originally required a `product_id`. But a manager says *"order 2 Pure Hazelnut Halo"* —
they don't know the SKU, and the `product` catalog lives in **BigQuery**, not AlloyDB, so the
AlloyDB insert couldn't look it up. I changed the tool to accept the **product name** and store it
directly. Now the natural-language order flow works end to end.

## A subtlety worth keeping

While testing I asked *"Does Midnight Swirl have any allergens?"* and the agent said **no** — which
looked like a bug, because a `%Midnight%` wildcard returned "Soy". It turned out the agent was
**right**: "Midnight Swirl" genuinely has no allergens; my wildcard had also matched *"Midnight
Papaya Halo"*, which does. A good reminder to check the data before "fixing" a correct answer.

## Result

A working HTAP agent, verified end to end:

```
"Does Pure Hazelnut Halo have any allergens?"
  → "Yes — Soy, Tree Nuts."                         (BigQuery read)

"Order 2 Pure Hazelnut Halo for Alice."
  → "Order placed. Your order ID is 2."             (AlloyDB write)
```

## Stack

Google ADK · Gemini 2.5 Flash (Vertex AI) · MCP Toolbox for Databases · Cloud Run (Direct VPC
egress) · Secret Manager · AlloyDB for PostgreSQL · BigQuery · Flask.

Full, reproducible deploy steps are in [`DEPLOYMENT.md`](./DEPLOYMENT.md). Code:
https://github.com/gowtham66867/froyo-data
