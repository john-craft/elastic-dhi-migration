# Elasticsearch and Kibana 

This repo contains instructions on how to take the Docker Compose 
file from the [Elastic blog post](https://www.elastic.co/blog/getting-started-with-the-elastic-stack-and-docker-compose) 
and migrate to use Docker Hardened Images (DHI).

## Migrating to the Docker Hardened Image (DHI)

The `es01` service was switched from the stock
`docker.elastic.co/elasticsearch/elasticsearch` image to the hardened image
`demonstrationorg/dhi-elasticsearch:8`. The DHI image is a minimal, locked-down
build: it runs as a different non-root user and ships a stripped entrypoint that
does **not** include the stock image's conveniences. That broke startup in three
stages, each fixed below.

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

### 3. Recurring entitlement WARNs (cosmetic)

The hardened image's entitlement policy doesn't grant the `x-pack-security`
hot-reload file watcher `read` access to the config files it polls
(`elasticsearch.yml`, `jvm.options[.d]`). Config still loads correctly at
startup; these are harmless `WARN`s emitted every few seconds. The
`config/elasticsearch.yml` raises that one logger to `ERROR` to keep the log
readable.

### 4. Data volume not writable (`failed to obtain node locks`)

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

- `docker-compose.yaml` — stack definition; `es01` uses the DHI image and mounts
  `config/elasticsearch.yml`. The `setup` service mounts `esdata01` to seed and
  chown the data directory before `es01` starts.
- `config/elasticsearch.yml` — Elasticsearch settings (previously passed as env
  vars).
- `.env` — versions, passwords, ports, memory limits.

## Usage

```bash
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
