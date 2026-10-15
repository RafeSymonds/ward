# Goldfish B: critic, round 1 (2026-10-14)

Agent: reviewer. Full report as returned. Finding numbers are the reviewer's own. "(verify)" marks claims the reviewer could not confirm from the repo.

## Critical

- F1 (ward.md:43) The rest of `.git` is writable. The agent can write `.git/commondir` pointing at a directory it controls, and git then reads `config` and `hooks` from there, bypassing both read-only mounts. Fix: protect all of `.git`, or check it on every start and exit.
- F2 (ward.md:43-44) A read-only bind inside a writable parent can be sidestepped by renaming the parent (`mv .claude .claude.old`, `mv .git .git.old`) and creating a new one. Linux blocks renaming a mount point but not its parents, and the rename reaches the host (verify on virtiofs).
- F3 (ward.md:81) Paths that do not exist at container creation get no protection: a project without `.git` where the agent runs `git init` or clones, the usually missing `.claude/settings.local.json`, a missing `.claude/`.
- F4 (ward.md:43, 81) The agent can create `sub/.git/` with `core.fsmonitor` or hooks anywhere in the project. Shell prompts and editors running `git status` on cd or open run it on the host. Matching the UID defeats git's `safe.directory` ownership check. Submodule configs and hooks under `.git/modules/<name>/` are writable.
- F5 (ward.md:43-44) Hook managers keep the code they run in the working tree: husky (`core.hooksPath=.husky`), lefthook, pre-commit. Project `settings.json` hooks often call `.claude/hooks/x.sh`. The read-only file calls a writable file.
- F6 (ward.md:128) Host auto-run surfaces are far from covered: approved `.mcp.json` servers, project `CLAUDE.md`, `.claude/skills|agents|commands`, `.vscode/tasks.json` `runOn: folderOpen`, `.venv/bin/*` via interpreter discovery or auto-activation, language servers running project code (rust-analyzer `build.rs` and proc macros, ESLint and Prettier JS configs, `go.mod` toolchain downloads).
- F7 (ward.md:56, 81, Linux) Passwordless sudo plus default caps (`CAP_CHOWN`, `CAP_FSETID`, `CAP_SETFCAP`, `CAP_MKNOD`) without a user namespace lets the agent create root-owned setuid binaries, file-capability binaries, or device nodes in the project that are real on the host. Fix: drop caps or userns-remap.
- F8 (ward.md:54, 78) Project at `$HOME` or above makes host `~/.claude/settings.json`, `~/.claude.json`, shell rc files, `~/Library/LaunchAgents`, `~/.ssh`, `~/.config/ward/` writable. That is host code execution. The warning is covered by Claude's full-screen UI.
- F9 (ward.md:115) "Nothing ward trusts is read from the project folder" is not enforced. `~/.config/ward/`, `~/.claude`, `~/.claude.json` can resolve inside the project or a RW extra mount (dotfiles repos, symlinks). The `$HOME` check misses `~/.config`, `~/.claude`, `/etc`, `/usr`.
- F10 (ward.md:43-47) Mount sources are resolved through host symlinks in agent-writable folders. Before the next creation the agent can replace `.git/config` or `.claude/settings.json` with a symlink to `~/.ssh/id_ed25519`, and Docker mounts the secret. Line 47 mounts the memory symlink's target. Fix: refuse symlinked sources or check targets.
- F11 (ward.md:5) Network reach is ignored: host services via `host.docker.internal` on Docker Desktop (verify loopback-only) or the Linux bridge gateway, the LAN, other containers including other ward containers, cloud metadata at 169.254.169.254. An unauthenticated local service means host code execution or data access.
- F12 (ward.md:97) The terminal is a second channel. Container output reaches the host terminal unfiltered, and any process running as the user can write to Claude's pty. OSC 52 sets the host clipboard. kitty `allow_remote_control` can launch host processes. iTerm2 and tmux have had RCE CVEs through escape sequences.

