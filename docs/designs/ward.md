# ward

## Problem

Coding agents like Claude Code are most useful when they can run commands without asking permission each time, but on a personal machine that means one bad command can delete or damage files anywhere the user can reach. Turning permission prompts back on makes the agent slow, and it still depends on the user catching every dangerous command. ward lets a developer run Claude Code with prompts off while limiting what it can damage to the project they are working in. It runs Claude in a container that sees the project folder and a few chosen folders, keeps the rest of the machine out of reach, and still feels like running Claude in their own terminal, with their settings, their resumable sessions, and access to dev servers the agent starts. It is built for one developer on macOS or Linux.

## Technical Plan

### Overview

ward is two programs. `ward` runs on the host, macOS or Linux, in the user's own terminal. `ward-agent` runs inside the container. Both are written in Go and built from one module. The host binary carries Linux builds of `ward-agent` (amd64 and arm64) inside it and copies the right one into the container. Go was chosen because it produces a single self-contained binary per platform and cross-compiles the Linux agent without extra tooling.

The user types `ward` in a project folder where they would have typed `claude`. Ward prepares a container for that folder, starts Claude inside it, and connects the user's terminal to it. Ward stays running for the whole session and exits when Claude exits. Only one copy of Claude runs, and it runs inside the container.

```
 Host (macOS or Linux)
 ┌──────────────────────────────────────────────────────────────────┐
 │ terminal ⇄ ward  (foreground for the whole session)               │
 │             ├─ session stream ─── docker exec (tty)  claude ───┐  │
 │             ├─ bridge stream ──── docker exec  ward-agent bridge┤  │
 │             ├─ 127.0.0.1:<port> listeners for forwarded ports   │  │
 │             └─ opens URLs, reads clipboard (paste-gated)        │  │
 └─────────────────────────────────────────────────────────────────┼──┘
                                                                   │ Docker API
 ┌─────────────────────────────────────────────────────────────────▼──┐
 │ container ward-<dirname>-<pathhash>   cpus / memory / process caps  │
 │  main process: ward-agent init   (reaps exited processes,           │
 │                                   stops container when idle)        │
 │  claude ── stand-in xdg-open / xclip / wl-paste ─► ward-agent bridge│
 │  dev server :3000 ◄── bridge connects to it for forwarded traffic   │
 └─────────────────────────────────────────────────────────────────────┘
```

Ward talks to Docker through the Docker API, at the endpoint of the user's current `docker context`. On a Mac this is Docker Desktop's socket under `~/.docker/run/`, which is not the default location, so reading the context matters.

### What the agent can see and change

The container sees only what ward mounts into it. The user's home folder is not mounted. The project folder appears at the same absolute path it has on the host, because Claude files each project's sessions and memory under that path, and matching it is what lets a session started in ward be resumed outside ward and the other way around.

| Host path | Inside the container | Why |
|---|---|---|
| Project folder | Read-write | This is the work. |
| `<project>/.git/hooks`, `<project>/.git/config` | Read-only | Git runs hooks, and commands named in `core.fsmonitor` or `core.hooksPath`, on the host later. A writable copy would let the agent plant a command that runs outside the container. |
| `<project>/.claude/settings.json`, `settings.local.json` | Read-only | Project hooks in these files run on the host in the user's next normal Claude session. |
| User Claude config in `~/.claude` (`settings.json`, `CLAUDE.md`, `skills/`, `agents/`, `commands/`, `plugins/`, `statusline.sh`, `keybindings.json`) | A copy, refreshed every run. Changes made in the container are overwritten next run. | The agent gets the user's setup, but cannot change hooks or scripts that would later run on the host. |
| `~/.claude/projects/<key>`, this project's session transcripts | Read-write, live | Sessions stay resumable from either side. |
| `~/.claude/projects/<key>/memory` | Read-only, mounted over the folder above | Memory is loaded as instructions into future host sessions. Ward creates the folder on the host if it does not exist, so the agent can never create it through the writable mount. If it is a symlink, ward mounts the symlink's target. |
| `~/.claude.json` | A copy made when the container is created, never copied back | It lists MCP server commands that Claude starts on the host. Claude writes this file constantly, so it cannot be read-only. |
| Extra folders listed in ward's config | Read-only, unless the config marks one read-write | For shared libraries or reference repos. |
| Everything else | Not visible | |

Claude writes many files under `~/.claude` during a normal run (history, session state, shell snapshots, file checkpoints). The container's `~/.claude` is therefore a real writable folder in the container's home volume, filled with the copied config, with only the session-transcript folder bound live to the host.

If the project folder is `$HOME`, `/`, or any folder above `$HOME`, ward prints a warning that all of it will be writable and continues.

The agent runs as a normal user with passwordless `sudo` inside the container. The container is never privileged and never sees the Docker socket. It has limits on CPU, memory, and number of processes.

