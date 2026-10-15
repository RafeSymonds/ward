# Goldfish A: comprehension, round 1 (2026-10-14)

Agent: generalist. Full report as returned.

## 1. What this document is trying to accomplish

ward lets one developer on macOS or Linux run Claude Code with permission prompts off (`--dangerously-skip-permissions`, L84) while limiting the damage it can do to the current project folder (L5). Claude runs inside a Docker container, one per project, that sees only the project folder and any extra folders the user configures. The goal is for this to still feel like running `claude` locally. The user keeps their own config, sessions can be resumed from either side, dev servers are reachable on the host, links open in the host browser, and clipboard images can be pasted (L5, L13). The threat model is damage to the host, not theft (L125). The main isolation goal is that the agent cannot change anything that later runs on the host automatically: git hooks and config, Claude hooks, MCP commands and memory (L43-48, L128, L140).

## 2. How the system currently works (as it relates to this feature)

- No ward code exists. `/Users/rafe/other/ward` contains only `docs/designs/ward.md` and `docs/egm/ward/*`. There is no Go module, no Dockerfile and no source. The design doc references no repository files, so this is a new build. I did not read the `docs/egm` files because the doc does not reference them.
- The doc depends on outside systems whose behavior it states but does not prove. I could not verify any of the following from referenced files:
  - Claude stores sessions and memory under `~/.claude/projects/<key>`, keyed by the project's absolute path (L38, L46-47). How `<key>` is derived from the path is never stated.
  - Claude writes history, session state, shell snapshots, file checkpoints and task files under `~/.claude` (L52, L151).
  - `~/.claude.json` lists MCP server commands that run on the host, and Claude writes this file constantly (L48).
  - Hooks in user and project settings run on the host (L44, L150). Git runs hooks and the commands named in `core.fsmonitor` and `core.hooksPath` (L43).
  - Docker Desktop's socket is under `~/.docker/run/` (L34). Published ports are fixed when a container is created (L183). Sharing a Unix socket through Docker Desktop is unreliable (L97).
  - `claude setup-token` produces a token that is independent of the normal login (L111). The status line command receives input from Claude (L107).
- The doc itself marks some things as unverified:
  - how Linux Claude reads clipboard images (L132)
  - whether refresh tokens rotate (L159)
  - whether `setup-token` works without a browser in a container (L161)
  - existing tools (L140) and other container runtimes (L144), which were not evaluated

## 3. What changes will this feature introduce

Proposed changes:

- C1 Two programs. `ward` (host) and `ward-agent` (container) are built from one Go module. The host binary embeds Linux amd64 and arm64 builds of `ward-agent` (L11).
- C2 Launch. The user types `ward` instead of `claude`. ward stays in the foreground and exits when Claude exits. Only one Claude runs, and it runs in the container (L13).
- C3 Docker access. ward uses the Docker API at the current `docker context` endpoint, not the docker CLI (L34, L177).
- C4 Mount table (L40-50): project folder read-write at the same absolute path. `.git/hooks`, `.git/config` and the project's `.claude/settings*.json` read-only. An allowlist of user `~/.claude` files copied in on every run. `~/.claude/projects/<key>` bound read-write and live, with `memory/` mounted read-only over it. `~/.claude.json` copied in once, at container creation. Extra folders read-only by default. The container's own `~/.claude` is a writable folder in the home volume (L52).
- C5 Home warning. If the project folder is `$HOME`, `/`, or any folder above `$HOME`, ward warns and continues (L54).
- C6 Container posture. Normal user with passwordless `sudo`. Not privileged, no Docker socket, CPU, memory and process limits (L56).
- C7 Lifecycle. One container per project, named `ward-<dirname>-<pathhash>`, found by a path label, shared by sessions in the same folder (L60). Home in a separate volume. Claude installed there with its native installer, `claude update` before each launch with a timeout (L62). Base image Debian slim, built locally from an embedded Dockerfile, tagged with the Dockerfile's hash (L64). `ward-agent init` is the main process, reaps exited processes, stops the container after 60 seconds with no `docker exec` processes (L66).
- C8 Subcommands. `reset`, `rm`, `prune --older-than`, `stop`, `ls`, `build`, `shell` (L68-74) and `login` (L111).
- C9 Session sequence. Eight steps (L78-85). Token and terminal variables passed as environment variables through the API (L84).
- C10 Terminal relay. Raw mode, resize forwarding, terminal restore on every exit path (L91). Ctrl-Z suspends ward on the host and is never sent to Claude. Ctrl+V is passed to Claude and opens a 2-second clipboard window. Both keys ignored inside bracketed paste and recognized in kitty keyboard protocol encoding (L93).
- C11 Bridge. Several streams multiplexed over a second `docker exec` (L97). Fixed request list, anything else closes the bridge (L99): open an http or https URL via an `xdg-open` stand-in (L101). Read a PNG clipboard image, gated by Ctrl+V, via `xclip` and `wl-paste` stand-ins (L102). Port opened or closed, `ward-agent` polls `/proc/net/tcp*` every second, host listens on `127.0.0.1` at the same port or the next free one (L103).
- C12 Status line. The status line command in the container's `settings.json` copy is replaced with `ward-agent statusline`, which runs the user's original status line and appends the port mappings (L107).
- C13 Login. `ward login` stores a pasted `setup-token` token in the Keychain, the Secret Service, or a user-only file under `~/.config/ward/` (L111).
- C14 Config. `~/.config/ward/config.toml` only, global defaults and per-project overrides: CPU (default half the cores), memory (default half of what Docker reports), processes (default 4096), extra mounts, ports to ignore. Nothing read from the project folder (L115-121).
- C15 Out of scope. Limits section (L125-132). Deferred: other harnesses (L141), per-session containers (L169), per-project Dockerfiles (L170), remote sessions (L190-195).

