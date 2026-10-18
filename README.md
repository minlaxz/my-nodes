# my-nodes

Services that run as nodes on the Mesh (a headscale tailnet). Each stack is a Docker Compose
project with a sing-box container that joins the tailnet, plus the service itself.

Split out of [minlaxz/nekohasekai](https://github.com/minlaxz/nekohasekai) (PR #62 there; see
`docs/adr/0004-nodes-live-in-my-nodes.md` in nekohasekai). The only link back is the headscale
control URL and a pre-auth key in each stack's `.env`.

## Stacks

| Stack | What it runs | Where |
|---|---|---|
| [`atuin/`](atuin/) | Atuin shell-history sync server + Postgres, on the tailnet at `:8888` | Any host |
| [`ccm-ocm/`](ccm-ocm/) | sing-box `ccm`/`ocm`: Claude Code and Codex multiplexers on `:4200`/`:4201` | Own VPS |
| [`vllm/`](vllm/) | vLLM serving `Qwen/Qwen3-VL-8B-Instruct` on `:8000` | EC2 g6.xlarge |

## Setup

In a stack's folder:

```sh
cp .env.sample .env   # fill in the headscale control URL, pre-auth key, hostname
docker compose up -d
```

Never commit `.env`; only `.env.sample` belongs in git. See each stack's README for details.

## Known manual steps

The vLLM node's EC2 data volume is attached by hand within the 5-minute `userdata.sh` window,
and snapshotted by hand for each availability zone. Tailscale state persists on
`/data/tailscale`, so the node keeps one name and IP across boots. Background:
`docs/research/magicdns-headscale.md` in nekohasekai.
