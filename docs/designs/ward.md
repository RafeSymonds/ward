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

The container sees only what ward mounts into it. The rule is that anything Claude needs to write is writable, and every other host path is either a copy or read-only. The user's home folder is not mounted. The project folder appears at the same absolute path it has on the host, because Claude files each project's sessions and memory under that path, and matching it is what lets a session started in ward be resumed outside ward and the other way around.

The project folder is the top of the git work tree that contains the current folder, or the current folder itself when it is not inside a git work tree or when that top would be `$HOME` or above it. Claude still starts in the current folder.

| Host path | Inside the container | Why |
|---|---|---|
| Project folder, including `.git` and `.claude/` | Read-write | This is the work. Read-only paths inside a writable folder can always be worked around (see Alternatives), so the whole folder is writable. |
| Folders the config marks container-only for this project, such as `node_modules` | A per-project volume mounted over that folder | Packages built inside the Linux container break when run on a Mac host, and the other way around. Each side keeps its own copy. |
| User Claude config in `~/.claude` (`settings.json`, `CLAUDE.md`, `skills/`, `agents/`, `commands/`, `plugins/`, `hooks/`, `output-styles/`, `statusline.sh`, `keybindings.json`) | A copy, refreshed every run. Changes made in the container are overwritten next run. | The agent gets the user's setup, but cannot change hooks or scripts that would later run on the host. |
| `~/.claude/projects/<key>`, this project's session transcripts | Read-write, live | Sessions stay resumable from either side. |
| `~/.claude/projects/<key>/memory` | Read-write, live, mounted separately over the folder above | Claude writes its memory here. Mounting it separately means the agent cannot replace the folder with a symlink that ward would follow on a later run. If the host folder is a symlink, ward mounts its target. |
| `~/.claude.json` | A filtered copy made when the container is created, never copied back. It holds the global settings and this project's entry, without any MCP servers. | It lists MCP server commands that Claude starts on the host, and it holds other projects' data. Claude writes this file constantly, so it cannot be read-only. |
| Git `user.name` and `user.email` from the host | Copied into the container's `~/.gitconfig` every run | The agent can commit. No push credentials are copied. |
| Extra folders listed in ward's config | Read-only, unless the config marks one read-write | For shared libraries or reference repos. |
| Everything else | Not visible | |

Claude writes many files under `~/.claude` during a normal run (history, session state, shell snapshots, file checkpoints). The container's `~/.claude` is therefore a real writable folder in the container's home volume, filled with the copied config, with only this project's folder bound live to the host.

The container's home folder is a volume mounted at the same path as the host home folder, so absolute paths in copied settings and plugin files keep working. When the project folder is `$HOME` or above it, the two cannot share a path, so the home volume moves to `/home/ward` and ward prints a warning that all of the project folder, which includes the host's own `~/.claude`, will be writable. Then it continues.

The container is never privileged and never sees the Docker socket. It has limits on CPU, memory, and number of processes. Who the agent runs as depends on the Docker setup:

- **Docker Desktop (macOS, and Docker Desktop for Linux):** a user with the host's UID and passwordless `sudo`. Root inside the container cannot affect host file ownership through Docker Desktop's file sharing.
- **Rootless Docker on Linux:** root inside the container, which maps to the host user. Files the agent writes are owned by the user.
- **Rootful Docker on Linux:** a user with the host's UID and no `sudo`. Root here would be real root over the project files, so it could leave a root-owned setuid program for the user to run.

### Network

The container reaches the public internet. It cannot reach the host, the local network, or other containers. When the container starts, its main process sets a firewall that rejects connections to the host's Docker gateway, private address ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `100.64.0.0/10`, IPv6 `fc00::/7`), link-local and cloud metadata addresses (`169.254.0.0/16`, `fe80::/10`), and multicast. DNS to the container's configured resolvers is allowed. Connections into the container from outside are dropped. Forwarded ports still work, because the bridge connects to them from inside the container.

Every process ward starts in the container, including Claude, runs without the right to change network settings. Only the main process keeps it, and the agent cannot take it over, so root reached through `sudo` cannot lift the firewall. If the firewall cannot be set, the container does not start.

### Container lifecycle

