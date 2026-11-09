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
| `whoami.yaml` | Demo service (`traefik/whoami`) on `127.0.0.1:10001`. |
| `.env.sample` | Copy to `.env`. |
| `tailscale-data/` | Tailscale state. Keeps the node's name and IP across restarts. |

## Setup

```sh
cp .env.sample .env   # fill in TSN_AUTH_KEY, TSN_CONTROL_URL, TSN_HOSTNAME
docker compose up -d && docker compose logs -f
docker compose -f whoami.yaml up -d
```

From another node on the Mesh:

```sh
curl http://<TSN_HOSTNAME>:10001
```

## Port plan

Two route rules in `config.json`, both `ts-ep` to `127.0.0.1:<same port>`. The split is by intent,
sing-box treats them the same.

| Rule | Tailnet port | What listens on the host | Add a service |
|---|---|---|---|
| 1 | `9999`, `10000`, `10001-10099` | Anything bound to `127.0.0.1:<port>`: a systemd service, or a container stack with `127.0.0.1:<port>:<cport>`. No shared Docker network, no proxy. | Bind to a free port in the range. No `config.json` change. |
| 2 | `20000` | A reverse proxy (Traefik, Caddy). Backends sit on the proxy's Docker network with no published port; the proxy picks one by `Host` header or TLS SNI. | Add the container to the proxy's network with its labels. No `config.json` change. |

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

Traefik example. Caddy with `caddy-docker-proxy` works the same way.

```
curl http://whoami.<TSN_HOSTNAME>.<base-domain>:20000    curl http://whoami2.<TSN_HOSTNAME>.<base-domain>:20000
      │                                                        │
      └──────── headscale extra_records: *.<TSN_HOSTNAME>.<base-domain> ─▶ <tailnet-ip>
                                 │
                                 ▼
              ts-ep ─▶ port 20000 allowed ─▶ 127.0.0.1:20000 (Traefik)
                                                   │ Host header
                                    ┌──────────────┴──────────────┐
                                    ▼                             ▼
                               whoami:80                     whoami2:80
                         (shared docker network,       (shared docker network,
                          no published port)            no published port)
```

- Traefik is the only container that publishes a port (`127.0.0.1:20000:80`). Backends join
  Traefik's Docker network, carry `traefik.http.routers.<name>.rule=Host(...)` labels, publish
  nothing. sing-box stays in the host netns and only ever dials `127.0.0.1:20000`.
- Hostnames need DNS. MagicDNS resolves `<TSN_HOSTNAME>.<base-domain>` only, so add an A record per
  backend (or a wildcard, if your Headscale version accepts it) to Headscale's `extra_records`
  file, pointing at the node's tailnet IP. Without DNS, `curl -H 'Host: whoami.<TSN_HOSTNAME>.<base-domain>' http://<TSN_HOSTNAME>:20000`
  still works, or use `PathPrefix` rules.
- TLS backends route by SNI without terminating TLS: Traefik TCP router with `HostSNI(...)`.
- Trade-off: a Traefik restart drops every backend at once. Rule 1 services are unaffected.

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