### Container lifecycle

Each project folder gets one container, named from the folder name plus a short hash of its absolute path, and found again by a label holding that path. It is created on the first `ward` run in that folder. Several `ward` sessions in the same folder share it.

The container's home folder lives in a separate Docker volume, so it survives the container being rebuilt. Claude Code is installed into that home folder with its native installer, and ward runs `claude update` before every launch, with a timeout so a slow network never blocks the start. Packages the agent installs with `sudo` live in the container itself and persist until the container is rebuilt.

The base image is built locally from a Dockerfile embedded in ward. It is Debian slim with git, ripgrep, build tools, Node, Python, `sudo`, `ncurses-term`, and `LANG=C.UTF-8`. Its tag is a hash of the Dockerfile, so a new ward release with a changed Dockerfile produces a new image. When a project's container was built from an older image, ward warns and suggests `ward reset`.

The container's main process is `ward-agent init`. It reaps exited processes and watches for sessions, meaning processes started through `docker exec`. When none have existed for 60 seconds, it exits and the container stops. A stopped container uses disk only. Background processes the agent started, such as a dev server, do not count as sessions. This works even if ward is killed without a chance to clean up.

- `ward reset` rebuilds the container from the current image. System packages are lost. The home volume is kept.
- `ward rm` deletes the container and its home volume.
- `ward prune --older-than <age>` deletes containers whose project has not been used within that age.
- `ward stop` stops the current project's container now.
- `ward ls` lists ward containers with their project path, state, image age, and disk use.
- `ward build` rebuilds the base image.
- `ward shell` opens a shell in the project's container.

### One session, step by step

1. **Home warning**, if applicable.
2. **Config and Docker endpoint.** Ward reads its config and resolves the current Docker context.
3. **Image.** Built if its tag is missing.
4. **Container.** Found or created. On creation ward applies the mount table, the resource limits, a user whose UID matches the host user, and `ward-agent init` as the main process, then copies `~/.claude.json` in and creates the host memory folder if needed. Ward warns about an outdated image and starts the container if it is stopped.
5. **Prepare.** Ward copies `ward-agent` in if its version differs, streams the Claude config allowlist into the container's `~/.claude`, following symlinks on the host so linked files arrive as content, and runs `claude update`.
6. **Bridge.** Ward starts `ward-agent bridge` through a second `docker exec` and keeps that stream open for the session.
7. **Claude.** Ward starts `claude --dangerously-skip-permissions` plus any arguments the user passed, through `docker exec` with a terminal, in the project's path. It passes `TERM`, `COLORTERM`, `LANG`, and the login token as environment variables through the Docker API, so the token never appears on a command line.
8. **Exit.** Ward restores the terminal, closes the bridge, which closes all forwarded ports, and exits with Claude's exit code. The container stops itself 60 seconds later if no other session is running.

Permission prompts are off by default because the container is the safety boundary. The prompts would add friction without adding protection.

### The terminal relay

Ward puts the user's terminal into raw mode and relays bytes in both directions between it and Claude's terminal inside the container. It forwards window resizes. It restores the terminal on every exit path, including errors and signals.

Ward inspects the input stream for two keys. Ctrl-Z suspends ward itself on the host, so the user's shell can resume it with `fg`. On resume ward sends a resize to make Claude redraw. Claude never receives Ctrl-Z, because nothing inside the container could resume it. Ctrl+V is passed through to Claude and also opens a two-second window in which the bridge may answer a clipboard request. Ward ignores both keys inside bracketed paste, so pasted text never triggers them, and recognizes both in the kitty keyboard protocol encoding as well as the plain control bytes.

### The bridge

The bridge is the only way the container can ask the host for anything. It runs over a second `docker exec` stream that ward opens, rather than a socket shared into the container, for two reasons. Sharing a host Unix socket through Docker Desktop's VM file sharing is not reliable, and a stream opened from the host side exposes nothing on the host that the container could connect to on its own. Several logical streams are multiplexed over the one exec stream.

The bridge accepts exactly these requests. Anything else closes it.

- **Open a URL.** `http` and `https` only. The host opens it with `open` on macOS or `xdg-open` on Linux. The container's `xdg-open` is a stand-in that sends this request, so links Claude tries to open land in the user's browser.
- **Read the clipboard image.** Answered only within two seconds of a Ctrl+V that ward saw. Returns a PNG. The container's `xclip` and `wl-paste` are stand-ins that send this request. The gate exists because copied passwords pass through the clipboard, and the agent should not be able to read it at will.
- **Port opened, port closed.** `ward-agent` checks `/proc/net/tcp` and `/proc/net/tcp6` once a second for listening sockets, skipping ports the config ignores. For each new one, ward listens on `127.0.0.1` on the host at the same port number, or the next free one if it is taken, and tells `ward-agent` which host port it got. Each incoming connection becomes a new stream that `ward-agent` connects to the port inside the container. Ward only listens on `127.0.0.1`, so forwarded ports are never exposed to the network.