Each project folder gets one container, named from the folder name plus a short hash of its absolute path, and found again by a label holding that path. It is created on the first `ward` run in that folder. Several `ward` sessions in the same folder share it.

The container's home folder lives in a separate Docker volume, so it survives the container being rebuilt. On the first run ward installs Claude Code into that home folder with its native installer, and the run fails if the install fails. On later runs ward runs `claude update` with a timeout, and if the update fails or times out ward continues with the installed version. Packages the agent installs with `sudo` live in the container itself and persist until the container is rebuilt.

The base image is built locally from a setup script embedded in ward, run in a pinned Debian slim image and saved as a new image. It adds git, ripgrep, build tools, Node, Python, `sudo`, `iptables`, `ncurses-term`, and `LANG=C.UTF-8`. Its tag is a hash of the script and the pinned base, so a new ward release with a changed script produces a new image. When a project's container was built from an older image, ward warns and suggests `ward reset`.

The container's main process is `ward-agent init`. It sets the firewall, reaps exited processes, and watches for sessions, meaning processes started through `docker exec`. When none have existed for 60 seconds, it stops every process and exits, and the container stops. A stopped container uses disk only. Background processes the agent started, such as a dev server, do not count as sessions.

Every Claude session runs under `ward-agent session`, which is tied to that session's bridge. If ward is killed, its end of the bridge closes, and `ward-agent session` ends Claude. Without this, Claude would keep running in the container with prompts off, and the container would never become idle. Each session has an ID, which link opening, the status line, and port forwarding use to find their own session's bridge.

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
4. **Container.** Found or created. Before creating it, ward creates this project's folder and memory folder under the host's `~/.claude/projects/` if they are missing. Creation applies the mount table, the resource limits, and `ward-agent init` as the main process. Ward then copies `ward-agent` and the filtered `~/.claude.json` into the created container, and only then starts it, because the main process must exist before the first start. Ward warns about an outdated image and starts the container if it is stopped.
5. **Prepare.** Ward replaces `ward-agent` if its content differs from the copy embedded in ward, streams the Claude config allowlist and the git identity into the container's home, following symlinks on the host so linked files arrive as content, and installs or updates Claude.
6. **Bridge.** Ward starts `ward-agent bridge` through a second `docker exec` and keeps that stream open for the session.
7. **Claude.** Ward starts `claude --dangerously-skip-permissions` plus any arguments the user passed, under `ward-agent session`, through `docker exec` with a terminal, in the current folder. It passes the terminal type, the time zone, `LANG=C.UTF-8`, the session ID, and the login token as environment variables through the Docker API, so the token never appears on a command line.
8. **Exit.** Ward restores the terminal, closes the bridge, which closes all forwarded ports, and exits with Claude's exit code. The container stops itself 60 seconds later if no other session is running.

If the container stops on its own between ward finding it running and ward starting a process in it, ward starts it again and retries once.

Permission prompts are off by default because the container is the safety boundary. The prompts would add friction without adding protection.

### The terminal relay

Ward puts the user's terminal into raw mode and relays bytes in both directions between it and Claude's terminal inside the container. It forwards window resizes. It restores the terminal on every exit path, including errors and signals.

Ward inspects the input stream for two keys. Ctrl-Z suspends ward itself on the host, so the user's shell can resume it with `fg`. Before suspending, ward turns off the terminal modes Claude turned on, such as bracketed paste, mouse reporting, and the kitty keyboard protocol, so the user's shell works normally. On resume ward turns them back on and resizes Claude's terminal to a different size and back, which makes Claude redraw. A resize to the same size would do nothing. Claude never receives Ctrl-Z, because nothing inside the container could resume it. Ctrl+V is passed through to Claude and also opens a two-second window in which the bridge may answer a clipboard request. Ward ignores both keys inside bracketed paste, so pasted text never triggers them, and recognizes both in the kitty keyboard protocol and xterm `modifyOtherKeys` encodings as well as the plain control bytes.

Ward also filters the output stream. It removes OSC 52 escape sequences, which would let anything in the container set the host clipboard, and tmux passthrough sequences, which could carry one past tmux. Copying text to the host clipboard from Claude does not work inside ward as a result.

### The bridge

