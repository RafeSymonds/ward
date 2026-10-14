# ward: first-draft technical proposal

Prose and block diagrams, not code. This is the Step 4 draft for debate. The design doc at `docs/designs/ward.md` is written only after this settles.

## Shape

`ward` is a single Go binary on the host. It shells out to the `docker` CLI for everything. `docker exec -it` already handles raw terminal mode, window resize, and signals, so ward never touches the terminal itself. Docker contexts also give phase 2 a remote engine without new plumbing.

```
 Mac
 ┌────────────────────────────────────────────────────────────┐
 │ ward                                                        │
 │   safety check → config → shared lock → image → container   │
 │   → sync config in → update claude → exec claude → stop?    │
 │ Keychain: "ward" token (from claude setup-token)            │
 └──────────────┬─────────────────────────────────────────────┘
                │ docker CLI
 ┌──────────────▼─────────────────────────────────────────────┐
 │ Docker Desktop VM                                            │
 │  container ward-<dirname>-<pathhash>  cpus/memory/pids caps  │
 │   <cwd at same absolute path>              RW  (bind)        │
 │   <cwd>/.git/hooks, .git/config            RO  (bind)        │
 │   <cwd>/.claude/settings*.json             RO  (bind)        │
 │   /home/agent                              volume, persists  │
 │     .claude/            config copied in from host each run  │
 │     .claude/projects/<key>                 RW  (bind, host)  │
 │     .claude/projects/<key>/memory          RO  (bind, host)  │
 │     .claude.json        copy seeded from host at create      │
 │     .local/bin/claude   native install, updated each run     │
 │   <extra mounts from config, same path>    RO default        │
 └────────────────────────────────────────────────────────────┘
```

## One launch, step by step

1. **Safety check.** The project is the current directory. Ward refuses to run in `$HOME`, `/`, or any directory above `$HOME` unless `--allow-home` is passed.
2. **Config.** Ward reads `~/.config/ward/config.toml`. It holds global defaults (cpus, memory, pids limit, extra mounts) and per-project overrides keyed by absolute path. Nothing is read from the project folder (F13).
3. **Lock.** Ward takes a shared lock on `~/.config/ward/state/<pathhash>.lock`. Every running session holds one.
4. **Image.** Ward embeds the Dockerfile for the O24 base image (Debian slim, git, ripgrep, build tools, Node, Python, sudo, tini). The tag is a hash of that Dockerfile. If the tag is missing, ward builds it locally.
5. **Container.** Ward finds the project's container by label. If none exists, it creates one with the mount table above, the resource caps, a user whose UID matches the host user with passwordless sudo, and an idle main process. On create it copies `~/.claude.json` in (F12) and creates the host memory folder if missing, so the agent can never create it on the host side of the RW mount. If the container was built from an older image, ward warns and points at `ward reset` (R2). If the container is stopped, ward starts it.
6. **Prepare.** Ward streams an allowlist of host config into the container's `~/.claude`: `settings.json`, `CLAUDE.md`, `skills/`, `agents/`, `commands/`, `plugins/`, `statusline.sh`, `keybindings.json`. Symlinks are followed on the host, so the symlinked `CLAUDE.md` arrives as content. Anything the container wrote to its copy last run is overwritten. Ward then runs `claude update` (or the installer on first run) with a timeout so a slow network never blocks launch.
7. **Run.** Ward runs `claude` inside the container through `docker exec -it`, in the project's path, passing any extra arguments through. The token reaches the container as an environment variable inherited from ward's own process environment, never as a command-line argument, so it does not show up in `ps`.
8. **Exit.** Ward releases its shared lock and tries an exclusive lock without waiting. If it gets it, no other session is running, and ward stops the container. A stopped container uses disk only.

## Commands

- `ward [-- claude args]` runs Claude for the current directory.
- `ward login` runs `claude setup-token` on the host and stores the token in the Keychain.
- `ward shell` opens a bash shell in the project's container.
- `ward ls` lists containers with project path, state, image age, and disk use.
- `ward stop`, `ward rm`, `ward reset` act on the current project's container. `reset` keeps the home volume, `rm` deletes it too.
- `ward prune --older-than <age>` removes containers not used within that age.
- `ward build` rebuilds the base image.

## Phases

- **Phase 1, local mode.** Everything above.
- **Phase 2, remote mode (O12).** `ward --host desktop` uses a Docker context pointed at the remote engine over SSH. Ward opens one SSH channel that runs `sshfs` in passive mode on the remote host, served by the Mac's `sftp-server` on the other end. The remote host mounts it and Docker binds the same subpaths as local mode. The container stays unprivileged. The remote host needs Docker, sshfs, and FUSE.
- **Later.** O23 separate containers, O25 per-project Dockerfile, other harnesses.

## Corrections and open items

- **F14.** O14 as written breaks Claude. Claude writes to `~/.claude` during normal runs: today's timestamps show writes to `history.jsonl`, `sessions/`, `session-env/`, `shell-snapshots/`, `file-history/`, `tasks/`. A read-only mount turns all of those into errors. **O28** replaces it: the container has its own writable `~/.claude` in the home volume, ward copies the config allowlist in each run, and only this project's session folder is a live bind. It keeps O14's goal (the agent cannot change host config) and Claude can write normally.
- **Q11.** Ward passes `--dangerously-skip-permissions` by default. The container is the safety boundary, so permission prompts add friction without adding protection.
- **Q12.** Port forwarding for dev servers. The agent runs `npm run dev` and you want it in your Mac browser. In or out of v1?
- **F3.** Refuse in `$HOME` with `--allow-home`, instead of warning. Still unconfirmed.
- **R3.** Host config can reference macOS-only things (Homebrew paths in hooks, `statusline.sh` calling macOS tools). Those fail inside the Linux container.
- **R4.** In phase 2, `sftp-server` has no path restriction. The remote host can read any file your Mac user can. The container still sees only its binds. Acceptable only because the remote host is yours.