## Important

- F13 (ward.md:66) Docker does not kill an exec'd process when the client disconnects. If ward dies, Claude keeps running headless with permissions skipped, counts as a session, and the container never stops.
- F14 (ward.md:103) Any port the agent listens on is bound on host loopback automatically, uncapped and unconfirmed. Host tools then send credentials to the agent (`psql` to 5432 while the host DB is down, CLI OAuth redirects to `localhost:<port>`). It also blocks host services from those ports.
- F15 (ward.md:99, 103) Root in the container can write to the bridge or replace `ward-agent`, so all bridge input is attacker-controlled. No cap on listeners, no per-stream flow control. The ignore list is enforced inside the container.
- F16 (ward.md:46-47) Transcripts in the writable `<key>` folder are loaded into host sessions on resume, and the agent can forge turns in the JSONL. The threat model is applied unevenly.
- F17 (ward.md:46) The agent can replace a session `.jsonl` with a symlink to a host file, and host Claude writes through it on resume. Same for `.git/COMMIT_EDITMSG` and similar (verify open semantics).
- F18 (ward.md:84) Env vars are readable via `/proc/<pid>/environ` and inherited by Bash tool subprocesses, so the agent has the long-lived token. Limits does not list this.
- F19 (ward.md:48) `~/.claude.json` holds other projects' paths and data, MCP `env` secrets, account metadata. Copying it contradicts line 153. Its MCP servers start in the container and fail on host paths. Fix: filtered subset.
- F20 (ward.md:5, 102, 125, 148) Threat model inconsistency: line 5 is accidental, line 125 says no theft protection, the clipboard gate and line 148 defend against theft.
- F21 (ward.md:45-48, 62) Security depends on Claude Code's undocumented layout. `claude update` every launch can add new host-run surfaces. `CLAUDE_CONFIG_DIR` is ignored.
- F22 (ward.md:43) Git writes config via `config.lock` and rename, so with `.git/config` read-only, `git remote add`, `git config`, `push -u`, `checkout --track`, `submodule init`, `lfs install` fail.
- F23 (ward.md:38, 60) In a linked worktree or submodule, `.git` is a file, the bind fails, and the real git dir is not mounted. A subfolder has no `.git`. "Project folder" (cwd or git root) is undefined, so `ward` in `proj/src` creates a second container and key.
- F24 (ward.md:81) Missing bind sources: with `Binds`, Docker creates them as root-owned folders on Linux (a folder named `settings.local.json` breaks host Claude). With `Mounts`, creation fails. Step 4 creates the memory folder after applying mounts.
- F25 (ward.md:49, 120) RW extra mounts get none of the `.git` or `.claude` protections. Config drift after creation is not detected.
- F26 (ward.md:84) Exec env missing `PATH` with `~/.local/bin`, `SHELL`, `TZ`, `EDITOR`, `BROWSER`, `TERM_PROGRAM`, `TMUX`, `DISPLAY`/`WAYLAND_DISPLAY`, proxy and CA vars. Host `LANG=en_US.UTF-8` overrides the image's `C.UTF-8` with a missing locale.
- F27 (ward.md:91) Restoring termios is not enough. Ward must reset alternate screen, bracketed paste, mouse reporting, kitty keyboard flags, hidden cursor on Ctrl-Z, crash, container death, and re-enable after `fg`.
- F28 (ward.md:93) Linux sends SIGWINCH only on size change. Fix: resize to a different size, then back.
- F29 (ward.md:93) Key detection gaps: `modifyOtherKeys` (`CSI 27;5;118~`) under tmux `extended-keys`, kitty release events firing twice, sequences split across reads, `keybindings.json` rebinding image paste, `ward shell` losing Ctrl-Z job control.
- F30 (ward.md:66, 171) The watchdog has the same start/exit race the lock was rejected for. How init identifies exec'd processes, and whether bridge, update and copy execs count, is unspecified.
- F31 (ward.md:60, 82) No locking. Two first runs race on the container name. Concurrent runs stream into the same `~/.claude` and run `claude update` while another Claude runs.
- F32 (ward.md:103, 107) Every session's bridge scans the same `/proc/net/tcp`, so two sessions forward every port. How the status line and stand-ins find their own session's bridge is undescribed.
- F33 (ward.md:105-107) Project settings can define `statusLine`, which wins and is read-only. Status line refreshes only on events. No log file, so bind errors, bridge failures, clipboard denials are invisible. Pre-launch warnings are covered by Claude's UI.
- F34 (ward.md:99) "Anything else closes it": a `file://`, `vscode://`, `mailto:` link or an ungated `xclip -o` may kill ports and clipboard for the session.
- F35 (ward.md:101) URL opening is ungated and unlimited, including `127.0.0.1` and LAN addresses (host browser as CSRF relay). Forwarded-port pages share `localhost` cookies. Safe argument passing to `open`/`xdg-open` unspecified.
- F36 (ward.md:81, 129, 144) Rootless Docker maps the matched UID to a subuid, so files have a foreign owner. Docker Desktop for Linux runs a VM, so "no VM" and "behaves the same" are wrong. SELinux needs `:z`, and relabeling `$HOME` damages the host.
- F37 (ward.md:117-119) Half the host's cores can exceed the Docker Desktop VM's CPUs, and Docker rejects that, so creation fails on a typical Mac. No swap or disk limit. The process limit counts threads.
- F38 (ward.md:50) No `~/.gitconfig`: no user name or email, signing, or push credentials. Whether the agent commits or pushes is unstated.
- F39 (ward.md:34) `DOCKER_HOST`, `DOCKER_CONTEXT`, remote `ssh://`/`tcp://` contexts, OrbStack, colima, Podman ignored. Docker Desktop shares only `/Users`, `/Volumes`, `/private`, `/tmp`, `/var/folders` by default.
- F40 (ward.md:62, 81-82) No step installs Claude. First-run network need, timeout mid-install, pinned version or `autoUpdates: false` unspecified. Each volume downloads its own copy.