The bridge is the only way the container can ask the host for anything. It runs over a second `docker exec` stream that ward opens, rather than a socket shared into the container, for two reasons. Sharing a host Unix socket through Docker Desktop's VM file sharing is not reliable, and a stream opened from the host side exposes nothing on the host that the container could connect to on its own. Several logical streams are multiplexed over the one exec stream.

The bridge accepts exactly these requests. Anything else gets an error reply, and the bridge stays open.

- **Open a URL.** `http` and `https` only. The host opens it with `open` on macOS or `xdg-open` on Linux. The container's `xdg-open` is a stand-in that sends this request, so links Claude tries to open land in the user's browser.
- **Read the clipboard image.** Answered only within two seconds of a Ctrl+V that ward saw. Returns a PNG. The container's `xclip` and `wl-paste` are stand-ins that send this request. The gate exists because copied passwords pass through the clipboard, and the agent should not be able to read it at will.
- **Port opened, port closed.** `ward-agent` checks `/proc/net/tcp` and `/proc/net/tcp6` once a second for listening sockets, skipping ports the config ignores. Each session forwards only the ports opened by its own processes, plus ports whose session has ended, which the newest session adopts. So two sessions never forward the same server twice. For each new port, ward listens on `127.0.0.1` on the host at the same port number, or the next free one if it is taken, and tells `ward-agent` which host port it got. Each incoming connection becomes a new stream that `ward-agent` connects to the port inside the container. Ward only listens on `127.0.0.1`, so forwarded ports are never exposed to the network.

### The status line

Ward cannot print into the terminal while Claude's screen is up, so it reports forwarded ports through Claude's own status line. Ward starts Claude with a `--settings` argument that sets the status line command to `ward-agent statusline`. Settings given on the command line outrank project settings, so a project's own status line cannot hide ward's. `ward-agent statusline` finds the status line command the user's settings would have used, runs it with the same input Claude gave it, then appends ward's information, for example `⇄ 3000→3001`. If the user has no status line, it shows only ward's information.

Errors ward cannot show while Claude owns the screen, such as a port that could not be forwarded or a link that could not be opened, go to ward's log file. When Claude exits, ward prints how many warnings were logged and where.

### Login

`ward login` asks the user to run `claude setup-token` on any machine with a browser and paste the resulting token. Ward stores it in the macOS Keychain, the Linux Secret Service, or, on Linux hosts without a Secret Service, a file under `~/.config/ward/` readable only by the user. The token is independent of the user's normal Claude login, so neither can log the other out, and it can be revoked on its own.

### Configuration

All configuration lives in `~/.config/ward/config.toml`. Nothing ward trusts is read from the project folder, because the agent can write there and could otherwise widen its own access for the next run. The file holds global defaults and per-project overrides keyed by absolute path:

- CPU limit, default half the CPUs Docker reports. On Docker Desktop that is the VM's CPU count, not the Mac's. A configured value above Docker's count is lowered to it, with a warning.
- Memory limit, default half the memory Docker reports.
- Process limit, default 4096.
- Extra mounts, each a host path, read-only unless marked read-write, mounted at the same path.
- Container-only folders, each a path inside the project, kept in a volume.
- Ports to never forward.

### Limits

ward protects the host from damage. It does not stop theft, and some things are out of its reach.

- The agent can read the project folder, including `.env` files, the copied Claude config, any extra mounts, and the login token, which it can read from Claude's environment. It can reach the public internet. It could send any of that elsewhere.
- The agent can delete or rewrite anything in the project folder, including its git history. Ward keeps no snapshots.
- Code the agent writes runs on the host if the user runs it there. This includes scripts, Makefiles, package scripts, and tests. It also includes project files that host tools run without being asked: git hooks and git config (on the next commit, or on `git status` from a shell prompt, also in nested repositories), the project's `.claude/settings.json` hooks and `CLAUDE.md` (in the next host Claude session), `.envrc`, and editor task files. The same goes for transcripts and memory, which host Claude reads on resume and in every session.
- In a linked git worktree, `.git` is a file pointing outside the project, so git does not work in the container unless the config mounts that folder.
- On Docker Desktop, a container escape lands in a VM that shares all of `/Users` read-write. Docker Desktop for Linux also runs a VM.
- With Docker on Linux without Docker Desktop, there is no VM. Containers share the host's kernel, and membership in the `docker` group is equivalent to root. Rootless Docker is the documented mitigation.
- Host config that refers to macOS-only paths or tools, such as Homebrew paths in hooks, fails inside the Linux container.
- IDE integration and MCP servers that drive host applications do not work inside the container.
- How Claude Code on Linux reads clipboard images, meaning which tool and which environment variables, is unverified. It must be tested before the stand-ins are built.