### The status line

Ward cannot print into the terminal while Claude's screen is up, so it reports forwarded ports through Claude's own status line. In the container's copy of `settings.json`, ward replaces the status line command with `ward-agent statusline`, and records the user's original command. `ward-agent statusline` runs the original command with the same input Claude gave it, then appends ward's information, for example `⇄ 3000→3001`. If the user has no status line, it shows only ward's information.

### Login

`ward login` asks the user to run `claude setup-token` on any machine with a browser and paste the resulting token. Ward stores it in the macOS Keychain, the Linux Secret Service, or, on Linux hosts without a Secret Service, a file under `~/.config/ward/` readable only by the user. The token is independent of the user's normal Claude login, so neither can log the other out, and it can be revoked on its own.

### Configuration

All configuration lives in `~/.config/ward/config.toml`. Nothing ward trusts is read from the project folder, because the agent can write there and could otherwise widen its own access for the next run. The file holds global defaults and per-project overrides keyed by absolute path:

- CPU limit, default half the host's cores.
- Memory limit, default half the memory Docker reports as available.
- Process limit, default 4096.
- Extra mounts, each a host path, read-only unless marked read-write, mounted at the same path.
- Ports to never forward.

### Limits

ward protects the host from damage. It does not stop theft, and some things are out of its reach.

- The agent can read the project folder, including `.env` files, the copied Claude config, and any extra mounts, and it has full network access. It could send any of that elsewhere.
- Code the agent writes, such as scripts, Makefiles, package scripts, and tests, runs on the host if the user runs it there. Ward only protects files that run automatically.
- On Linux hosts there is no VM. Containers share the host's kernel, and membership in the `docker` group is equivalent to root. Rootless Docker is the documented mitigation.
- Host config that refers to macOS-only paths or tools, such as Homebrew paths in hooks, fails inside the Linux container.
- IDE integration and MCP servers that drive host applications do not work inside the container.
- How Claude Code on Linux reads clipboard images, meaning which tool and which environment variables, is unverified. It must be tested before the stand-ins are built.

## Alternatives

Every approach below came up during design and was rejected or deferred. Each entry says why, so the question does not need to be argued again.

### Overall approach

- **Use an existing tool instead of building ward.** Docker's `sbx`, yolobox, aibox, claude-code-sandbox, and servusdei2018/sandbox all run coding agents in a container or similar sandbox with the current folder mounted. They were not evaluated in depth. ward exists for the parts this design treats as essential: keeping the agent from changing anything that later runs on the host (`~/.claude` hooks, `~/.claude.json` MCP commands, git hooks, project settings) while keeping the user's config and resumable sessions, and the host bridge for ports, links, and clipboard. If an existing tool turns out to do all of that, this decision should be revisited.
- **Support every harness (Codex, Aider, and others) from the start.** Deferred. Each harness has its own config folder, login, and files that run on the host, and each needs its own version of the isolation work done here for Claude. v1 supports Claude Code only.
- **Run Claude on the host and send only its shell commands into the container, using a Claude hook.** Rejected. Claude's own file tools for writing and editing do not go through the shell, so they would still change host files directly. It would also keep the agent's whole process on the host.
- **Rely on Claude Code's built-in sandbox.** Rejected as the only protection. It restricts the shell commands Claude runs, not the Claude process itself, and it does not give the agent a machine where it can install packages with `sudo`.
- **Use a different container runtime (Apple `container`, Podman, OrbStack).** Not evaluated. Docker was chosen because it is already installed and behaves the same on macOS and Linux. Apple `container` is macOS-only.

### What the container can see

- **Mount the whole home folder read-only.** Rejected. It stops deletion but lets the agent read `~/.ssh`, `~/.aws`, GitHub CLI tokens, and every other secret in the home folder, and the agent has network access to send them anywhere. Nothing from the home folder is mounted except what the mount table lists.
- **Refuse to run when the project folder is `$HOME` or above it.** Rejected for now in favor of a warning, by the user's choice. A refusal with an override flag remains an option if the warning proves easy to ignore.
- **Mount `~/.claude` read-write.** Rejected. The agent could add a hook to `settings.json` or edit `statusline.sh`, and the change would run on the host the next time the user starts Claude outside ward.
- **Mount `~/.claude` read-only, with only the session folders writable.** Rejected after it was first approved. Claude writes to many places in `~/.claude` during a normal run, including history, session state, shell snapshots, file checkpoints, and task files. A read-only mount turns those writes into errors.
- **Give each container its own full, separate `~/.claude`.** Rejected. Sessions started in ward would not appear in the user's normal Claude, and the container's config would drift from the host's. The chosen design copies only the config allowlist each run and binds this project's session folder live.
- **Mount all of `~/.claude` read-only at a side path and copy the config inside the container.** Rejected. It would expose every other project's transcripts and session files to the agent. Ward streams only the allowlisted files from the host instead.
- **Mount `~/.claude.json` read-write or read-only.** Both rejected. Read-write lets the agent add an MCP server command that runs on the host. Read-only fails because Claude writes the file constantly. The container gets a copy made at creation time.
- **Read ward's settings from a file in the project folder.** Rejected. The agent can write to the project folder and could widen its own mounts for the next run. All ward configuration lives in `~/.config/ward/`.