## Gap

- F41 (ward.md:103) Which address `ward-agent` dials is unspecified (`::1`-only servers). Host binds only `127.0.0.1`, so a host `[::1]` service coexists. Ports below 1024 fail on Linux. Remapped ports break Vite HMR, OAuth redirects, CORS.
- F42 (ward.md:103) The 1-second scan forwards short-lived listeners: Node inspector, test runners, language servers, Playwright.
- F43 (ward.md:81) macOS GID 20 is `dialout` in Debian. User created at build or by init at runtime (needs root)?
- F44 (ward.md:52) Container home path unspecified. If different from host home, absolute paths in `settings.json` and `plugins/installed_plugins.json` break. If the same, a project under `$HOME` on Linux nests in or replaces the home volume.
- F45 (ward.md:45) Allowlist misses `~/.claude/hooks/`, `output-styles/`, statusline scripts not named `statusline.sh`, `@path` imports from `CLAUDE.md`. Host deletions do not propagate. No loop or size guard on symlink following. `plugins/` can be large.
- F46 (ward.md:82) Archive API copies take owner from tar headers (root by default), so `~/.claude.json` could be unwritable. Extracting into an agent-controlled `~/.claude` may follow planted symlinks (verify).
- F47 (ward.md:48, 68) Whether `ward reset` overwrites the container's `~/.claude.json` or keeps it stale.
- F48 (ward.md:68-74, 84) Subcommands collide with passthrough args and Claude's own subcommands (`mcp`, `doctor`, `update`). No `--` rule.
- F49 (ward.md:91) Non-tty use (`ward -p ... | jq`, piped stdin) unspecified.
- F50 (ward.md:84) Host-path arguments (`--add-dir ../x`, `--mcp-config`, `--settings`) and screenshots dragged from Finder point at unmounted paths.
- F51 (ward.md:60) Path canonicalization: `/tmp` vs `/private/tmp`, `$PWD` vs `getcwd`, case-insensitive APFS. Docker name charset needs sanitizing.
- F52 (ward.md:64, 177) Building without the CLI means the deprecated legacy builder or an embedded BuildKit client. Debian base unpinned, so the Dockerfile hash does not identify contents. `ward build` reuses the tag, so the outdated check is undefined.
- F53 (ward.md:111) Login: no-token behavior, expiry, logout, rotation, API key / Bedrock / Vertex / `apiKeyHelper`, `setup-token` coverage unverified, Keychain ACLs tied to code signature, `security -w` puts the token on a command line.
- F54 (ward.md:45) Enterprise managed settings not copied, silently bypassing `disableBypassPermissionsMode` or failing.
- F55 (ward.md:42) Linux native modules in `node_modules` and `.venv` links break host runs on macOS and the reverse. Root-owned files from `sudo npm install` on Linux are hard to clean.
- F56 (ward.md:5, 101-102) Linux over SSH has no host browser or clipboard, forwarded ports need a second SSH forward, WSL needs `wslview`, multi-user Linux exposes `127.0.0.1` to other users.
- F57 (ward.md:129) A container escape on Docker Desktop lands in a VM sharing all of `/Users` RW.
- F58 (ward.md:140) Existing tools "not evaluated in depth". Anthropic's reference devcontainer with an egress firewall is not listed.
- F59 (ward.md:134-195) Not considered: egress allowlist, gVisor or Kata, network per project, snapshot before each session (the project has no `rm -rf` protection), scanning protected surfaces on exit.
- F60 (ward.md:132) Only clipboard is listed as unverified. Rename semantics, orphaned execs, `setup-token` scope, key derivation, Docker Desktop loopback are also unverified. No escape test suite or acceptance criteria.
- F61 (ward.md:47) Auto-memory writes fail with EROFS. User-visible behavior unspecified.
- F62 (ward.md:38, 46) Checkpoints, todos, prompt history live outside `<key>`, so `/rewind` and history do not carry across. Newer container Claude may write transcripts older host Claude cannot read.

