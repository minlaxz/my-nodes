# atuin

Self-hosted [Atuin](https://atuin.sh) sync server: shell history from every device, end-to-end
encrypted, synced over the tailnet.

`docker-compose.yaml`, three containers:

- `ts` — sing-box with a `tailscale` endpoint. Owns the network namespace.
- `atuin` — `network_mode: service:ts`, so the server's `:8888` is `127.0.0.1:8888` inside `ts`.
- `db` — Postgres. Data lives in `./database`. Back this up.

`<atuin-host>` below is the node's tailnet IP (`100.x.y.z`, from `tailscale status` or
`headscale nodes list`) or its MagicDNS name (`<hostname>.<base_domain>`, where `base_domain` is
set in the headscale config).

The server has no web UI: `http://<atuin-host>:8888/` only returns a JSON greeting. All use goes
through the `atuin` CLI.

Browsers with secure DNS (DoH) on cannot resolve MagicDNS names. Use the tailnet IP, or turn secure DNS off.

## Client: first device

1. Install the client.
   ```sh
   curl --proto '=https' --tlsv1.2 -LsSf https://setup.atuin.sh | sh   # or: brew install atuin
   ```
   The installer adds `eval "$(atuin init zsh)"` (or the bash/fish equivalent) to your shell rc
   file. Add it by hand if it is missing.
2. Point the client at this server in `~/.config/atuin/config.toml`.
   ```toml
   sync_address = "http://<atuin-host>:8888"
   ```
3. Register. `-u` is the username, `-e` the email. It prompts for a password.
   ```sh
   atuin register -u <username> -e <email>
   ```
4. Print the encryption key and store it somewhere safe, such as a password manager.
   ```sh
   atuin key
   ```
   History is encrypted with this key before it leaves the device. The server cannot recover it:
   lose the key and the synced history is unreadable. Every other device needs it to log in.
5. Import the shell's existing history and upload it.
   ```sh
   atuin import auto
   atuin sync
   ```

Registration is open (`ATUIN_OPEN_REGISTRATION: "true"`). Once every account exists, set it to
`"false"` in `docker-compose.yaml` and run `docker compose up -d`. Logging in still works.

## Client: each new device

1. Install the client and set `sync_address` (steps 1 and 2 above).
2. Log in. It prompts for the password and the key from `atuin key`.
   ```sh
   atuin login -u <username>
   ```
3. Pull the history from the server. To also keep this device's own history, import it first.
   ```sh
   atuin import auto   # optional: merge this device's existing shell history
   atuin sync
   ```
4. Check: `atuin history list | tail` shows commands from your other devices.

Atuin syncs on its own after that (`auto_sync`, on by default). Run `atuin sync` to sync now.

## Turn off the shell's own history file

Atuin records every command itself, so the shell's history file is no longer needed. Do this
only after `atuin sync` has succeeded on the device. Put the line at the **end** of the rc file,
after any framework (such as oh-my-zsh) that sets `HISTFILE`.

| Shell | rc file | Line |
|---|---|---|
| zsh | `~/.zshrc` | `unset HISTFILE` |
| bash | `~/.bashrc` | `unset HISTFILE` |
| fish | `~/.config/fish/config.fish` | `set -g fish_history ""` |

The shell keeps history in memory for the current session (so `!!` still works) but no longer
writes it to disk. Open a new shell to apply it. The old file, such as `~/.zsh_history`, stays
until you delete it.
