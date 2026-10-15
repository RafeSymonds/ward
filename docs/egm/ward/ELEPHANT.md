feature: ward
design_doc_path: docs/designs/ward.md
date_opened: 2026-10-14
current_step: 5
last_update: 2026-10-14

## Context loaded

Greenfield. No prior code or design docs. Host facts gathered for the design:

- `~/.claude/` (870M) holds settings.json, statusline.sh, skills/, agents/, plugins/, projects/ (session transcripts keyed by absolute path), sessions/, history.jsonl.
- `~/.claude.json` (377K) is a separate global config file in `$HOME`, outside `~/.claude/`.
- Claude Code OAuth credentials live in the macOS Keychain (`Claude Code-credentials`), not in `~/.claude/`.
- `~/.claude/CLAUDE.md` and `~/.claude/projects/-Users-rafe/memory` are symlinks into `~/claude-memories/`.
- Prior art: Docker `sbx` (`sbx run claude <dir>`), yolobox, aibox, claude-code-sandbox, servusdei2018/sandbox.

## Interview decisions

- Name: `ward`. Repo: `~/other/ward`, remote `RafeSymonds/ward` (private).
- Threat model: protect against accidental deletion/damage first. Secret theft is a nice-to-have.
- `$HOME` is not mounted. Mount cwd plus `~/.claude` plus a configurable allowlist.
- Harness scope: Claude Code only for now.
- Container limits: CPU cap (memory cap proposed).
- Agent gets sudo inside the container. No `--privileged`, no Docker socket.
- Running in `$HOME` or above: F3 decided as warn, not refuse, for now.
- Remote execution: O12 chosen. Remote host mounts the local folder via sshfs in passive mode over ward's SSH connection, served by the Mac's sftp-server. Remote host bind-mounts it into an unprivileged container. Rejected for now: O13 Mutagen two-way sync.
- `~/.claude`: O14 chosen. Read-only mount, with session state (`projects/`, `sessions/`, `history.jsonl`) writable. Rejected: O15 ward-owned copy (sessions and config drift from host), O16 full RW (agent can plant host-side hooks).
- Auth: O17 chosen. One-time `claude setup-token`, ward stores the token in its own Keychain entry and injects it as an env var. Rejected: O19 copying the host Keychain login (R1: shared refresh token can log one side out).
- F7 must be fully closed. Remaining gap under O14 is `projects/<key>/memory`, which feeds future host sessions. Proposed: nested mounts, `~/.claude` RO, `projects/<key>` RW, `projects/<key>/memory` RO.
- F11: read as approved. `.git/hooks`, `.git/config`, project `.claude/settings*.json` mounted RO by default (pending user confirmation).
- Lifecycle: O21 chosen. One persistent container per project. Stopped when not in use. Explicit delete commands plus configurable cleanup.
- F12 approved: each container gets its own copy of `~/.claude.json`, seeded from host at container creation, never copied back.
- F13 approved: all ward config lives in `~/.config/ward/`, keyed by project path. Nothing ward trusts lives in cwd.
- Concurrency: O22 (shared container per project) for v1. User wants O23 (separate containers) configurable later; design deferred.
- O27 chosen: keep the container, stop it when the last session exits, update Claude Code before each launch, home folder in a volume. Rejected: O26 recreate every start (loses sudo installs).
- Concurrency setting: proposed building O22 only and adding the setting when O23 is designed. User has not replied.
- Image: O24 approved for v1 (one base image). O25 per-project Dockerfile deferred.
- Host OS: ward must run on macOS and Linux.
- Q11 approved: `--dangerously-skip-permissions` passed by default.
- O28 approved, replaces O14: container has its own writable `~/.claude` in the home volume, ward copies a config allowlist in each run, only `projects/<key>` is a live host bind (memory RO on top). Reason: F14, Claude writes to `~/.claude` during normal runs.
- Remote: deferred entirely. User rejected O12 for cloud and is not satisfied with O29 either. Revisit after local mode is designed.
- O31 approved: ward stays in the foreground and serves the bridge. O32 daemon rejected. O30 watchdog kept as backstop.
- Q12 approved: port forwarding is in v1, via the O31 bridge.
- Proposal draft 2: `docs/egm/ward/proposal-draft-2.md`. Adds `ward-agent` (PID 1, bridge, stand-ins), bridge over a `docker exec` stream (F20), Docker API via current context, R7, R8.
- O33 approved: ward wraps the status line command in the container's settings copy and appends forwarded ports. Rejected O34 terminal title (hidden under tmux unless configured).
- Host runtime: Docker Desktop 29.2.1, linux/arm64. No OrbStack, Colima, Podman, or Apple `container`.

