# giangmichaelphan-tunnel

A standalone Cloudflare Tunnel project. It runs a single `cloudflared` connector that publishes the home lab services (portfolio, signage, Portainer) to Cloudflare, so no inbound router port forwards are required.

This project is intentionally separate from the application stacks. The tunnel connector serves services from multiple projects, so it does not belong to any one app.

## Overview

`cloudflared` makes an **outbound** connection to Cloudflare and holds it open. Cloudflare terminates TLS at the edge with a valid certificate and forwards requests down that connection to the origin services on this host, over plain HTTP on the LAN.

Because the connector dials out:

- No inbound router port forward is needed.
- The server's public IP is irrelevant, so no dynamic DNS is needed. AT&T can change the public IP without breaking anything; the connector re-establishes itself using the tunnel token, not the IP.

### HTTP to the origin; Cloudflare handles HTTPS

Applications published through this tunnel only need to serve **plain HTTP on the LAN**. Cloudflare terminates TLS at its edge with a managed certificate and forwards the request to the origin over HTTP. In practice that means:

- The origin service (for example the portfolio) is configured to listen on **HTTP only** (its published HTTP port, for example `8081`), and its public URLs are served by Cloudflare as HTTPS.
- **The origin does not need a self-signed (or any) HTTPS certificate for public access.** The certificate your browser sees is Cloudflare's edge certificate, not the origin's.
- Removing origin HTTPS also removes the whole class of "self-signed certificate" and "certificate name mismatch" problems for public traffic, along with the Kestrel `https://*:443` binding and `.pfx` certificate password.

Local development is the exception: when you run an app directly (not behind the tunnel) and want to exercise HTTPS locally, you still need a **development certificate** (for example `dotnet dev-certs https`). That certificate is only for `localhost` development and is not used in production, where Cloudflare provides TLS.

## Files

- `docker-compose.yml` - the `cloudflared` service definition.
- `variables.env` - holds `TUNNEL_TOKEN` (secret, git-ignored; never commit it).
- `.gitignore` - keeps `variables.env` out of source control.

## Tunnel

- Tunnel name: `gp-signage`
- Tunnel ID: `05e8f5ca-1409-4a6a-ba12-8cb8229389c5`
- Connector network mode: `host` (so the connector can reach origins bound to the host on `localhost`)

The `cloudflared` service runs with `network_mode: host`. This is deliberate: it removes the Docker bridge NAT hop between the connector and the host, which otherwise causes intermittent timeouts (HTTP 522) when the connector reaches a host-published port. With host networking, `localhost:<port>` inside the connector is the host's published port.

## Public hostnames

Hostnames and their origin services are configured on the tunnel (Cloudflare Zero Trust > Networks > Tunnels > `gp-signage` > Public Hostnames). The connector receives this configuration remotely; it is not stored in this project.

Origins use **HTTP** wherever the service allows it. Only Portainer is reached over HTTPS, because Portainer itself serves TLS on `9443`; that single route needs `No TLS Verify`.

| Hostname | Origin service | Notes |
| --- | --- | --- |
| `giangmichaelphan.com` | `http://localhost:8081` | Portfolio. Public. Origin is HTTP. |
| `www.giangmichaelphan.com` | `http://localhost:8081` | Portfolio. Public. Origin is HTTP. |
| `signage.giangmichaelphan.com` | `http://localhost:8080` | Signage app. Origin is HTTP. |
| `portainer.giangmichaelphan.com` | `https://localhost:9443` | Portainer, gated by Cloudflare Access. Origin is HTTPS (self-signed); requires `No TLS Verify`. |

Cloudflare stores the apex `giangmichaelphan.com` as a **CNAME-flattened** record to the tunnel target (`*.cfargotunnel.com`); the other hostnames are normal tunnel CNAMEs. None of them point at the server's public IP.

### Portainer and `No TLS Verify`

The portfolio and signage are reached over HTTP, so they need no origin certificate at all. Portainer is the exception: it serves its own **self-signed** certificate on port `9443` (issued for `0.0.0.0`), which the connector cannot validate against `localhost`. Set **No TLS Verify = ON** for the `portainer.giangmichaelphan.com` route only. This affects only the LAN hop between the connector and the origin; the public leg still uses Cloudflare's certificate and Cloudflare Access still requires login. If Portainer were configured to serve plain HTTP on an unused port, this exception would not be needed.

