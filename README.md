# Tollhouse Edge

**One Caddy in front of every project on your server.** Automatic HTTPS, one place for certificates, one place to add a domain.

This is the companion of [Tollhouse](https://github.com/your-name/tollhouse) (a web + sandboxed-worker framework for pay-per-use services), but it works with anything that runs in Docker and speaks HTTP.

## Why

If every project ships its own Caddy, only one of them can own ports 80 and 443. The edge solves that: it is the only container that publishes those ports, terminates TLS for all domains, and forwards each domain to the right backend container over a shared Docker network.

```
Internet --443--> edge (Caddy) --network "edge"--> myapp-web:3000
                                              \--> other-web:3000
```

Backends are reached by **network alias**, not by published ports, so nothing but the edge is exposed to the internet. Sidecar containers such as workers, databases and caches stay on their project's private network and never join the edge network.

## Files

| File | Purpose |
|---|---|
| `Caddyfile` | One site block per project, plus the shared `(common)` snippet. |
| `docker-compose.yml` | The Caddy container, certificate volumes, the external network. |
| `README.ai.md` | Short operating notes for AI assistants working on your server or projects. |

The `(common)` snippet enables compression and answers `404` for `/internal/*`, so worker APIs can never be reached from outside even by mistake. Import it in every site block.

## Setup

Requirements: a server with Docker and the Compose plugin, ports 80 and 443 (TCP and UDP for HTTP/3) open, and a DNS A/AAAA record per domain pointing at the server.

```
sudo git clone https://github.com/your-name/tollhouse-edge /opt/edge
cd /opt/edge
docker network create edge          # once per server
# edit Caddyfile: replace the example blocks with your domains and aliases
docker compose up -d
docker compose logs -f caddy        # wait for "certificate obtained successfully"
```

## Connect a project

In the project's `docker-compose.yml`, attach **only the web-facing container** to the edge network with a unique alias, and keep the project's default network:

```yaml
services:
  web:
    networks:
      default: {}               # keep this, otherwise your other services cannot reach web
      edge:
        aliases: [myapp-web]    # must be unique per server

networks:
  edge:
    external: true
    name: edge
```

Then add a block to the `Caddyfile`:

```
app.example.com {
	import common
	reverse_proxy myapp-web:3000
}
```

and reload without downtime:

```
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

If you rename the network, change it in three places: `docker network create`, this compose file, and every project that joins it.

## Migrating from a project with its own Caddy

1. Back up the project's `docker-compose.yml`.
2. Remove its `caddy` service and the `caddy_data` / `caddy_config` volumes.
3. Attach its web container to the edge network as shown above.
4. `docker compose up -d --remove-orphans` in the project (this frees ports 80/443; expect a short outage).
5. If the project has a firewalled worker that pins the web container's IP, restart the worker afterwards.
6. Start the edge (`docker compose up -d`). The new Caddy fetches fresh certificates, which takes up to a minute.
7. Check: `curl -sI https://app.example.com | head -1` returns 200 and `curl -s -o /dev/null -w "%{http_code}" https://app.example.com/internal/x` returns 404.

Rollback: stop the edge, restore the project's backup compose file, `docker compose up -d`.

## Operations

```
docker compose logs -f caddy                       # certificates, errors
docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile
docker compose pull && docker compose up -d        # update Caddy
```

Certificates live in the `caddy_data` volume. Back it up if you want to avoid re-issuing them after a server move (Let's Encrypt rate limits apply).

## License

MIT. See `LICENSE`.
