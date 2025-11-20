## Veilgate – Quickstart (Docker + docker-compose)

This document explains how to run a released Veilgate package using Docker and
docker-compose.

### Requirements

- Docker (Desktop / Engine) installed.
- docker-compose available (either `docker compose` or the standalone
  `docker-compose` binary).

### Package contents

After unpacking the `veilgate-<VERSION>.zip` archive, you should see:

- `docker-compose.yml` – services for Veilgate + demo upstream.
- `config/veilgate.yaml` – gateway configuration file.
- `docs/QUICKSTART.md` – this quickstart.
- `VERSION` – release version.
  (Images are pulled from a registry, e.g. GitHub Container Registry).

### Running the stack

1. Unpack the archive:

```bash
unzip veilgate-<VERSION>.zip
cd veilgate-<VERSION>
```

2. Start the services:

```bash
docker-compose up
```

The first run may take a while as Docker pulls or builds images.

3. Verify that everything works:

- Gateway (demo route behind the API gateway):

```bash
curl -H 'X-API-Key: demo-secret-key' http://localhost:8080/
```

You should see a JSON response from the demo upstream.

- Dashboard and admin endpoints:
  - `http://localhost:9090/dashboard` – built-in Veilgate dashboard.
  - `http://localhost:9090/healthz` – liveness endpoint.
  - `http://localhost:9090/readyz` – readiness endpoint.
  - `http://localhost:9090/metrics` – Prometheus metrics.

### Configuration

You can change the gateway configuration in:

- `config/veilgate.yaml`

Typical things you may want to adjust:

- `upstreams` – addresses of your backend services,
- `routes` – paths, methods and upstream assignment,
- `security.api_keys` – API keys (replace `demo-secret-key` with your own),
- `rate_limit` – limits per route / IP / API key.

### Reloading configuration without restart (hot reload)

Veilgate supports **configuration hot reload** – instead of restarting the
process or container, you can reload `veilgate.yaml` on the fly. This means:

- existing connections are served to completion,
- new requests start using the **new route tree** and settings,
- you avoid losing metrics or logs due to process restarts.

#### How it works under the hood

- The `veilgate` process is started with:

  ```bash
  veilgate serve -config /etc/veilgate/config.yaml
  ```

- In the container from the release package (`docker-compose.yml`), the
  `config/veilgate.yaml` file from the host is mounted as:

  ```yaml
  volumes:
    - ./config/veilgate.yaml:/etc/veilgate/config.yaml:ro
  ```

- When the process receives a **`SIGHUP`** signal it:
  - reloads the configuration file from the `-config` path,
  - rebuilds from scratch:
    - the upstream registry,
    - API key and JWT configuration,
    - rate limit configuration,
    - the **HTTP router** (complete route tree),
  - atomically swaps the HTTP handler to the new one without restarting the
    server.

If the new configuration file contains an error (e.g. a missing upstream),
reload fails and the **previous configuration remains active**.

#### Hot reload with Docker + docker-compose

Assume you started Veilgate using the quickstart above (with the
`docker-compose.yml` from the release package) where the `veilgate` service is
defined as:

```yaml
services:
  veilgate:
    container_name: veilgate-release
    image: ghcr.io/blackmasoon/veilgate:<VERSION>
    volumes:
      - ./config/veilgate.yaml:/etc/veilgate/config.yaml:ro
```

1. **Modify the configuration on the host**

   Update `config/veilgate.yaml` (for example, add a new route or change the
   upstream):

   ```yaml
   routes:
     - id: example-route
       path: /
       method: GET
       upstream_id: example-api
     - id: extra-route
       path: /extra
       method: GET
       upstream_id: example-api
   ```

   The file is mounted read‑only inside the container, but the **host** controls
   its contents, so editing it on the host is enough.

2. **Send a `SIGHUP` signal to the Veilgate process in the container**

   The default container name in the quickstart is `veilgate-release`, so you
   can trigger a reload with:

   ```bash
   docker kill -s HUP veilgate-release
   ```

   If you use `docker compose` without an explicit `container_name`, you can use
   the service name instead:

   ```bash
   docker compose kill -s HUP veilgate
   ```

3. **Verify new routes without restarting**

   Existing routes should behave as before:

   ```bash
   curl -H 'X-API-Key: demo-secret-key' http://localhost:8080/
   ```

   New routes should be available after the hot reload:

   ```bash
   curl -H 'X-API-Key: demo-secret-key' http://localhost:8080/extra
   ```

#### When a full container restart is still required

Hot reload applies to logical configuration (routes, upstreams, security,
rate limiting). **Process-level changes** still require a restart, for example
when you:

- change `server.listen_address` or the admin port (`admin.listen_address`),
- upgrade the Docker image to a new Veilgate version,
- change how the process is started (`ENTRYPOINT`, flags) in
  `docker-compose.yml`.

In those cases, use a classic restart:

```bash
docker-compose down
docker-compose up
```

### Stopping the stack

To stop the services:

```bash
docker-compose down
```


