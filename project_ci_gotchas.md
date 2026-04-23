---
name: CI workflow gotchas (Caddy, setup-pixi, readiness probes, shell)
description: Non-obvious fixes discovered while getting `.github/workflows/ci.yml` working on GitHub Actions ubuntu-latest runners. Referenced when debugging new CI failures or adding similar service setups.
type: project
originSessionId: 98d8c875-05ec-4fca-bb82-8ec75f5e53a7
---
Discovered 2026-04-22 → 2026-04-23 during the `update-to-use-pixi-env` PR. Keeping these together so future sessions don't relitigate.

## Caddy on GitHub Actions runners

Two default Caddy behaviors collide with `ubuntu-latest`. Fix both in the global block:

```
{
  admin off
  auto_https disable_redirects
}
```

- **`admin off`** — Caddy's admin API binds `127.0.0.1:2019` by default. Something on the runner already holds that port → `listen tcp 127.0.0.1:2019: bind: address already in use`. We don't use the admin API (static config, launched once, killed at job end) so disable it outright.
- **`auto_https disable_redirects`** — With a site block like `https://...`, Caddy also binds `:80` for automatic HTTP→HTTPS redirect. Port 80 is already held on ubuntu-latest → `listen tcp :80: bind: address already in use`. We only need HTTPS, so disable the redirect. Prefer `disable_redirects` over blanket `auto_https off` — more surgical.
- **Caddyfile heredoc formatting warning** is cosmetic (leading-whitespace indentation). Run `caddy fmt --overwrite /tmp/Caddyfile` before `caddy run` to silence, or ignore.

## `nohup + sudo` breaks PID capture

`nohup sudo -E foo ... &; PID=$!` captures **sudo's** PID, not foo's. Subsequent `kill $PID` doesn't reach the real process. For Caddy specifically, grant `cap_net_bind_service` so sudo isn't needed:

```
sudo setcap cap_net_bind_service=+ep "$(command -v caddy)"
nohup caddy run --config ... &
CADDY_PID=$!          # now the real caddy PID
```

Same pattern applies to any long-lived process we want to clean up cleanly.

## `wait` after `disown` is a no-op

`disown` removes the job from bash's job table, so `wait "${PID}"` returns immediately (exit 127) without actually waiting. Don't write it — it's misleading dead code. `kill -SIGINT` delivers the signal regardless of disown status.

## `prefix-dev/setup-pixi@v0.8.8` when pixi.toml is not at workspace root

Our real `pixi.toml` lives in `${CLONED_REPO}` (a runtime-cloned profile repo), not at workspace root. The action's defaults assume a workspace-root pixi.toml and error in post-step with `ENOENT: no such file or directory, lstat '.pixi'`. Fix:

```yaml
- uses: prefix-dev/setup-pixi@v0.8.8
  with:
    pixi-version: v0.67.1
    run-install: false    # skip auto-install; we run `pixi install` manually with --manifest-path
    cache: false          # skip the post-step cache save that hits ENOENT
```

## Tiled readiness probe: use `/healthz`, not catalog endpoints

`/api/v1/metadata` depends on catalog/duckdb initialization and can 5xx/404 before tiled is truly serving. Use `/healthz` — designed as the up/down probe, no catalog dependency. Matches what tiled's own `SimpleTiledServer` fix does upstream.

## rdkafka + `localhost` resolves to IPv6 first

With `BLUESKY_KAFKA_BOOTSTRAP_SERVERS=localhost:9092` and a Docker port mapping on IPv4 only, rdkafka logs `Connect to ipv6#[::1]:9092 failed: Connection refused` before retrying on IPv4. Cosmetic but noisy. Fix: use `127.0.0.1:9092` in env vars and `/etc/bluesky/kafka.yml` to force IPv4.

## Kafka in KRaft mode for CI

Single-node Kafka via `apache/kafka:3.9.0` in KRaft mode (no Zookeeper). See `ci.yml` "Start Kafka" step for the full env-var set. Readiness check uses `docker exec kafka /opt/kafka/bin/kafka-broker-api-versions.sh --bootstrap-server localhost:9092` — proper broker-ready check, not just port-open.

## Shell pattern: readiness loops

Use flag + break, not `exit 0` inside `if` inside `for`. The `exit`-inside-nested-structures pattern sometimes exits non-zero under `bash -leo pipefail` in ways that are hard to diagnose. Pattern:

```bash
ready=0
for i in {1..N}; do
  if <probe>; then ready=1; break; fi
  sleep 1
done
if [ "$ready" -ne 1 ]; then
  echo "not ready"
  <dump logs>
  exit 1
fi
echo "ready"
```

## Mocking external NSLS-II API calls for profile startup

Profile startup files often call `https://api.nsls2.bnl.gov/v1/facility/nsls2/cycles/current` to fetch the current beam cycle. In CI, this URL is unreachable (no external network). Mock it with:

1. **Add DNS hijack** in "Configure DNS and seed Redis" step:
   ```bash
   echo "127.0.0.1 api.nsls2.bnl.gov" | sudo tee -a /etc/hosts
   ```

2. **Start mock HTTP server** (in "Start Tiled" step, before Tiled startup):
   ```bash
   python3 << 'MOCK_API_EOF' &
   from http.server import HTTPServer, BaseHTTPRequestHandler
   import json
   
   class MockAPIHandler(BaseHTTPRequestHandler):
       def do_GET(self):
           if self.path == '/v1/facility/nsls2/cycles/current':
               response = {'cycle': '2025-2', 'start_date': '2025-02-01', 'end_date': '2025-02-28'}
               self.send_response(200)
               self.send_header('Content-type', 'application/json')
               self.end_headers()
               self.wfile.write(json.dumps(response).encode())
           else:
               self.send_response(404)
               self.end_headers()
       
       def log_message(self, format, *args):
           pass  # suppress logs
   
   server = HTTPServer(('127.0.0.1', 8080), MockAPIHandler)
   server.serve_forever()
   MOCK_API_EOF
   MOCK_API_PID=$!
   disown
   echo "MOCK_API_PID=${MOCK_API_PID}" >> $GITHUB_ENV
   ```

3. **Port forward 80 → 8080** (in same step, after mock server starts):
   ```bash
   sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -d 127.0.0.1 -j REDIRECT --to-port 8080 || true
   ```

4. **Cleanup** mock server in "Stop mock API server" step:
   ```bash
   if [ -n "${MOCK_API_PID:-}" ]; then
     kill -SIGINT "${MOCK_API_PID}" 2>/dev/null || true
   fi
   ```

This allows profiles to make HTTP calls to the mocked API without blocking on network errors.
