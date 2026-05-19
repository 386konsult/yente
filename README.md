# yente

`yente` is an open source data match-making API. The service provides several HTTP endpoints to search, retrieve or match [FollowTheMoney entities](https://www.opensanctions.org/docs/entities/), including people, companies or vessels that are subject to international sanctions.

The yente API is built to provide access to [OpenSanctions data](https://www.opensanctions.org/datasets/), and can also be used to search and match other data, such as [company registries](https://www.opensanctions.org/datasets/kyb/) or [custom watchlists](https://www.opensanctions.org/docs/yente/datasets/).

While `yente` is the open source core code base for the [OpenSanctions API](https://www.opensanctions.org/api/), it can also be run [on-premises as a KYC appliance](https://www.opensanctions.org/docs/self-hosted/) so that no customer data leaves your infrastructure.

* [yente documentation](https://www.opensanctions.org/docs/yente/) - install, configure and use the service.

## Self-hosted Deployment (SmartComply)

This deployment runs yente on a single server using Docker Compose with a multi-replica architecture and external cron-based reindexing. All infrastructure configuration is self-contained in this repository.

### Architecture Overview

```
                         ┌──────────────┐
                         │   Traefik    │
                         │  (reverse    │
                         │   proxy)     │
                         └──────┬───────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
            ┌───────▼───────┐     ┌─────────▼───────┐
            │   app         │     │   app           │
            │  (replica 1)  │     │  (replica 2)    │
            │  YENTE_AUTO_  │     │  YENTE_AUTO_    │
            │  REINDEX=false│     │  REINDEX=false  │
            └───────┬───────┘     └────────┬────────┘
                    │                      │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼──────────┐
                    │  index              │
                    │  (Elasticsearch)    │
                    │  port 9200          │
                    └─────────────────────┘
                               ▲
                    (depends_on: healthy)
                               │
                    ┌──────────┴──────────┐
                    │  cron-scheduler     │
                    │  panubo/cron:latest │
                    │  reads crontab.txt  │
                    └──────────┬──────────┘
                               │
                               │ docker compose run --rm
                               │
                    ┌──────────▼──────────┐
                    │  reindexer          │
                    │  (one-shot job)     │
                    │  yente reindex      │
                    └─────────────────────┘
```

| Service | Image | Replicas | Purpose |
|---------|-------|----------|---------|
| `traefik` | `traefik:v3` | 1 | TLS termination, routing to `app` |
| `index` | `elasticsearch:8.19.13` | 1 | Search index, 8GB JVM heap |
| `app` | `yente:5.3.0` | **2** | API serving `/search`, `/match` |
| `reindexer` | `yente:5.3.0` | 1 (profile) | Runs `yente reindex`, exits |
| `cron-scheduler` | `panubo/cron:latest` | 1 | Schedules reindexer via cron |

### Why This Setup

**Multi-replica app (replicas: 2):** Two yente instances behind a load balancer provide resilience. If one fails, the other continues serving. Under normal operation, requests are distributed across both.

**External cron instead of auto-reindex:** With multiple app replicas, `YENTE_AUTO_REINDEX=true` causes all replicas to clash, each trying to reindex simultaneously. Setting `YENTE_AUTO_REINDEX=false` disables the built-in scheduler, and the `cron-scheduler` container triggers reindexing externally — the standard pattern recommended by the yente project for multi-instance deployments.

**Zero-downtime reindexing:** Reindex is non-blocking because Elasticsearch uses index aliases. While `reindexer` builds a new index, the existing index continues serving API requests via the alias. The alias is atomically swapped once the new index is ready — no gap in service.

**panubo/cron:** Chosen over alternatives because it's actively maintained since 2016, logs to stdout/stderr (integrated with `docker compose logs`), and supports dynamic crontab reloading without restart. The container runs as root to install the crontab; go-crond executes jobs as the unprivileged `cron` user.

### Configuration

Environment variables are set in `.env`:

| Variable | Value | Purpose |
|----------|-------|---------|
| `YENTE_INDEX_TYPE` | `elasticsearch` | Search backend type |
| `YENTE_INDEX_URL` | `http://index:9200` | Internal ES connection |
| `YENTE_MANIFEST` | `/app/manifests/commercial.yml` | Dataset manifest |
| `OPENSANCTIONS_DELIVERY_TOKEN` | `<redacted>` | OpenSanctions data access token |
| `YENTE_AUTO_REINDEX` | `false` | Disabled — reindex via cron |

### Schedule

Reindexing runs **twice daily at 00:00 and 12:00 UTC**, defined in `crontab.txt`:

```
0 0,12 * * * cd /app && docker compose run --rm reindexer >> /proc/1/fd/1 2>&1
```

This schedule is version-controlled. To change the frequency, edit `crontab.txt` and restart the scheduler:

```bash
docker compose restart cron-scheduler
```

### Deployment

```bash
# Start everything including cron
docker compose up -d

# Verify all services are healthy
docker compose ps

# Manually trigger a reindex (for immediate updates)
docker compose --profile reindex run --rm reindexer

# View real-time logs
docker compose logs -f cron-scheduler
docker compose logs -f reindexer

# Restart cron with updated schedule
docker compose restart cron-scheduler
```

### Pros & Cons

| Pro | Con |
|-----|-----|
| **Zero-downtime reindex** — ES aliases enable atomic index swap | **Additional container** — cron-scheduler adds minimal overhead (~5MB) |
| **Resilient** — 2 app replicas handle failure gracefully | **Docker socket mount** — cron container requires socket access to orchestrate reindexer |
| **External cron survives app restarts** — reindex schedule is independent of app lifecycle | **UTC-only scheduling** — cron container runs in UTC; DST/timezone shifts not supported |
| **Schedule is version-controlled** — `crontab.txt` lives in repo | **No built-in alerting** — reindex failures are only visible via logs |
| **Portable** — all config lives in `docker-compose.yml` and `.env` | **Pre-initial index required** — fresh deployment is unavailable until first reindex completes |
| **Efficient** — delta/incremental updates by default (`YENTE_DELTA_UPDATES=true`) | **Shared ES resource contention** — heavy reindex load may slightly elevate API latency during the ~5–15min window |

### Troubleshooting

**Reindex failing:**
```bash
# Check the reindexer exit code and logs
docker compose --profile reindex run --rm reindexer
docker compose logs reindexer
```

**Cron not firing:**
```bash
# Verify the container is running
docker compose ps cron-scheduler

# Check scheduler logs
docker compose logs cron-scheduler

# Force a reload of the crontab
docker compose exec cron-scheduler sh -c "kill -HUP \$(cat /tmp/go-crond.pid)"
```

**Elasticsearch not healthy:**
```bash
# Check ES health directly
docker compose exec index curl -s http://localhost:9200/_cluster/health

# View ES logs
docker compose logs index
```

**API returning stale data:**
```bash
# Force an immediate reindex
docker compose --profile reindex run --rm reindexer --force

# Verify the alias points to the latest index
docker compose exec index curl -s http://localhost:9200/_cat/aliases/yente
```

## Development

`yente` is implemented in asynchronous, typed Python using the FastAPI framework. We're happy to see any bug fixes, improvements or extensions from the community. To set up a local development environment, use `uv`:

```bash
git clone https://github.com/opensanctions/yente.git
cd yente
# Install runtime and development dependencies
uv sync
# Install pre-commit hooks with useful checks
prek install
# Activate the virtual environment
source .venv/bin/activate
```

This will install a broad range of dependencies, including `numpy`, `scikit-learn` and `pyicu`, which are binary packages that may require a local build environment. For `pyicu` in particular, refer to the [package documentation](https://pypi.org/project/PyICU/).

### Running the server

Once you've set the ``YENTE_INDEX_URL`` environment variable to point to a running instance of ElasticSearch or OpenSearch, you can run the web server like this:

```bash
yente serve
```


### Releasing

    bump2version --verbose minor # or patch
    git push && git push --tags

## License and Support

``yente`` is licensed according to the MIT license terms documented in ``LICENSE``. Using the service in a commercial context may require a [data license for OpenSanctions data](https://www.opensanctions.org/licensing/).
