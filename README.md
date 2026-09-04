# nmrac

A headless CLI for connecting to [Netmaker](https://netmaker.io) remote-access networks — what Netmaker Desktop does, without a GUI. Runs on **Linux and macOS**.

Netmaker Desktop (the Remote Access Client) is a graphical app. On a server, a container, or anything over SSH there is nothing to click. `nmrac` drives the same server API from a shell: SSO login in your browser, pick a network, and it renders a WireGuard config and brings the tunnel up with `wg-quick`.

```console
$ sudo nmrac connect

Networks (user: you@example.com)

  1) [off]  office-split-tunnel   gw gateway-a         
  2) [on ]  lab-network           gw gateway-b         10.20.30.8/32
  3) [off]  build-network         gw gateway-c         

Select network [1-3], or q to quit: 1
```

## Why not netclient?

`netclient` is Netmaker's official headless agent, and if it works for you, use it. But `netclient join -s <server>` registers the machine as a **host**, and on a Netmaker Pro server that path is restricted to admin users. A regular user gets:

```
Server SSO Error
Error reason: only admin users can register using SSO
```

That check is server-side, in the OAuth register callback — no client version or flag gets around it:

```go
// pro/auth/register_callback.go
if user.PlatformRoleID != schema.AdminRole && user.PlatformRoleID != schema.SuperAdminRole {
    response := returnErrTemplate(userClaims.getUserName(), "only admin users can register using SSO", state, reqKeyIf)
    w.WriteHeader(http.StatusForbidden)
```

Remote access is a different thing entirely. A remote-access client is an **external client** of a gateway, not a meshed host, and regular users are expected to have them. `nmrac` uses those endpoints, so it needs no admin role — the same access your Netmaker Desktop login already gives you.

## Requirements

- `bash` (3.2 is fine, so stock macOS `bash` works), `curl`, `jq`
- `wireguard-tools` — `wg` and `wg-quick`
- `node` >= 22, for the browser SSO login — it uses Node's built-in `WebSocket`, no npm packages. Not needed with `nmrac login --paste`.
- root, for `/etc/wireguard` and `wg-quick`
- A Netmaker **Pro** server with OAuth/SSO configured, and a user attached to at least one remote-access gateway

On Debian/Ubuntu:

```sh
sudo apt install curl jq wireguard-tools
```

On macOS:

```sh
brew install jq wireguard-tools node
```

Tested against Netmaker server **v1.5.1**. See [Server compatibility](#server-compatibility).

## Install

```sh
git clone https://github.com/meta-boy/nmrac
sudo install -m 0755 nmrac/nmrac /usr/local/bin/nmrac
```

## Quick start

```sh
sudo nmrac server https://netmaker.example.com   # once
sudo nmrac login                                 # opens a URL, sign in with your IdP
sudo nmrac list                                  # what you can reach
sudo nmrac connect                               # pick from a menu
sudo nmrac disconnect
```

`login` prints a URL and waits. Open it, authenticate, and the token comes back over the same socket — nothing to copy or paste.

## Commands

| Command | What it does |
| --- | --- |
| `nmrac server <url>` | Save which Netmaker server to talk to. No argument prints the current one. |
| `nmrac login [--paste]` | SSO login via browser. `--paste` falls back to pasting a JWT by hand. |
| `nmrac logout` | Forget the stored token. Leaves tunnels running. |
| `nmrac list` | Networks, their gateways, and whether each tunnel is up. |
| `nmrac connect [--dns\|--no-dns] [network]` | Bring a tunnel up. Prompts if no network is given. |
| `nmrac disconnect [network]` | Take a tunnel down, or all of them. |
| `nmrac status` | WireGuard peer state for active tunnels. |
| `nmrac conf <network>` | Print the config the server hands out, without applying it. |
| `nmrac remove <network>` | Delete this machine's client from a network, server-side. |

`connect` is idempotent. Each machine gets its own external client, keyed on a stable client id (`$(hostname)-cli`, override with `NMRAC_CLIENT_ID`), so reconnecting reuses the same client and IP rather than allocating a new one on every call. It will not touch a client belonging to another device.

## Configuration

| Variable | Default | Purpose |
| --- | --- | --- |
| `NMRAC_API` | from `/etc/nmrac/server` | Server URL; overrides the saved one |
| `NMRAC_CLIENT_ID` | `$(hostname)-cli` | This machine's client identity |
| `NMRAC_STATE_DIR` | `/etc/nmrac` | Token, server URL, interface map |
| `NMRAC_WG_DIR` | `/etc/wireguard` | Where `.conf` files are written |

State on disk:

- `/etc/nmrac/server` — server URL (`0644`)
- `/etc/nmrac/token` — your JWT (`0600`)
- `/etc/nmrac/ifaces` — interface-to-network map
- `/etc/wireguard/nm-*.conf` — generated configs (`0600`)

## How it works

| Step | Endpoint |
| --- | --- |
| Login | `ws /api/oauth/headless` → `/api/oauth/register/<state>` |
| List networks and gateways | `GET /api/users/{user}/remote_access_gw` |
| Does this machine have a client? | `GET /api/extclients/{net}/{clientid}` |
| Create one | `POST /api/extclients/{net}/{gwid}` |
| Fetch the WireGuard config | `GET /api/extclients/{net}/{clientid}/file` |
| Remove it | `DELETE /api/extclients/{net}/{clientid}` |

The config is rendered by the server, the same one the dashboard's *View Config* gives you. `nmrac` does not build it, it only strips the `DNS =` line unless you pass `--dns` (see below).

### Login, specifically

Netmaker's OAuth callback dispatches on the **length** of the state parameter:

```go
// pro/auth/auth.go
node_signin_length     = 64
headless_signin_length = 32

switch len(state) {
case node_signin_length:     HandleHostSSOCallback(w, r)      // admin-gated
case headless_signin_length: HandleHeadlessSSOCallback(w, r)  // no admin check
}
```

Both arrive at the same `/api/oauth/register/{regKey}` URL, but a 64-character state is host registration and admin-only, while a 32-character state is a headless *user* login — commented in the source as "headless logins such as Netmaker CLI". `nmrac` opens a websocket to `/api/oauth/headless`, which mints a 32-character state, and reads the JWT back off that socket.

## Notes and limitations

- **Tokens expire.** When one does, any command says so; run `nmrac login` again. Existing tunnels keep working — WireGuard does not care about the API.
- **No boot persistence.** The tunnel does not come back after a reboot. `wg-quick@nm-*.service` will bring up an existing config, but the token may have expired by then, so a truly durable setup needs a re-login.
- **DNS is decided per platform.** The server always emits a `DNS =` line. macOS `wg-quick` applies it natively via `scutil`, so it is kept. Linux `wg-quick` shells out to `resolvconf`, which many minimal images lack, so the line is kept only when `resolvconf` is installed and stripped otherwise — the tunnel comes up either way and your existing resolver is left alone. Force it with `--dns` or `--no-dns`.
- **Interface names.** Linux caps them at 15 characters and Netmaker network names are often longer, so configs are named `nm-<first 12 chars>`, falling back to `nm-<hash>` if two networks collide. `/etc/nmrac/ifaces` keeps the mapping, and `list`/`status` show real network names. On macOS the kernel assigns `utunN` regardless, so `status` shows both (`nm-lab-networ -> utun4`).
- **Split vs full tunnel** is the gateway's decision, not this tool's. `nmrac` installs exactly the `AllowedIPs` the server sends.

## Platform support

| | Linux | macOS |
| --- | --- | --- |
| Interface naming | as configured (`nm-*`) | kernel-assigned `utunN`, mapped via `/var/run/wireguard/<name>.name` |
| DNS from the gateway | needs `resolvconf` | applied natively by `wg-quick` |
| Machine name | `hostname -s` | `scutil --get LocalHostName`, then `ComputerName` |
| Address lookup | `ip -4 -o addr` | `ifconfig` |

Interface liveness is read from `wg show interfaces`, which behaves the same on both.

macOS can report a hostname like `Unknown_aa:bb:cc:dd:ee:ff` when no name is set. Those colons are not valid in a Netmaker client id, so the derived id is lowercased and stripped to `[A-Za-z0-9._-]` — that example becomes `unknown_aabbccddeeff-cli`, which is ugly but stable. Set `NMRAC_CLIENT_ID` if you want something friendlier.

## Server compatibility

Developed and tested against Netmaker server **v1.5.1** (Pro, self-hosted).

Two API families that look like the right answer are deliberately unused:

- **`/api/v1/rac/*`** exists on 1.5.1 and is correctly non-admin, but its authorization requires a `TARGET_RSRC` request header. Headers with underscores are dropped by common reverse-proxy defaults (nginx's `underscores_in_headers off`), so behind a proxy every call returns `403 target rsrc is missing`. `/api/users/{user}/remote_access_gw` is wrapped in `ContinueIfUserMatch` and needs no such header, so `nmrac` uses that instead.
- **`/api/v1/device/*`** is the newer desktop-client API, with a proxy-safe `X-Host-ID` header and an explicit exemption from those header checks. It does not exist on 1.5.1 (`404`). On a newer server it would be the better path.

If you run a different version and something breaks, an issue with your server version and the failing response would be useful.

## Troubleshooting

**`403 target rsrc is missing`** — see above; it means a request went to `/api/v1/rac/*`.

**`no gateway for '<network>'`** — your user is not attached to a remote-access gateway on that network. An admin attaches users to gateways; `nmrac` cannot.

**`--dns needs resolvconf`** — Linux only: `apt install openresolv`, or drop the flag and let DNS be skipped.

**`access denied` on an extclient path** — on v1.5.1 a client that does not exist answers `403 access denied` rather than `404`. `nmrac` treats any non-200 on the existence probe as "not present" and creates the client, so you should not see this; if you do, it is a genuine permission problem on the gateway.

**Tunnel up but nothing reachable** — check `nmrac status` for a recent handshake and non-zero transfer. If those look healthy, the crypto path is fine and you are looking at ACLs or filtering. Note that many gateways drop ICMP, so a failed `ping` proves nothing; try a real port.

## License

MIT — see [LICENSE](LICENSE).