## Open questions

- Q1: recorded as remote mode (O12) plus correct `~/.claude` handling. Not confirmed by user.
- Step 4 first draft: `docs/egm/ward/proposal-draft-1.md`. Superseded in part by the decisions above.
- Terminal path explained: one Claude, inside the container. Ward replaces itself with `docker exec -it`, which relays bytes to a PTY in the container.
- O30 proposed: container stops itself via an idle watchdog as PID 1, replacing the host flock in step 8.
- F16: clipboard image paste, opening a browser, IDE integration, and host-app MCP servers do not cross the container boundary.
- F17: ward forwards TERM, COLORTERM, LANG. Image needs ncurses-term (user runs TERM=tmux-256color).
- R6: Ctrl-Z suspends Claude with no job-control shell to resume it.
- User asked whether ward can be long-running. Proposed O31: ward stays in the foreground for the session, owns the terminal relay, and serves a narrow host bridge (URL open, paste-gated clipboard, port forwarding, local Ctrl-Z). Rejected O32: background daemon (install units, upgrade and crash handling, no v1 need). O30 watchdog kept as backstop.
- F18: Docker port publishing is fixed at container create, so dynamic port forwarding needs a ward-side relay.
- F19: the container-to-host bridge is the most security-sensitive part. Fixed message types only, never a generic exec.
- F15: Keychain is macOS-only. Linux needs Secret Service, with a 0600 file fallback for headless hosts.
- R5: on Linux hosts, containers share the host kernel and docker group membership is root-equivalent.
- R2: persistent containers drift from the base image (old Claude Code, old tools) until reset.

## Section status

- Problem: approved 2026-10-14. Theft protection and remote sessions intentionally left out of the problem statement (secondary goal and deferred).
- Technical Plan: approved 2026-10-14. Decisions made while writing and approved: D1 `ward login` takes a pasted `claude setup-token` token (headless setup-token in the container unverified), D2 defaults half CPUs / half Docker memory / 4096 processes, D3 `ward-agent statusline` wraps the user's status line, D4 forwarded ports bind 127.0.0.1 only, D5 explicit Limits subsection.
- Alternatives: approved 2026-10-14.
- Detailed Implementation: not yet written. User asked to run Goldfish round 1 on sections 1-3 first.

human_review_gate: pending
next_step: /goldfish docs/designs/ward.md (round 1 on sections 1-3), then write Detailed Implementation

## Goldfish round 1 decisions (2026-10-14)

- G7 approved: per-project container-only folders in ward config, volume-backed (node_modules, .venv, target/).
- G6 approved: every session runs under `ward-agent session`, tied to its bridge, with a session ID used by stand-ins, status line, and port forwarding.
- G5 approved: ward drops OSC 52 from container output.
- G4 decided: no access to the host or local network by default. Allowlist configuration may come later.
- G3 approved: rootless Docker on Linux runs the agent as container root (maps to host user). Rootful Docker on Linux runs the agent without sudo. macOS unaffected.
- G2 decided: no project snapshots. User is not worried about deletion inside the project.
- G1 decided: the project folder is fully writable, including `.git` and `.claude/`. Read-only applies to host paths outside the project. User: "the folder should be write and readonly should be the other things." Reason: tested on Docker Desktop that RO sub-mounts are bypassable (F21 rename `.git` and plant a hook, F22 whole-`.git` RO blocks commits, F23 nested repo with core.fsmonitor runs on host `git status`), and the user rejects bypassable read-only. Rejected: RO sub-mounts, O35 filtered sync, O36 FUSE filter, O37 mount hardening. Q16 protected list is moot. Consequence for Limits (R9): the agent can change git hooks, git config, project Claude settings and other files host tools run automatically.
- Q19 decided: "anything claude needs should be write and then read only can be other things." `~/.claude/projects/<key>` stays a live RW bind. Read-only applies to extra mounts. Host config stays copied in, not mounted (O28, F12).
- Q21 open: under the Q19 rule, `projects/<key>/memory` would become RW, reversing F7. Proposed yes, because project CLAUDE.md and `.claude/settings.json` are now writable (G1) and host sessions load them the same way.
- G8-G22: proposed fixes, not yet explicitly approved.