## Prerequisites

- Docker with the Compose plugin.
- A Cloudflare account that owns `giangmichaelphan.com`, with the tunnel `gp-signage` created and its public hostnames configured.

## Configure the token

The connector authenticates with a **Cloudflare Tunnel token**. The token is a secret; it is read from `variables.env`, never inlined in the compose file and never committed.

`variables.env` (git-ignored), a single line with no surrounding quotes:

```sh
TUNNEL_TOKEN=<tunnel token>
```

Get the token from Cloudflare Zero Trust > Networks > Tunnels > `gp-signage` > Configure. The `cloudflared` image reads `TUNNEL_TOKEN` from the environment automatically, so the compose `command` does not pass `--token` explicitly.

> Do not inline the token in `docker-compose.yml` (for example as `--token ${TUNNEL_TOKEN}`). Compose substitutes an empty string if the variable is not set, which leaves a dangling `--token`; cloudflared then prints usage and exits, and the container restart-loops. Loading the token through `env_file` avoids this.

If a token is exposed (committed, pasted, or shared), rotate it in Cloudflare and update `variables.env`.

## Run

From this directory:

```sh
docker compose up -d
```

Recreate after changing `variables.env` or `docker-compose.yml`:

```sh
docker compose up -d --force-recreate
```

## Verify

Confirm the connector registered with Cloudflare:

```sh
docker logs cloudflared 2>&1 | grep -i "Registered tunnel connection"
```

You want several `Registered tunnel connection connIndex=...` lines. `Unauthorized` or repeated `Register tunnel error` means the token is wrong or stale.

Confirm the connector is on host networking:

```sh
docker inspect cloudflared --format '{{.HostConfig.NetworkMode}}'   # expect: host
```

Confirm each service responds through Cloudflare:

```sh
curl -sI https://giangmichaelphan.com        | head -1
curl -sI https://www.giangmichaelphan.com    | head -1
curl -sI https://signage.giangmichaelphan.com | head -1
curl -sI https://portainer.giangmichaelphan.com | head -1
```

Each should return a `200` (Portainer may redirect to a Cloudflare Access login first).

## Common problems

| Symptom | Cause | Fix |
| --- | --- | --- |
| Container `Restarting (0)` and logs show cloudflared help text | Token empty or `--token` passed with no value | Use `env_file` and `command: tunnel --no-autoupdate run` (no inline `--token`); confirm `TUNNEL_TOKEN` is set |
| Logs show `Unauthorized` / register errors | Token wrong, stale, or rotated | Copy the current token from the tunnel page into `variables.env`, then `--force-recreate` |
| HTTP 522 (connection timed out) for a hostname | Connector cannot reach the origin | Ensure `network_mode: host` and the ingress origin is `http://localhost:<port>` for that service |
| 502 for `portainer.giangmichaelphan.com` | Self-signed cert not trusted | Set `No TLS Verify = ON` on that route only |
| All hostnames fail at once | Connector is not running/registered | Check `docker logs cloudflared`; fix the token; ensure exactly one `cloudflared` container exists |
| Duplicate/intermittent failures | More than one connector registered for the tunnel | Keep a single `cloudflared` container; remove any others (`docker ps -a | grep cloudflared`) |

## Operational notes

- Run exactly **one** `cloudflared` container for the tunnel. Multiple connectors with the same token are load balanced across potentially different network topologies and cause intermittent failures.
- The connector configuration (hostnames and origins) lives in Cloudflare, not here. To change routing, edit the tunnel's public hostnames in the dashboard; the connector picks up the change remotely.
- **Origins should serve HTTP.** Cloudflare handles public HTTPS, so applications behind the tunnel do not need to listen on `443` or present a certificate. Point the ingress at the app's HTTP port (`http://localhost:<port>`). Keep a development certificate only for running apps locally outside the tunnel (`dotnet dev-certs https`).
- Because every public hostname resolves to the tunnel and nothing points at the server's public IP, **dynamic DNS and inbound router port forwards are not required**. The connector tolerates public IP changes on its own.
- The connector's token is the only secret in this project. Keep `variables.env` out of source control and rotate the token if it is ever exposed.