## Alternatives

Every approach below came up during design and was rejected or deferred. Each entry says why, so the question does not need to be argued again.

### Overall approach

- **Use an existing tool instead of building ward.** Docker's `sbx`, yolobox, aibox, claude-code-sandbox, and servusdei2018/sandbox all run coding agents in a container or similar sandbox with the current folder mounted. They were not evaluated in depth. ward exists for the parts this design treats as essential: keeping the agent from changing the user's own Claude setup that later runs on the host (`~/.claude` hooks and scripts, `~/.claude.json` MCP commands) and from reaching the host or the local network, while keeping the user's config and resumable sessions, and the host bridge for ports, links, and clipboard. If an existing tool turns out to do all of that, this decision should be revisited.
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
- **Read ward's settings from a file in the project folder.** Rejected. The agent can write to the project folder and could widen its own mounts for the next run. All ward configuration lives in `~/.config/ward/`. For the same reason ward never follows a `.git` file in the project to find and mount a worktree's git folder.
- **Mount `.git/hooks`, `.git/config`, and the project's `.claude/settings*.json` read-only inside the writable project.** Rejected after it was first approved. Tested on Docker Desktop: root in the container could not write, remount, or unmount them, but it renamed `.git`, created a new one with a hook, and the next host `git commit` ran that hook. Mounting all of `.git` read-only stops the rename but stops the agent from committing, and a new repository in a subfolder with `core.fsmonitor` set still ran its command on the host's `git status`. A mount protects one path that exists when the container starts. The danger is any path with a given name, at any depth, including paths that do not exist yet. The user's rule is that read-only protection that can be worked around is not worth having.
- **Keep the project in a volume and sync it to the host, holding back changes to protected files for review.** Rejected. It is the only way to enforce read-only paths where the agent cannot reach, but the protection would still be partial, because build files, package scripts, and source code run on the host too and cannot be held back. It also adds sync delay, conflicts, and a second copy of every project.
- **Filter writes through a filesystem layer inside the container.** Rejected for the same partial coverage, plus extra container privileges, slower file access on macOS, and a complex setup to keep the agent from reaching the unfiltered folder.
- **Mount the memory folder read-only.** Rejected after it was first approved. Claude writes its memory there, and with the project's `CLAUDE.md` writable, read-only memory no longer stops anything.
- **Use the current folder as the project folder even inside a git work tree.** Rejected. Running ward in a subfolder would create a second container for the same repository, and git needs the whole work tree.
- **Mount the container's home folder at `/home/<user>`.** Rejected. Copied settings and plugin files hold absolute paths under the host home folder, which would not exist.

### Login

- **Copy the host's Claude login from the Keychain into the container.** Rejected. Host and container would share one refresh token. If the server issues a new refresh token on every refresh, whichever side refreshes first logs the other out. Whether it does was not verified, and the risk was not worth taking.
- **Log in inside each container and keep the credentials in its home volume.** Rejected. Every project has its own container and volume, so the user would log in once per project, and again after `ward rm`.
- **Have `ward login` run `claude setup-token` inside the container.** Rejected for v1. Whether `setup-token` works there without a browser was not verified. The user runs it on any machine with a browser and pastes the token.

### Container lifecycle

