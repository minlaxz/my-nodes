# host-node

One Mesh node for a whole host, instead of one sing-box sidecar per stack. The `ts` container runs
with `network_mode: host`, so a tailnet hit on `<TSN_HOSTNAME>:<port>` goes to `127.0.0.1:<port>`
on the host. Any stack that publishes `127.0.0.1:<port>` is then reachable over the tailnet.

## Files

| File | Role |
|---|---|
| `docker-compose.yaml` | The host node: one sing-box container, host network. |
| `config.json` | sing-box config template. `${TSN_*}` placeholders, port allowlist. |
| `entrypoint.sh` | Fills placeholders from env, `sing-box check`, `sing-box run`. |
| `whoami.yaml` | Demo service (`traefik/whoami`) on `127.0.0.1:10001`. Rule 1. |
| `traefik-proxy.yaml` | Traefik on `127.0.0.1:20000`, creates `traefik-proxy-net`. Rule 2. |
| `traefik-whoami.yaml` | Demo backend `whoami-traefik`, no published port. Path `/whoami-traefik` or subdomain. |
| `caddy-proxy.yaml` | Caddy (`caddy-docker-proxy`) on `127.0.0.1:20000`, creates `caddy-proxy-net`. Rule 2, alternative to Traefik. |
| `caddy-whoami.yaml` | Demo backend `whoami-caddy`, no published port. Path `/whoami-caddy` or subdomain. |
| `.env.sample` | Copy to `.env`. |
| `tailscale-data/` | Tailscale state. Keeps the node's name and IP across restarts. |

## Setup

```sh
cp .env.sample .env   # fill in TSN_AUTH_KEY, TSN_CONTROL_URL, TSN_HOSTNAME, TSN_BASE_DOMAIN
docker compose up -d && docker compose logs -f
docker compose -f whoami.yaml up -d                                                 # rule 1 demo
docker compose -f traefik-proxy.yaml up -d && docker compose -f traefik-whoami.yaml up -d   # rule 2, Traefik
# or
docker compose -f caddy-proxy.yaml up -d && docker compose -f caddy-whoami.yaml up -d       # rule 2, Caddy
```

Run one proxy at a time; both bind `127.0.0.1:20000`. From another node on the Mesh:

```sh
curl http://<TSN_HOSTNAME>:10001                                  # whoami, straight to loopback
curl http://<TSN_HOSTNAME>:20000/whoami-traefik                   # path, through Traefik
curl http://<TSN_HOSTNAME>:20000/whoami-caddy                     # path, through Caddy
curl http://whoami-traefik.<TSN_HOSTNAME>.<TSN_BASE_DOMAIN>:20000 # subdomain, needs the DNS records below
curl http://whoami-caddy.<TSN_HOSTNAME>.<TSN_BASE_DOMAIN>:20000
```

## Port plan

Two route rules in `config.json`, both `ts-ep` to `127.0.0.1:<same port>`. The split is by intent,
sing-box treats them the same.

| Rule | Tailnet port | What listens on the host | Add a service |
|---|---|---|---|
| 1 | `9999`, `10000`, `10001-10099` | Anything bound to `127.0.0.1:<port>`: a systemd service, or a container stack with `127.0.0.1:<port>:<cport>`. No shared Docker network, no proxy. | Bind to a free port in the range. No `config.json` change. |
| 2 | `20000` | A reverse proxy: Traefik (`traefik-proxy.yaml`) or Caddy (`caddy-proxy.yaml`). Backends sit on the proxy's Docker network with no published port; the proxy picks one by `Host` header, path or TLS SNI. | Join the proxy network, add labels (see `traefik-whoami.yaml`, `caddy-whoami.yaml`). No `config.json` change. |

```
ts-ep:9999, :10000, :10001-10099 ─▶ 127.0.0.1:<same port> ─▶ service (host process or container)

ts-ep:20000 ─▶ 127.0.0.1:20000 ─▶ reverse proxy ─▶ backend A   (routed by Host / SNI,
                                                ─▶ backend B    shared Docker network,
                                                ─▶ backend N    nothing bound on loopback)
```