### Login

- **Copy the host's Claude login from the Keychain into the container.** Rejected. Host and container would share one refresh token. If the server issues a new refresh token on every refresh, whichever side refreshes first logs the other out. Whether it does was not verified, and the risk was not worth taking.
- **Log in inside each container and keep the credentials in its home volume.** Rejected. Every project has its own container and volume, so the user would log in once per project, and again after `ward rm`.
- **Have `ward login` run `claude setup-token` inside the container.** Rejected for v1. Whether `setup-token` works there without a browser was not verified. The user runs it on any machine with a browser and pastes the token.

### Container lifecycle

- **A fresh container every run.** Rejected. Anything the agent installed with `sudo` would be gone next run, which makes `sudo` nearly useless.
- **Recreate the container every run but keep its files.** Rejected. Packages installed with `sudo` live in system folders that belong to the image. Keeping those folders across a rebuild would hide the new image's versions, including any new Claude Code, so the user would get neither benefit. Claude Code lives in the home volume and updates itself instead.
- **Bake Claude Code into the image.** Rejected. Every Claude release would need an image rebuild and a container rebuild. The native installer in the home volume updates in place.
- **Bake `ward-agent` into the image.** Rejected. Ward copies `ward-agent` in when its version differs, so host and agent always match without rebuilding the image.
- **A separate container for each session in the same project.** Deferred, not rejected. The user wants it as a setting later. It has not been designed, and a setting with one working value adds nothing, so v1 shares one container per project.
- **A Dockerfile per project.** Deferred. v1 uses one base image, and the agent installs anything else with `sudo`.
- **A host lock file to stop the container when the last session exits.** Rejected in favor of the in-container idle watchdog. The lock left the container running whenever ward was killed or the terminal was closed, and it had a race between one session exiting and another starting.

### The host program

- **Ward replaces itself with `docker exec` once setup is done.** Rejected. With no ward process left during the session, there is nothing to serve the bridge, catch Ctrl-Z, or gate the clipboard.
- **A background ward daemon managing all containers.** Rejected. It needs a launchd or systemd service, upgrade and crash handling, and authentication on its socket. Docker already keeps the container state, and no v1 feature needs a daemon.
- **Drive the `docker` command-line tool for everything.** Rejected. Ward relays the terminal itself, and doing that through the CLI would mean running the CLI on a second pseudo-terminal that ward controls. Ward uses the Docker API at the current context's endpoint instead.

### Bridge and ports

- **Share a host Unix socket into the container for the bridge.** Rejected. It is not reliable through Docker Desktop's VM file sharing, and it would give the container something on the host to connect to on its own. The bridge runs over a `docker exec` stream that ward opens.
- **A bridge request that runs an arbitrary command on the host.** Rejected permanently. It would undo the sandbox. The bridge has a fixed list of requests.
- **Publish a fixed range of ports when the container is created.** Rejected. Docker fixes published ports at creation, so the range would have to be guessed in advance and would hold those host ports for every project's container. Ward relays ports as they appear.
- **Listen on all network interfaces for forwarded ports.** Rejected. Dev servers would be reachable from the local network. Ward listens on `127.0.0.1` only.
- **Show forwarded ports in the terminal title.** Rejected. The user runs tmux, which hides the title unless configured to show it. Ward uses Claude's status line.
- **Let the agent read the clipboard at any time.** Rejected. Copied passwords pass through the clipboard. Reads are answered only within two seconds of a Ctrl+V.

### Remote sessions (deferred)

Running the container on another machine is deferred until local mode works. The options considered so far, none accepted:

- **Mount the local folder on the remote machine over SSH (sshfs).** Every file operation crosses the network, which is slow over the internet. The SSH file server has no path restriction, so the remote machine could read anything the user can read on their machine.
- **Two-way file sync (Mutagen).** Fast on both sides, but deletions sync back, so the agent can still delete local files.
- **Git as the transport, with results returned as a branch.** The agent never touches local files. The user judged it not clean enough and wants a better option.
- **Managed sandbox services (e2b, Modal, Daytona).** Each ties ward to one provider. A plain SSH host was preferred.
