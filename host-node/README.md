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
| `whoami.yaml` | Demo service (`traefik/whoami`) on `127.0.0.1:9999`. |
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
curl http://<TSN_HOSTNAME>:9999
```

## Add a service

1. Publish the service on loopback only, for example `127.0.0.1:8888:8888`.
2. Add its port to the `"port"` list in `config.json`.
3. Run `docker compose up -d --force-recreate`.

## Notes

- The port allowlist matters. Without it, every port bound to `127.0.0.1` on the host is open
  to the tailnet. Tailnet traffic to any other port is rejected.
- Publish services on `127.0.0.1`, not `0.0.0.0`. A plain `9999:80` also opens the port on the
  host's public interface.
- Restarting the `ts` container drops tailnet access to every service on the host.
- `auto_detect_interface` is `false` on purpose. When `true`, sing-box binds its sockets to the
  host's main interface (for example `ens4`). The node then cannot reach a headscale that runs in
  Docker on the same host through the public IP, and never registers. The node needs no TUN, so the
  setting is not needed.
- Env values must not contain `|`, `&`, or `"`; the entrypoint uses `sed`.