## Nit

- F63 (ward.md:166) "including any new Claude Code" contradicts lines 62 and 167.
- F64 (ward.md:13) "Only one copy of Claude runs" contradicts line 60.
- F65 (ward.md:87) No way to turn prompts on. `--permission-mode` behavior undefined.
- F66 (ward.md:102) PNG-only means copied text passwords are never exposed, so the gate's stated reason does not match. Clipboard-write stand-ins and empty-clipboard responses unspecified.
- F67 (ward.md:68-72) Labels are immutable, so prune's "last used" needs timestamps. Volume deletion and effect on live sessions unstated.
- F68 (ward.md:82) Version check trusts the in-container binary. Compare a hash from the host.
- F69 (ward.md:101-102) Stand-in install location unspecified. `BROWSER` users (gh, Python `webbrowser`) missed.
- F70 (ward.md:66) Whether init forwards SIGTERM, so dev servers die after 10 seconds on stop.
- F71 (ward.md:64) Old images never removed.
- F72 (ward.md:11) Keychain and clipboard image on macOS need cgo or shelling out.
- F73 (ward.md:107) `ward-agent statusline` behavior when the original command is missing or times out, and where it is stored.
- F74 (ward.md:43) `.git/objects/info/alternates` can point host git at other repos, and `git repack -a` copies their objects in (verify).
- F75 (ward.md:82) Refresh during another session can expose a half-written `settings.json`.
- F76 (ward.md:93) While ward is suspended, forwarded connections and bridge requests stall.

Reviewer's summary: F1-F5 and F10 bypass every git and project-Claude protection. F7 (Linux) and F12 (terminal) give host code execution. F11 leaves the rest of the machine reachable over the network.