Nothing should bind `127.0.0.1` in `10001-10099` except services meant for the tailnet. Every
port in a listed range is open to the tailnet whether or not a service is there.

Ports outside both rules need a `config.json` edit and `docker compose up -d --force-recreate`.

## How routing works

```
tailnet peer: curl <TSN_HOSTNAME>:9998
      │
      ▼
ts-ep (userspace tailscale inside sing-box, owns <tailnet-ip>)
      │
      ▼
route.rules, first match wins
      │
      ├─ port in rule 1 or 2 ─▶ action: route, outbound: direct-out,
      │                         override_address: 127.0.0.1
      │                         ─▶ 127.0.0.1:<port> on the host ─▶ service
      │
      └─ anything else ───────▶ action: reject ─▶ TCP RST
                                client: curl: (56) Recv failure: Connection reset by peer
```

1. **Only listed ports pass.** `port` and `port_range` in the first two rules are the allowlist.
   Anything else falls through to the reject rule.
2. **The last rule rejects everything else from `ts-ep`.** It is the default-deny. Rule order
   matters: sing-box takes the first matching rule, so the reject rule stays last.
3. **`override_address` is what makes loopback services reachable.** The packet arrives with
   destination `<tailnet-ip>:<port>` (the node's tailnet IP). `direct-out` would dial that address on
   the host, where nothing listens: the tailnet IP exists only inside sing-box's userspace
   tailscale, not on a host interface. `override_address` swaps the destination IP for
   `127.0.0.1` and keeps the port, so a service bound to `127.0.0.1:<port>` answers.

## Reverse proxy on 20000

Both proxies watch the Docker socket and build routes from container labels. A new backend needs
no proxy restart and no proxy config file. Pick one:

| | Traefik (`traefik-proxy.yaml`) | Caddy (`caddy-proxy.yaml`) |
|---|---|---|
| Network | `traefik-proxy-net` | `caddy-proxy-net` |
| Opt-in | `traefik.enable: "true"` (`exposedByDefault=false`) | any `caddy*` label |
| Route by path | `traefik.http.routers.<n>.rule`: PathPrefix(`/x`) | `caddy_0: :80`, `caddy_0.handle_path: /x*`, `caddy_0.handle_path.reverse_proxy: "{{upstreams 80}}"` |
| Route by subdomain | Host(`x.<TSN_HOSTNAME>.<TSN_BASE_DOMAIN>`) | `caddy_1: http://x.<TSN_HOSTNAME>.<TSN_BASE_DOMAIN>`, `caddy_1.reverse_proxy: "{{upstreams 80}}"` |
| Both | join with `\|\|` in one rule | `caddy_0` and `caddy_1`, one site each |
| Backend port | auto when the image exposes one port, else `traefik.http.services.<n>.loadbalancer.server.port` | `{{upstreams <port>}}` |
| TLS by SNI | TCP router, `HostSNI()`, in the stock image | needs a custom build with `caddy-l4` |

Caddy notes: every backend that uses `:80` adds its block to the same site, so path routes from many
containers merge into one listener. `handle_path` strips the prefix before proxying
(`/whoami-caddy/x` reaches the backend as `/x`); use `handle` to keep it. Subdomain sites need the
`http://` prefix, or Caddy tries HTTPS on 443 and ACME for the name. `CADDY_INGRESS_NETWORKS` is
set because auto-detection of the upstream network is unreliable. Both proxies ignore the
`:20000` in the `Host` header.

Add a backend (Traefik):

```yaml
services:
  app:
    image: ...
    networks: [traefik-proxy-net]          # no ports:
    labels:
      traefik.enable: "true"
      traefik.http.routers.app.rule: PathPrefix(`/app`) || Host(`app.${TSN_HOSTNAME:?}.${TSN_BASE_DOMAIN:?}`)
networks:
  traefik-proxy-net:
    external: true
```

Add a backend (Caddy):

```yaml
services:
  app:
    image: ...
    networks: [caddy-proxy-net]            # no ports:
    labels:
      caddy_0: :80
      caddy_0.handle_path: /app*
      caddy_0.handle_path.reverse_proxy: "{{upstreams 80}}"
      caddy_1: http://app.${TSN_HOSTNAME:?}.${TSN_BASE_DOMAIN:?}
      caddy_1.reverse_proxy: "{{upstreams 80}}"
networks:
  caddy-proxy-net:
    external: true
```

`${TSN_HOSTNAME}` and `${TSN_BASE_DOMAIN}` come from `.env` in this directory; compose fills them
in. Path rules work with no DNS setup. Subdomain rules need the records below. Rootless Docker:
set `DOCKER_SOCK=/run/user/<uid>/docker.sock` in `.env`.

## DNS for subdomains

MagicDNS resolves `<TSN_HOSTNAME>.<TSN_BASE_DOMAIN>` only. Each subdomain needs an A record to the
node's tailnet IP, served by Headscale to every client. On the control plane, in the file named by
`dns.extra_records_path` in `headscale-config.yaml` (`headscale-dns_records.json` in this setup):

```json
[
  { "name": "whoami-traefik.<TSN_HOSTNAME>.<TSN_BASE_DOMAIN>", "type": "A", "value": "<tailnet-ip>" },
  { "name": "whoami-caddy.<TSN_HOSTNAME>.<TSN_BASE_DOMAIN>",   "type": "A", "value": "<tailnet-ip>" }
]
```

Headscale watches the file and pushes changes without a restart. One record per name; no wildcards.
Check from a client:

```sh
dig +short whoami-caddy.<TSN_HOSTNAME>.<TSN_BASE_DOMAIN> @100.100.100.100
```

Test a subdomain route before the record exists:

```sh
curl -H 'Host: whoami-caddy.<TSN_HOSTNAME>.<TSN_BASE_DOMAIN>' http://<TSN_HOSTNAME>:20000
```

HTTP path by hostname, Traefik shown:

```
curl http://whoami-traefik.<TSN_HOSTNAME>.<TSN_BASE_DOMAIN>:20000    curl http://app.<TSN_HOSTNAME>.<TSN_BASE_DOMAIN>:20000
      │                                                                 │
      └──────── headscale extra_records: one A record each ─▶ <tailnet-ip>
                                 │
                                 ▼
              ts-ep ─▶ port 20000 allowed ─▶ 127.0.0.1:20000 (Traefik)
                                                   │ Host header
                                    ┌──────────────┴──────────────┐
                                    ▼                             ▼
                          whoami-traefik:80                     app:80
                         (shared docker network,       (shared docker network,
                          no published port)            no published port)
```

- The proxy is the only container that publishes a port (`127.0.0.1:20000:80`). Backends join its
  network and publish nothing. sing-box stays in the host netns and only ever dials
  `127.0.0.1:20000`.
- TLS backends route by SNI without terminating TLS: Traefik TCP router with `HostSNI(...)`.
- Trade-off: a proxy restart drops every backend at once. Rule 1 services are unaffected.

## Notes

- The port allowlist matters. Without it, every port bound to `127.0.0.1` on the host is open
  to the tailnet. Tailnet traffic to any other port is rejected.
- Publish services on `127.0.0.1`, not `0.0.0.0`. A plain `10001:80` also opens the port on the
  host's public interface.
- Restarting the `ts` container drops tailnet access to every service on the host.
- `auto_detect_interface` is `false` on purpose. When `true`, sing-box binds its sockets to the
  host's main interface (for example `ens4`). The node then cannot reach a headscale that runs in
  Docker on the same host through the public IP, and never registers. The node needs no TUN, so the
  setting is not needed.
- Env values must not contain `|`, `&`, or `"`; the entrypoint uses `sed`.
