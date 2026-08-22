# monitoring

Prometheus + Grafana + Loki + Alloy for the whole VPS. An independent project:
every monitored stack is just a tenant, described entirely by files in *this*
repo. Current tenants are `planelyx`, `listryx` and `auth` (Keycloak).

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

Two things behave differently locally, because those dev stacks are shaped
differently — not because the monitoring config forks:

- planelyx's Spring API runs from the IDE on the host, so it is scraped through
  `host.docker.internal` and its **logs are not collected** (Alloy only sees
  containers).
- Keycloak is its own compose project in prod (`auth_auth`), so `.env` there
  loads `projects/auth.compose.yaml`. Locally the target file assumes it is
  reachable on a network Prometheus already joins. If your local Keycloak also
  runs as a separate project, add the override and `AUTH_NETWORK` to `.env` too.

listryx has neither quirk: its whole dev stack runs in compose under a pinned
project name, so its target file is byte-identical in both environments.

`auth` is a tenant in its own right, not part of planelyx. Keycloak is a
third-party service that happens to authenticate planelyx, it is deployed and
versioned separately, and Alloy already labels its logs `project="auth"` from
the compose label — so labelling its metrics `project="planelyx"` would split
one service across two names depending on which datasource you asked. The JVM
dashboard spans both with `up{project=~"planelyx|auth"}`.

## Adding a project

Everything lives here. Nothing in the monitored project's repo.

1. `projects/<name>.compose.yaml` — declare its network as external and attach
   Prometheus to it. Copy `planelyx.compose.yaml`.
2. `prometheus/targets/<env>/<name>.yml` — one entry per scrape target. Address
   targets by **container name**, not compose service name:

   ```yaml
   - targets: ['my-project-api-1:8080']
     labels:
       job: my-project-api
       project: my-project
       __metrics_path__: /metrics
   ```

   Prometheus sits on every tenant's network at once, and Docker's embedded DNS
   has no per-network FQDN to disambiguate with. A bare service name that two
   projects share — `api` is both a Spring app and a Nest app here — resolves to
   whichever network answers first, so a tenant's job silently scrapes another
   tenant's container. Container names are unique across the box; service names
   are not.
3. Wire it into **both** environments, or it only works on your laptop:
   - `.env` for local — append the override to `COMPOSE_FILE` and set
     `<NAME>_NETWORK`.
   - `.github/workflows/deploy.yml` for prod — the rendered `.env` is a literal
     in the *Stage the rendered .env* step. Append the override to its
     `COMPOSE_FILE` line, `printf` the network variable, and add that variable
     to the *Validate inputs* list and to `VPS_SETUP.md`.

Prometheus re-reads the target files every 30s, so adding a target to a project
that is *already* wired up needs no restart at all.

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
