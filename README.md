# monitoring

Prometheus + Grafana + Loki + Alloy for the whole VPS. An independent project:
planelyx is simply its first tenant, described by two files in *this* repo.

- **Metrics** — Prometheus, scraping targets listed in `prometheus/targets/`.
- **Logs** — Alloy discovers every container through the Docker API and ships to
  Loki. No per-project configuration; a new container is picked up automatically.
- **Host and container resources** — node-exporter and cAdvisor, likewise for
  everything on the box with no cooperation from the thing being monitored.

Grafana is the only thing exposed, at `https://monitoring.macedosoftware.com`.
Everything else binds `127.0.0.1`.

## Running it locally

```bash
cp .env.example .env      # defaults are already the local ones
docker compose up -d
```

Then <http://localhost:3000> — user `admin`, password from `.env`.

`.env` selects the environment. Locally it points at the dev networks and
`prometheus/targets/local/`; the deploy workflow renders the prod equivalent.

Two things behave differently locally, both because the planelyx dev stack is
shaped differently — not because the monitoring config forks:

- The Spring API runs from the IDE on the host, so it is scraped through
  `host.docker.internal` and its **logs are not collected** (Alloy only sees
  containers).
- Keycloak's compose service is `keycloak` locally and `auth` in prod.

listryx has neither quirk: its whole dev stack runs in compose under a pinned
project name, so it is scraped by service name and labelled `project="listryx"`
in both environments.

## Adding a project

Two files, both here. Nothing in the monitored project's repo.

1. `projects/<name>.compose.yaml` — declare its network as external and attach
   Prometheus to it. Copy `planelyx.compose.yaml`.
2. `prometheus/targets/<env>/<name>.yml` — one entry per scrape target:

   ```yaml
   - targets: ['my-service:8080']
     labels:
       job: my-service
       project: my-project
       __metrics_path__: /metrics
   ```

Then append the override to `COMPOSE_FILE` in `.env` and set the network name.
Prometheus re-reads the target files every 30s, so adding a target to an existing
project needs no restart at all.

A project with no metrics endpoint needs neither file: its logs and container
resources are already collected.

If the project's log lines need parsing — mapping pino's numeric `level` back to a
word, or stitching Java stack traces — add a `stage.match` in `alloy/config.alloy`.
Select on **both** `project` and `service`: service names collide across projects
(`api` is both a Spring app and a Nest app here), and a project that does not pin
its compose project name is labelled with its directory locally and its real name
in prod, so those selectors need an alternation.

## Deploying

`.github/workflows/deploy.yml`, triggered manually. It ships the config tree,
renders `.env` from secrets, brings the stack up, installs the nginx vhost
(validating before reloading, restoring the previous file if `nginx -t` fails),
and then fails the run if any Prometheus target is down.

First time on a new box: **`VPS_SETUP.md`**.

## Layout

```
compose.yaml                  the stack
projects/                     one override per monitored project
prometheus/prometheus.yml     scrape config
prometheus/targets/{local,prod}/
loki/loki-config.yml
alloy/config.alloy            log discovery and processing
grafana/provisioning/         datasources and the dashboard provider
grafana/dashboards/           dashboard JSON, provisioned read-only
nginx/monitoring.conf         vhost, installed by the pipeline
```

Dashboards are provisioned from files, so edits made in the Grafana UI are
overwritten on restart. Change the JSON and redeploy.