- **A fresh container every run.** Rejected. Anything the agent installed with `sudo` would be gone next run, which makes `sudo` nearly useless.
- **Recreate the container every run but keep its files.** Rejected. Packages installed with `sudo` live in system folders that belong to the image. Keeping those folders across a rebuild would hide the new image's versions, so the user would get neither benefit. Claude Code lives in the home volume and updates itself instead.
- **Bake Claude Code into the image.** Rejected. Every Claude release would need an image rebuild and a container rebuild. The native installer in the home volume updates in place.
- **Bake `ward-agent` into the image.** Rejected. Ward copies `ward-agent` in when its version differs, so host and agent always match without rebuilding the image.
- **A separate container for each session in the same project.** Deferred, not rejected. The user wants it as a setting later. It has not been designed, and a setting with one working value adds nothing, so v1 shares one container per project.
- **A Dockerfile per project.** Deferred. v1 uses one base image, and the agent installs anything else with `sudo`.
- **A host lock file to stop the container when the last session exits.** Rejected in favor of the in-container idle watchdog. The lock left the container running whenever ward was killed or the terminal was closed, and it had a race between one session exiting and another starting.
- **Snapshot the project before each session.** Rejected by the user. Deletion inside the project is not a concern for this design.
- **Build the image from a Dockerfile through Docker's builder.** Rejected. Docker's classic builder is deprecated, and driving BuildKit through the API needs a large extra client. Ward runs its setup script in a container from the pinned base image and saves the result as an image, using only basic API calls.

### The host program

- **Ward replaces itself with `docker exec` once setup is done.** Rejected. With no ward process left during the session, there is nothing to serve the bridge, catch Ctrl-Z, or gate the clipboard.
- **End orphaned sessions with a heartbeat from ward.** Rejected. Ctrl-Z stops ward's process, which would look the same as a dead ward. The bridge closing is the signal instead, because the operating system closes ward's connection when ward dies.
- **A background ward daemon managing all containers.** Rejected. It needs a launchd or systemd service, upgrade and crash handling, and authentication on its socket. Docker already keeps the container state, and no v1 feature needs a daemon.
- **Drive the `docker` command-line tool for everything.** Rejected. Ward relays the terminal itself, and doing that through the CLI would mean running the CLI on a second pseudo-terminal that ward controls. Ward uses the Docker API at the current context's endpoint instead.

### Bridge and ports

- **Share a host Unix socket into the container for the bridge.** Rejected. It is not reliable through Docker Desktop's VM file sharing, and it would give the container something on the host to connect to on its own. The bridge runs over a `docker exec` stream that ward opens.
- **A bridge request that runs an arbitrary command on the host.** Rejected permanently. It would undo the sandbox. The bridge has a fixed list of requests.
- **Publish a fixed range of ports when the container is created.** Rejected. Docker fixes published ports at creation, so the range would have to be guessed in advance and would hold those host ports for every project's container. Ward relays ports as they appear.
- **Listen on all network interfaces for forwarded ports.** Rejected. Dev servers would be reachable from the local network. Ward listens on `127.0.0.1` only.
- **Show forwarded ports in the terminal title.** Rejected. The user runs tmux, which hides the title unless configured to show it. Ward uses Claude's status line.
- **Let the agent read the clipboard at any time.** Rejected. Copied passwords pass through the clipboard. Reads are answered only within two seconds of a Ctrl+V.
- **Close the bridge on any unexpected request.** Rejected. An ordinary `file://` or `mailto:` link would end port forwarding and image paste for the rest of the session. Unknown requests get an error reply.

### Network and terminal

- **Let the container reach the host and the local network.** Rejected by the user. An unauthenticated service on the host or the LAN would be a way out of the container. An allowlist in the config may be added later.
- **Block the public internet except an allowlist.** Not adopted. The agent installs packages and reads documentation, and the threat model is damage to the host, not theft.
- **Pass OSC 52 through to the host terminal.** Rejected. Anything in the container could set the host clipboard and plant a command in the user's next paste.

### Remote sessions (deferred)

Running the container on another machine is deferred until local mode works. The options considered so far, none accepted:

- **Mount the local folder on the remote machine over SSH (sshfs).** Every file operation crosses the network, which is slow over the internet. The SSH file server has no path restriction, so the remote machine could read anything the user can read on their machine.
- **Two-way file sync (Mutagen).** Fast on both sides, but deletions sync back, so the agent can still delete local files.
- **Git as the transport, with results returned as a branch.** The agent never touches local files. The user judged it not clean enough and wants a better option.
- **Managed sandbox services (e2b, Modal, Daytona).** Each ties ward to one provider. A plain SSH host was preferred.