Ambiguities and gaps (reviewer numbering):

- F1 Protected files that do not exist yet. Memory is pre-created (L47), but `.git/hooks`, `.git/config`, `.claude/settings*.json` have no such handling when missing (L43-44). If `.git` is absent, the agent could create `.git/config` with `core.fsmonitor` and host git would run it. Repos where `.git` is a file (worktrees, submodules) are not covered.
- F2 Existing `core.hooksPath` pointing into the project leaves that hooks folder writable. L43 protects only `.git/hooks` and `.git/config`.
- F3 Inconsistent rule for content loaded into host sessions. Memory is read-only because it is loaded as instructions (L47), but project `CLAUDE.md`, other `.claude/` contents, and live transcripts (L46) are writable. The doc does not say whether the project folder can hold MCP definitions equivalent to `~/.claude.json`. L128 does not settle which rule applies.
- F4 Bridge trust boundary. With passwordless `sudo` (L56) the agent controls `ward-agent`. The doc does not say the host validates every bridge message, how the version check (L82) resists a tampered binary, or limit agent-sent "port opened" messages that make the host open listeners (L103).
- F5 Bootstrap order. Step 4 starts the container with `ward-agent init` as main process, but step 5 copies `ward-agent` in (L81-82). On first creation the binary is missing. Storage location and upgrade of a running init are unspecified.
- F6 Several sessions in one container (L60): each has its own bridge (L83), so a dev server could be forwarded twice. Which bridge the stand-ins reach, whose Ctrl+V gate applies, and how `ward-agent statusline` learns mappings are unspecified. A new session's prepare step (L82) overwrites config while another runs.
- F7 First install of Claude (L62) is missing from the step list (L78-85). `claude update` timeout value and failure behavior unspecified.
- F8 Changes after creation. Mounts, limits, `~/.claude.json` copy are fixed at creation (L48, L81). Whether config edits apply without `ward reset`, whether `reset` re-copies `~/.claude.json`, and whether the copy is filtered to this project are unspecified.
- F9 Resume claim. Only transcripts are live (L46). Checkpoints, session state, history stay in the container (L52). Full resume from either side (L38) is unverified.
- F10 Token exposure. The env var token (L84) is readable by the agent but L127 does not list it. Variable name and behavior with no stored token are unspecified.
- F11 Project folder is `$HOME`. Host `~/.claude`, `~/.claude.json`, `~/.config/ward/config.toml` then sit inside the writable mount, defeating L45-48 and L115. The warning (L54) does not say these protections are lost.
- F12 Container user. GID, username, home path, and how a user is added to a shared image at creation are unspecified. On Linux, files created with `sudo` in the project folder may be root-owned on the host.
- F13 Command-line parsing between ward subcommands (L68-74) and pass-through Claude args (L84) is unspecified.
- F14 Idle watchdog. How `ward-agent init` detects exec'd processes (L66) is unspecified. Start race: container found running at step 4 idle-stops before step 5 or 6.
- F15 Read-only project settings: a status line in project settings cannot be rewritten (L44, L107). Claude's writes to `settings.local.json` would fail, the same failure used to reject a read-only `~/.claude` (L151).
- F16 Host-side clipboard image read per OS and the no-image response are unspecified (L102).
- F17 Behavior when Claude tries to write read-only memory is unspecified (L47).
- F18 `ward prune`: how "used" is tracked, `<age>` format, whether home volumes are deleted (L70).
- F19 No TOML schema, list-merge rules for overrides unspecified (L115-121). Precedence of `DOCKER_HOST` and `DOCKER_CONTEXT` vs current context unspecified (L34).
- F20 Remapped ports and links. If 3000 is forwarded as 3001 (L103), an open-URL request for `localhost:3000` (L101) reaches the wrong host port.
- F21 Whether host-side deletions are removed from the container's allowlist copy (L45, L82).
- F22 Bridge framing unspecified (L97). Session behavior after the bridge closes on a bad request (L99) unspecified.
- F23 Whether the container can reach services on the host or LAN is not discussed beyond theft (L127).

Reviewer's priority: F1 through F5. F1-F4 each leave a way for the agent to change what runs or loads on the host. F5 stops the first run from starting.
