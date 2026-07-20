# Elasticsearch and Kibana 

This repo contains instructions on how to take the Docker Compose 
file from the [Elastic blog post](https://www.elastic.co/blog/getting-started-with-the-elastic-stack-and-docker-compose) 
and migrate to use Docker Hardened Images (DHI).

**Current versions:** Elasticsearch and Kibana **9.1** (DHI), with a matching
official `docker.elastic.co` image used only by the `setup` service (which needs
`curl` to run the cert-generation and password-bootstrap script).

## Migrating to the Docker Hardened Image (DHI)

Both the `es01` and `kibana` services were switched from the stock
`docker.elastic.co` images to hardened images:

- `demonstrationorg/dhi-elasticsearch:9.1`
- `demonstrationorg/dhi-kibana:9.1`

The DHI images are minimal, locked-down builds: they run as a different non-root
user, ship a stripped entrypoint that does **not** include the stock images'
conveniences, and do **not** bundle `curl` or `wget`. That broke startup in
several stages, each fixed below.

### 1. Certificate permissions (`AccessDeniedException: .../config/certs`)

The `setup` service generates the TLS certs and `chown`s them `root:root` with
`750`/`640` permissions. That worked on the stock image only because its
Elasticsearch user belongs to group `root` (gid 0). The DHI image runs as a
different non-root user that is **not** in gid 0, so it couldn't even list the
`certs` directory.

**Fix:** in `setup`, the cert tree is now made world-readable so any runtime UID
can read it:

```
find . -type d -exec chmod 755 {} \;
find . -type f -exec chmod 644 {} \;
```

> [!NOTE]
> `644` makes the demo TLS keys readable by any user inside the container —
> acceptable for this local/POC setup. For a locked-down variant, `chown` the
> certs to the image's actual runtime UID/GID and keep `750`/`640`.

### 2. Settings ignored → bootstrap checks failed

The stock image's entrypoint translates dotted `environment:` variables
(`discovery.type`, `xpack.security.*`, `node.name`, …) into Elasticsearch
settings. The DHI entrypoint does not, so every such variable was silently
ignored — ES fell back to multi-node discovery with no transport TLS and died on
the production bootstrap checks.

**Fix:** the equivalent settings now live in a real config file,
[`config/elasticsearch.yml`](config/elasticsearch.yml), mounted read-only into
`es01`. `discovery.type: single-node` now applies (which also skips the bootstrap
checks).

### 3. Kibana: no `curl` → healthcheck fails

The DHI Kibana image also lacks `curl`, so the original healthcheck
(`curl -s -I http://localhost:5601 | grep -q 'HTTP/1.1 302 Found'`) would
never pass, keeping Kibana in `starting` state indefinitely.

**Fix:** same `/dev/tcp` approach as es01, with a `start_period` to avoid
counting early failures:

```yaml
healthcheck:
  test: ["CMD-SHELL", "bash -c ': >/dev/tcp/127.0.0.1/5601'"]
  interval: 10s
  timeout: 10s
  retries: 120
  start_period: 30s
```

### 4. Kibana: cert path resolved relative to wrong working directory

The DHI Kibana 9.1 image installs under `/opt/kibana/kibana-9.1.x/` and
Node.js runs from that directory as its `cwd`. The original config passed a
relative path for the CA certificate:

```
ELASTICSEARCH_SSL_CERTIFICATEAUTHORITIES=config/certs/ca/ca.crt
```

Node resolved this to `/opt/kibana/kibana-9.1.x/config/certs/ca/ca.crt`,
which doesn't exist — the certs volume is mounted at `/usr/share/kibana/config/certs`.

**Fix:** use the absolute path:

```
ELASTICSEARCH_SSL_CERTIFICATEAUTHORITIES=/usr/share/kibana/config/certs/ca/ca.crt
```

### 5. Fleet encryption key not set

Kibana's Fleet plugin requires `xpack.encryptedSavedObjects.encryptionKey` to
store agent binary source metadata. Without it, Fleet logs a repeating warning
and cannot fully initialise.

**Fix:** pass the key via environment variable (the value is already in `.env`
as `ENCRYPTION_KEY`):

```
XPACK_ENCRYPTEDSAVEDOBJECTS_ENCRYPTIONKEY=${ENCRYPTION_KEY}
```

### 7. Recurring entitlement WARNs (cosmetic)

The hardened image's entitlement policy doesn't grant the `x-pack-security`
hot-reload file watcher `read` access to the config files it polls
(`elasticsearch.yml`, `jvm.options[.d]`). Config still loads correctly at
startup; these are harmless `WARN`s emitted every few seconds. The
`config/elasticsearch.yml` raises that one logger to `ERROR` to keep the log
readable.

### 8. Data volume not writable (`failed to obtain node locks`)

With config now loading correctly, `es01` died on startup with:

```
failed to obtain node locks, tried [/usr/share/elasticsearch/data]; maybe these
locations are not writable or multiple nodes were started on the same data path?
```

Despite the "multiple nodes" wording, this is a **permissions** problem (we run
`discovery.type: single-node`): the DHI runtime user couldn't write to its data
path. Docker seeds a fresh named volume with the ownership of the directory in
the image of the **first container to mount it**. `esdata01` was first mounted by
`es01` (the DHI image), so it inherited an ownership the non-root DHI user can't
write to.

**Fix:** the `setup` service (which uses the stock image, runs as root, and starts
before `es01`) now also mounts `esdata01` and explicitly chowns it to the runtime
user:

```
chown -R 1000:0 /usr/share/elasticsearch/data;
chmod 770 /usr/share/elasticsearch/data;
```

The earlier cert-permission pass was also scoped from `.` (the whole working
directory) down to `config/certs`, so it no longer re-chmods the entire data tree
on every `docker compose up`.

## Files

- `docker-compose.yaml` — stack definition; `es01` and `kibana` use DHI images.
  `es01` mounts `config/elasticsearch.yml`. The `setup` service (official image,
  needs `curl`) mounts `esdata01` to seed and chown the data directory before
  `es01` starts.
- `config/elasticsearch.yml` — Elasticsearch settings (previously passed as env
  vars).
- `.env` — versions, passwords, ports, memory limits. Copy `.env.example` to
  `.env` and adjust before first run.
- `.env.example` — template with all required variables and safe defaults.

## Usage

```bash
cp .env.example .env   # then edit passwords / ports as needed
docker compose up
```

Verify the cluster is up and authenticating:

```bash
curl -s --cacert config/certs/ca/ca.crt -u elastic:$ELASTIC_PASSWORD \
  https://localhost:9200/_cluster/health
```

## Quieting es01 logs

`es01` is verbose at startup and can drown out the rest of the stack in the
terminal. The compose file sets `attach: false` on `es01`, which stops Compose
from *streaming* its output during `docker compose up` while still **capturing**
it — `docker compose logs es01` works, and the healthcheck and other services are
unaffected. This is the intended tool for the job; keep it.

Other options, depending on the goal:

- **Declutter the terminal** — `attach: false` (current approach), or run detached
  and tail only what you care about:

  ```bash
  docker compose up -d
  docker compose logs -f kibana
  ```

- **Cap log disk usage** — add a logging driver limit to `es01` (rotation only;
  does not reduce verbosity):

  ```yaml
  logging:
    driver: json-file
    options:
      max-size: "10m"
      max-file: "3"
  ```

- **Genuinely quiet ES itself** — raise log levels in `config/elasticsearch.yml`,
  the same mechanism used for the entitlement logger, e.g.
  `logger.org.elasticsearch: WARN`. Use sparingly: most boot noise (GC, plugin
  loading, bootstrap) is normal and occasionally useful for debugging.
