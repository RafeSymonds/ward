# ward: technical proposal, draft 2

Prose and block diagrams, not code. Supersedes `proposal-draft-1.md`.

Changes since draft 1: O28 replaces O14, O30 idle watchdog replaces the host lock, O31 makes ward a foreground supervisor with a host bridge, port forwarding is in v1, hosts are macOS and Linux, F3 is a warning, `--dangerously-skip-permissions` is the default, remote sessions are deferred.

## Components

1. **`ward`**, the host program. Go, runs on macOS and Linux. It stays in the foreground for the whole session (O31). It talks to Docker through the Docker API, using the endpoint of the current `docker context`, so Docker Desktop's non-default socket works without configuration.
2. **`ward-agent`**, the in-container program. Same Go module, compiled for linux/amd64 and linux/arm64, embedded in the `ward` binary, and copied into the container when its version differs. It has three roles: PID 1 (`init`, which reaps zombie processes and runs the O30 idle watchdog), the container end of the bridge (`bridge`), and stand-ins for `xdg-open`, `xclip`, and `wl-paste`.
3. **Base image** (O24). Debian slim with git, ripgrep, build tools, Node, Python, sudo, `ncurses-term`, and `LANG=C.UTF-8`. Claude Code is not in the image. Its native installer puts it in the home volume, where it updates itself.
4. **Config** at `~/.config/ward/config.toml`: global defaults (cpus, memory, pids limit, extra mounts, ignored ports) and per-project overrides keyed by absolute path. Nothing ward trusts is read from the project folder (F13).
5. **Token store**: macOS Keychain, Linux Secret Service, or a 0600 file on headless Linux (F15).

```
 Host (macOS or Linux)
 ┌──────────────────────────────────────────────────────────────────┐
 │ terminal ⇄ ward  (foreground, raw mode)                           │
 │             ├─ session stream ─── docker exec (tty)  claude ───┐  │
 │             ├─ bridge stream ──── docker exec  ward-agent bridge┤  │
 │             ├─ 127.0.0.1:<port> listeners for forwarded ports   │  │
 │             └─ opens URLs, reads clipboard (paste-gated)        │  │
 └─────────────────────────────────────────────────────────────────┼──┘
                                                                   │ Docker API
 ┌─────────────────────────────────────────────────────────────────▼──┐
 │ container ward-<dirname>-<pathhash>   cpus / memory / pids caps     │
 │  PID 1  ward-agent init   (reaper, idle watchdog)                   │
 │  claude ── xdg-open / xclip / wl-paste stand-ins ─► ward-agent bridge│
 │  dev server :3000 ◄── bridge dials it for each forwarded connection │
 │                                                                     │
 │  <cwd, same absolute path>                    RW  bind              │
 │  <cwd>/.git/hooks, .git/config                RO  bind              │
 │  <cwd>/.claude/settings*.json                 RO  bind              │
 │  /home/agent                                  volume, persists      │
 │    .claude/                 own copy, allowlist copied in each run  │
 │    .claude/projects/<key>                     RW  bind to host      │
 │    .claude/projects/<key>/memory              RO  bind to host      │
 │    .claude.json             copy seeded from host at create         │
 │    .local/bin/claude        native install, updated each run        │
 │  <extra mounts from config, same path>        RO by default         │
 └─────────────────────────────────────────────────────────────────────┘
```

## One session

1. **Home warning.** If the current directory is `$HOME`, `/`, or above `$HOME`, ward prints a warning that the whole folder is writable, then continues (F3).
2. **Config and Docker endpoint.** Ward reads its config and resolves the current Docker context.
3. **Image.** The tag is a hash of the embedded Dockerfile. Ward builds it if missing.
4. **Container.** Found by label, created if missing. Creation applies the mount table, resource caps, a user matching the host UID with passwordless sudo, and `ward-agent init` as PID 1. It also seeds `~/.claude.json` (F12) and creates the host memory folder if missing (F7). Ward warns if the container predates the current image (R2) and starts it if stopped.
5. **Prepare.** Ward copies `ward-agent` in if its version differs, streams the config allowlist into the container's `~/.claude` (O28), and runs `claude update` with a timeout.
6. **Bridge.** Ward starts `ward-agent bridge` through a second `docker exec` and multiplexes all bridge traffic over that one stream. The host opens it, so no host socket is ever exposed to the container (F20).
7. **Session.** Ward starts `claude --dangerously-skip-permissions <args>` through `docker exec` with a terminal, passing `TERM`, `COLORTERM`, `LANG`, and the token as environment. Ward puts the host terminal in raw mode, relays input and output, forwards window resizes, catches Ctrl-Z to suspend itself locally (R6), and notes each Ctrl+V for clipboard gating.
8. **Exit.** Ward restores the terminal, closes the bridge (forwarded ports close with it), and exits with Claude's exit code. The container stops itself after 60 seconds with no sessions (O30).

## Bridge (F19)

The bridge accepts exactly these requests. Anything else closes it.

- **Open URL.** `http` and `https` only. The host opens it with `open` or `xdg-open`.
- **Read clipboard image.** Answered only within 2 seconds of a Ctrl+V that ward saw in the terminal input. Returns PNG.
- **Port opened / port closed.** `ward-agent` polls `/proc/net/tcp` and `/proc/net/tcp6` every second for listening sockets, minus ignored ports. Ward listens on `127.0.0.1:<same port>` on the host, or the next free port if taken, and relays each connection as a new stream that `ward-agent` dials inside the container.

## Commands

`ward [-- args]`, `ward login`, `ward shell`, `ward ls`, `ward stop`, `ward reset`, `ward rm`, `ward prune --older-than <age>`, `ward build`. Unchanged from draft 1.

## Findings and risks

- **F20.** Bind-mounting a host Unix socket into a container is not reliable through Docker Desktop's VM file sharing. The bridge runs over a `docker exec` stream instead, which behaves the same on macOS and Linux.
- **R3.** Host config can reference macOS-only paths and tools. Those fail inside the container.
- **R5.** On Linux, containers share the host kernel and `docker` group membership is root-equivalent. Document rootless Docker.
- **R7.** How Claude Code on Linux reads clipboard images (which tool, which environment variables it checks first) is unverified. Test before building the stand-ins.
- **R8.** Input interception must ignore bytes inside bracketed paste and must recognize Ctrl-Z and Ctrl+V in kitty keyboard protocol encoding, or ward misfires or misses keys.

## Open

- **Q15.** How ward shows forwarded ports while Claude owns the screen. O33: wrap the status line command in the container's settings copy and append ward's info. O34: set the terminal title.
