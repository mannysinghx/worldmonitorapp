# 🚂 Deploying World Monitor to Railway

This guide deploys the **World Monitor dashboard** (nginx static frontend + Node
API) to [Railway](https://railway.com) using the project's existing `Dockerfile`.

The dashboard runs out of the box on public data sources. A cache (Redis) and
the AIS relay are **optional** add-on services that unlock cached/seeded data and
live vessel tracking respectively.

---

## TL;DR

1. Create a Railway project from this GitHub repo.
2. Railway reads [`railway.json`](../railway.json) and builds the root
   `Dockerfile` automatically.
3. The container binds to Railway's injected `$PORT` (no config needed).
4. Add a public domain in **Settings → Networking**.
5. (Optional) Add the Redis data layer and AIS relay services below.

That's it for a working dashboard. The data-layer and relay sections are
optional enhancements.

---

## 1. The web app (required)

### Builder

Railway picks up [`railway.json`](../railway.json), which pins the **Dockerfile**
builder and a health check:

```jsonc
{
  "build":  { "builder": "DOCKERFILE", "dockerfilePath": "Dockerfile" },
  "deploy": { "healthcheckPath": "/", "restartPolicyType": "ON_FAILURE" }
}
```

> **Why the health check targets `/` and not `/api/health`:** `/api/health` is a
> *data-readiness* probe — it returns **503 (`REDIS_DOWN`)** whenever no Redis
> cache is configured (see [`api/health.js`](../api/health.js)). Since the
> dashboard is designed to run fine without the cache, gating Railway's
> liveness probe on `/api/health` makes a perfectly healthy container fail its
> health check and get killed. `/` (the nginx-served SPA) returns `200` as soon
> as the web server is up, which is the correct liveness signal. Once you add
> the Redis layer (§2) you can switch the probe back to `/api/health` for true
> data-readiness checks.

> **Note on `nixpacks.toml`:** the repo also ships a root `nixpacks.toml` that
> the upstream maintainer uses for a *separate* AIS-relay service. Because
> `railway.json` explicitly sets `builder: DOCKERFILE`, it takes precedence and
> the web-app service ignores `nixpacks.toml`. No action needed.

### Port

The container reads `$PORT` at boot (`docker/entrypoint.sh` → nginx
`listen ${PORT}`), falling back to `8080` for local Docker Compose. Railway
injects `$PORT` automatically — **do not** hard-code a `PORT` variable.

### Health check

Railway probes `/` (the SPA), which returns `200` as soon as nginx is serving.
`/api/health` is a richer *data-readiness* endpoint but returns `503` without a
Redis cache — see the note above for why it is not used as the liveness probe.

### Minimum environment variables

None are required to boot. Add API keys to unlock data feeds (all optional,
features degrade gracefully). Common ones:

| Variable | Unlocks |
| --- | --- |
| `GROQ_API_KEY` *or* `OPENROUTER_API_KEY` | LLM intelligence assessments |
| `FINNHUB_API_KEY` | Market data |
| `FRED_API_KEY` / `EIA_API_KEY` | Economic & energy data |
| `NASA_FIRMS_API_KEY` | Wildfire detections |
| `AISSTREAM_API_KEY` | Live vessels (also needed by the relay) |
| `ACLED_EMAIL` + `ACLED_PASSWORD` | Conflict events |
| `CLOUDFLARE_API_TOKEN` | Internet outages (paid Radar) |

See [`SELF_HOSTING.md`](../SELF_HOSTING.md) and `.env.example` for the full list.

---

## 2. Data layer — Redis (optional but recommended)

The dashboard reads cached/seeded data through the **Upstash REST protocol**
(`UPSTASH_REDIS_REST_URL` + `UPSTASH_REDIS_REST_TOKEN`). Pick one option:

### Option A — Upstash Serverless Redis (simplest)

1. Create a free database at <https://upstash.com>.
2. Copy its REST URL + token into the web-app service variables:

   ```
   UPSTASH_REDIS_REST_URL=https://<your-db>.upstash.io
   UPSTASH_REDIS_REST_TOKEN=<your-token>
   ```

No proxy needed — Upstash speaks the REST protocol natively.

### Option B — Railway Redis + REST proxy

1. Add Railway's **Redis** database to the project.
2. Add a second service from this repo that builds `docker/Dockerfile.redis-rest`
   (point the service's *Config-as-code* path at a copy of `railway.relay.json`
   edited to use `dockerfilePath: docker/Dockerfile.redis-rest`, or set the
   builder/Dockerfile path in the service UI). Set on that proxy service:

   ```
   SRH_TOKEN=<generate: openssl rand -hex 32>
   SRH_CONNECTION_STRING=redis://default:${{Redis.REDIS_PASSWORD}}@${{Redis.RAILWAY_PRIVATE_DOMAIN}}:6379
   ```
3. Point the web app at the proxy over the private network:

   ```
   UPSTASH_REDIS_REST_URL=http://<redis-rest-service>.railway.internal:80
   UPSTASH_REDIS_REST_TOKEN=<same SRH_TOKEN as above>
   ```

### Seeding data

Cached feeds stay empty until the seed scripts populate Redis. Run them from any
host that can reach the REST endpoint (see [`SELF_HOSTING.md`](../SELF_HOSTING.md)
§ Seeding), or schedule them with a Railway **Cron** service:

```bash
set -a; . ./.env; set +a
export UPSTASH_REDIS_REST_URL=...   # your endpoint
export UPSTASH_REDIS_REST_TOKEN=... # your token
./scripts/run-seeders.sh
```

---

## 3. AIS relay — live vessels (optional)

For live vessel tracking, deploy the relay as a separate service:

1. New service → same repo → set its *Config-as-code* path to
   [`railway.relay.json`](../railway.relay.json) (builds `Dockerfile.relay`).
2. Set `AISSTREAM_API_KEY` on it.
3. Point the web app at it over the private network:

   ```
   WS_RELAY_URL=http://<relay-service>.railway.internal:${{relay.PORT}}
   ```

The relay reads `$PORT` automatically.

---

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| Build uses Nixpacks instead of Docker | Confirm `railway.json` is at repo root and the service's config path points to it. |
| Health check fails / 502 | Confirm the service has no manual `PORT` override; the app must bind Railway's injected `$PORT`. |
| `0/N OK` on `/api/health` | Expected with no cache — add the Redis layer (§2) and run the seeders. |
| No vessel data | Deploy the relay (§3) and set `AISSTREAM_API_KEY` on **both** services. |
