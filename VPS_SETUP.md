# VPS bootstrap

Run once, on the box, before the first deploy. Everything after this is the
`deploy` workflow — including the nginx vhost, which it installs and reloads on
every run.

Throughout, `$VPS_USER` is the SSH user GitHub Actions deploys as.

---

## 1. Prerequisites

```bash
docker --version && docker compose version
nginx -v && certbot --version
sudo ufw status                       # expect 22, 80, 443 and nothing else
docker network ls | grep -E 'planelyx|listryx|auth'   # note the exact names
```

Those names are what `PLANELYX_NETWORK`, `LISTRYX_NETWORK` and `AUTH_NETWORK`
must be set to — one per network Prometheus has to reach into. On a box running
`compose.prod.yaml` (project `planelyx`, network key `planelyx`) the first is
`planelyx_planelyx`. Do not guess any of them — a wrong value fails at `up`
with "network not found".

Note that Keycloak is deployed as its own compose project rather than as part of
planelyx, so it is a third network (`auth_auth`) and a tenant in its own right,
labelled `project="auth"`.

Nothing here needs a port opened. Every service binds `127.0.0.1`, and the only
public entry point is nginx on 443. Publishing a container port on `0.0.0.0`
would bypass ufw entirely, so do not "fix" a connection problem that way.

---

## 2. DNS

Create an `A` record:

```
monitoring.macedosoftware.com.   A   <VPS public IP>
```

Confirm it before touching certbot — requesting a certificate against
un-propagated DNS burns a Let's Encrypt rate-limit slot for nothing:

```bash
dig +short monitoring.macedosoftware.com     # must print the VPS IP
```

---

## 3. Certificate

```bash
sudo mkdir -p /var/www/certbot
sudo certbot certonly --webroot -w /var/www/certbot -d monitoring.macedosoftware.com
sudo certbot renew --dry-run
```

**`certonly --webroot`, not `--nginx`.** `--nginx` rewrites the vhost file it
manages, and the deploy workflow overwrites that same file on every run. The two
would fight and eventually break TLS. `certonly` never touches nginx config; our
vhost carries the `ssl_certificate` paths itself.

For the webroot challenge to work, nginx must already serve
`/.well-known/acme-challenge/` for this hostname — so if this is the very first
run, do step 4 first with a temporary HTTP-only vhost, or issue with
`--standalone` after `sudo systemctl stop nginx`.

---

## 4. The vhost file and its ownership

The deploy workflow rewrites this file without `sudo`, so create it owned by the
deploy user, and symlink it once:

```bash
sudo install -o "$VPS_USER" -g "$VPS_USER" -m 644 /dev/null \
    /etc/nginx/sites-available/monitoring.conf
sudo ln -sfn /etc/nginx/sites-available/monitoring.conf \
    /etc/nginx/sites-enabled/monitoring.conf
```

The file is empty at this point, which nginx accepts. The first deploy fills it.

---

## 5. Sudoers rule

The pipeline needs exactly two privileged commands:

```bash
sudo visudo -f /etc/sudoers.d/monitoring-deploy
```

```
<VPS_USER> ALL=(root) NOPASSWD: /usr/sbin/nginx -t, /bin/systemctl reload nginx
```

Deliberately not blanket `NOPASSWD`: a leaked deploy key then buys a config test
and a reload, not root. Check the binary paths with `command -v nginx` and
`command -v systemctl` — they differ across distributions, and a path that does
not match is a rule that silently never applies.

Verify **as the deploy user**, non-interactively, or the failure surfaces
mid-deploy instead:

```bash
sudo -n nginx -t && echo "sudoers rule works"
```

---

## 6. GitHub secrets and variables

Repository → Settings → Environments → `production`.

**Secrets**

| Name | How to get it |
|---|---|
| `VPS_HOST` | The VPS hostname or IP |
| `VPS_USER` | The deploy user |
| `VPS_SSH_KEY` | Private key whose public half is in that user's `authorized_keys` |
| `VPS_SSH_KNOWN_HOSTS` | `ssh-keyscan -H <VPS_HOST>` — paste the output |
| `GF_SECURITY_ADMIN_PASSWORD` | Generate one: `openssl rand -base64 24`. No single quotes. |

**Variables**

| Name | Value |
|---|---|
| `PLANELYX_NETWORK` | From step 1, e.g. `planelyx_planelyx` |
| `LISTRYX_NETWORK` | From step 1, the listryx network |
| `AUTH_NETWORK` | From step 1, the Keycloak network, e.g. `auth_auth` |
| `MONITORING_HOST` | A name for this box; becomes the `host` label on every log line |
| `PROM_RETENTION_TIME` | Optional, default `30d` |
| `PROM_RETENTION_SIZE` | Optional, default `8GB` |

These may reuse the same SSH secrets as `planelyx-infra` if that deploys as the
same user.

---

## 7. Disk

```bash
df -h /
```

Defaults are 30 days of metrics capped at 8 GB, plus 30 days of logs with no cap.
Prometheus enforces whichever of time/size it hits first, so the 8 GB is the real
protection; Loki's retention is time-only, and its footprint depends entirely on
how chatty the containers are.

Budget roughly 12-15 GB for both and adjust `PROM_RETENTION_SIZE` down if the box
is tight. Changing it later is a variable change and a redeploy, not a migration.

---

## 8. First deploy

Run the `deploy` workflow. It fails loudly if any Prometheus target is down, so a
green run means metrics, logs and the proxy are all genuinely working.

Then verify by hand:

```bash
# From your laptop
curl -sI https://monitoring.macedosoftware.com/login | head -1     # 200
curl -sI https://planelyx.com/ui/ | head -1                        # 200, neighbour intact
curl -s https://planelyx.com/actuator/prometheus -o /dev/null -w '%{http_code}\n'  # 404
curl -s https://planelyx.com/ocr/metrics -o /dev/null -w '%{http_code}\n'          # 404

# The monitoring ports must not be reachable from outside
for p in 9090 3100 3000; do
    timeout 3 bash -c "</dev/tcp/<VPS_IP>/$p" 2>/dev/null \
        && echo "PORT $p IS OPEN — fix this" || echo "port $p closed"
done

# On the box
cd ~/monitoring && docker compose ps
curl -s 'http://127.0.0.1:9090/api/v1/targets?state=any' | jq -r \
    '.data.activeTargets[] | "\(.health)\t\(.labels.job)"'
```

Log in to Grafana as `admin` with `GF_SECURITY_ADMIN_PASSWORD` and change it, or
add your own user and disable the shared admin.

After 24 hours, check the retention assumptions against reality:

```bash
docker system df -v | grep -E 'monitoring_(prometheus|loki)-data'
```

---

## 9. Rollback and teardown

Take the site offline without touching planelyx:

```bash
sudo rm /etc/nginx/sites-enabled/monitoring.conf
sudo nginx -t && sudo systemctl reload nginx
cd ~/monitoring && docker compose down
```

Roll back a bad vhost by hand (the workflow already does this automatically when
`nginx -t` fails):

```bash
cp ~/monitoring/nginx/monitoring.conf.prev /etc/nginx/sites-available/monitoring.conf
sudo nginx -t && sudo systemctl reload nginx
```

`docker compose down` keeps the named volumes, so metrics and logs survive. Add
`-v` only if you mean to discard the history.
