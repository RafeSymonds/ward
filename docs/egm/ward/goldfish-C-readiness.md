# Goldfish C: implementation readiness, round 1 (2026-10-14)

Agent: architect. Full report as returned. Question numbers are the reviewer's own.

NOT-READY

The design doc is the only file in the tree besides `docs/egm/` and references no project files, so every question cites `docs/designs/ward.md`.

## Build and packaging

1. Go module path, directory layout (`cmd/ward`, `cmd/ward-agent`?), minimum Go version? (Overview)
2. How are the Linux `ward-agent` binaries built before the host build embeds them (Makefile, `go generate`, goreleaser)? How is a release distributed? (Overview, Container lifecycle)
3. Is "the right" `ward-agent` chosen by host arch, Docker daemon arch, or image arch? (Overview)
4. What does "its version differs" compare: ldflags version, content hash, other? A dev build with a fixed version string would never be recopied. (step 5)
5. Which third-party dependencies are allowed: Docker Go SDK vs raw HTTP, yamux/smux vs custom framing, TOML library, keyring library? (Overview, Bridge, Login, Configuration)

## Docker endpoint

6. Context resolution order. Do `DOCKER_HOST` and `DOCKER_CONTEXT` override? TLS contexts supported? (Overview)
7. When the current context is a remote engine (`ssh://`, `tcp://` elsewhere), where host paths do not exist: refuse, warn, continue? (Overview, Remote sessions)
8. Are Colima, OrbStack, and Docker Desktop for Linux supported endpoints? (Overview, Alternatives)

## Project identity

9. Is the project folder the cwd or the git root when run in a subdirectory? If cwd, does each subdirectory get its own container? (Overview, Container lifecycle)
10. Physical realpath (e.g. `/private/tmp/x`) or logical `$PWD`? Cross-side resume depends on matching what host Claude records. (What the agent can see)
11. What algorithm turns the path into `<key>`? It must match Claude Code's encoding exactly, including truncation or hashing of long paths. Computed from the project folder or Claude's working directory? (mount table)
12. How is `<dirname>` sanitized for Docker's name charset? Which hash and length? What label key? These persist. (Container lifecycle)
13. Home volume name, and how it is found again? (Container lifecycle)

## Container user and environment

14. Is container home `/home/<user>` or the host `$HOME` path? Copied hooks, `statusLine` commands and plugin metadata can hold absolute `/Users/<name>/.claude/...` paths. (mount table, status line)
15. Where and when is the user created? A UID build arg breaks the Dockerfile-hash tag. Creating it at start in `ward-agent init` as root is the alternative. Username and GID? macOS GID 20 is `dialout` in Debian. Where is the sudoers entry? (step 4)
16. What does the Claude exec get beyond `TERM`, `COLORTERM`, `LANG`, token? `PATH` must include the native installer's bin. `HOME`, `USER`, `SHELL`, `TZ`? (step 7)
17. Is `LANG` passed as-is or forced to `C.UTF-8`? Host `en_US.UTF-8` has no locale in Debian slim. (Container lifecycle, step 7)
18. Token environment variable name? (step 7, Login)
19. Which step installs Claude Code on first run? Behavior when the install fails or the machine is offline? (Container lifecycle, step 5)
20. `claude update`: timeout value, behavior on timeout or failure, is output shown, two sessions running it at once, update while another session's Claude runs? (Container lifecycle, step 5)

## Mount table

21. If `.git` is a file (worktree, submodule), how are hooks and config protected? (mount table)
22. If `core.hooksPath` points into the working tree (e.g. `.husky/`), those hooks are writable and run on the host. In scope? (mount table, Limits)
23. Is it accepted that git commands that write config fail in the container (`git remote add`, `checkout -b --track`, `submodule init`, `git config`)? (mount table)
24. Project `CLAUDE.md`, `.mcp.json`, `.claude/commands`, `agents`, `skills` (which can carry hooks and `!` shell execution), and live transcripts replayed by host `--resume` are writable and loaded by host sessions. The memory row's reason applies to all of them. Which are in scope? (mount table)
25. Other auto-running host files in scope: `.envrc` (direnv), `.vscode/tasks.json` with `runOn: folderOpen`? (Limits)
26. The `~/.claude` allowlist omits `hooks/` and `output-styles/`, but user hooks in the copied `settings.json` often call scripts in `~/.claude/hooks/`. Intended? Should user hooks run in the container at all? (mount table)
27. Does the per-run refresh mirror (delete agent-added files) or only overwrite? Does a refresh run while another session uses the files? (mount table)
28. Symlink handling: recursive directory symlinks, cycles, dangling targets? (step 5)
29. Is `~/.claude.json` copied as-is or sanitized? After `ward reset`, recopied or kept? Missing host file? (mount table, step 4)
30. Is the memory folder checked every run or only at creation? User later deletes it or makes it a symlink? Does Claude tolerate a read-only memory folder? (memory row, step 4)
31. Does ward create `~/.claude/projects/<key>` on the host as a bind source? What if `~/.claude` does not exist? (mount table)
32. Extra mounts: `~` expansion, relative paths, missing path error or skip? Overlap with protected paths checked? A RW extra mount of `~/.claude` or `~/.config/ward` undoes the design. (Configuration)
33. Mounts fixed at creation. On config change: detect drift and warn, apply limits via container update, or require `ward reset`? (Configuration, Container lifecycle)
34. Refuse or warn when the project folder is or contains `~/.claude` or `~/.config/ward` (which may hold the token file)? (What the agent can see, Login)

## Configuration and limits

35. "Half the host's cores": host CPU count or Docker VM NCPU? Which `docker info` field is "memory Docker reports"? (Configuration)
36. TOML schema: key names, units (`"8g"` vs bytes), per-project lists replace or extend, project key exact or prefix match, `XDG_CONFIG_HOME`, missing or invalid config? (Configuration)

## Image

37. Debian release, pinned base? Node source and version? Which packages are "build tools"? (Container lifecycle)
38. Image name format? Does `ward build` skip cache or pull fresh? A rebuild keeps the same hash tag, so how is "older image" detected: tag, image ID, label? Who removes old images? (Container lifecycle)
39. Are stand-ins baked into the image as symlinks to a fixed `ward-agent` path, or installed during prepare? Does `ward-agent` live in the rootfs or the home volume? (Bridge, step 5)

## Lifecycle

40. How does `init` detect exec'd processes (e.g. PPid 0 in its PID namespace)? Poll interval? On idle exit, SIGTERM children with grace? Handling of `docker stop`? (Container lifecycle)
41. Races: container idle-stops between ward finding it running and the first exec. Name conflict when two concurrent `ward` runs create the same container. (Container lifecycle, step 4)
42. How does `prune` know last use, given labels are immutable? Age format (Go durations have no `d`)? Does it delete volumes, images, containers whose folder is gone? (Container lifecycle)
43. Should `rm`, `reset`, `stop` refuse, prompt, or force when other sessions are active? (Container lifecycle)
44. `ls` disk use: writable layer, volume, or both? "Image age": age or outdated? (Container lifecycle)
45. `ward shell`: which shell and user? Does it create or start the container, run prepare, start a bridge? Does it use the relay with Ctrl-Z interception, breaking job control in the shell? (Container lifecycle)

## CLI

46. How are ward subcommands told apart from Claude arguments (`ward build`, `ward --help`, `ward -p "..."`)? `--` separator? ward's own flags, e.g. `--version`? (step 7)
47. Non-TTY stdin or stdout (`ward -p "..." | jq`): exec without TTY, or refuse? (Terminal relay)
48. Exit code when Claude dies by signal? Codes for ward's own setup failures? (step 8)
49. Where do runtime errors go while Claude owns the screen (bridge failure, port bind failure, URL open failure on headless Linux)? Log file? Image build output shown? (Status line)

## Terminal relay

50. Under tmux with `extended-keys`, Ctrl-Z and Ctrl-V can arrive as modifyOtherKeys (`CSI 27;5;122~`). Must ward recognize these? (Terminal relay)
51. The copied `keybindings.json` can move image paste off Ctrl+V, so the gate never opens. Does ward read the binding? (Terminal relay, mount table)

## Bridge

52. Wire protocol: framing, encoding, multiplexing? (Bridge)
53. With several sessions in one container, how do stand-ins and `ward-agent statusline` reach their own session's bridge: in-container Unix socket, session ID env var on the exec, other? (Bridge, Status line, Container lifecycle)
54. All bridges see the same listening sockets. Does every session forward every port (second session gets 3001 for the same server), or only ports its own processes opened? (Bridge)
55. After "anything else closes it": session continues without bridge, bridge restarts, or session ends? (Bridge)
56. Rate limit on opening URLs? (Bridge)
57. When 3000 is remapped to 3001, Claude still opens `http://localhost:3000`. Should the URL handler rewrite forwarded ports? (Bridge, Status line)
58. Forwarding: agent always connects to `127.0.0.1:<port>`? Servers on `::1` only or the container's `eth0`? Skipped ports (Claude's own, below 1024, ephemeral range)? Upper bound for "next free"? Close open connections when the listener goes? (Bridge)
59. Host clipboard image read: `pbpaste` is text-only, macOS needs cgo/NSPasteboard or `osascript`, cgo conflicts with a single cross-compiled binary. On Linux, is host `wl-paste`/`xclip` a runtime dependency? Non-PNG (TIFF)? (Bridge, Overview)
60. One Ctrl+V allows one clipboard request, or unlimited within two seconds? (Bridge)
61. Stand-in handling of other invocations: `xclip -t TARGETS -o` probes, `xclip -i` or `wl-copy` writes, `xdg-open` on a file path? (Bridge)

## Status line

62. Where is the user's original command recorded? A project `.claude/settings.json` with its own `statusLine` overrides the rewritten one and is read-only, so ward's info disappears. (Status line)
63. Same line or new line? Port changes show only when Claude next refreshes the status line. Accepted? (Status line)

## Login

64. With no stored token: start login, error, or start unauthenticated? Unauthenticated leads to `/login` storing credentials in the home volume, a rejected alternative. (Login, step 7)
65. `ward login` details: hidden input, format check, Keychain service and account names, detecting a missing Secret Service, fallback file name and format, `ward logout`, expired token reporting? (Login)
66. The agent can read the token through `/proc`. Is theft of a long-lived subscription token accepted under "does not stop theft"? (Limits, step 7)
67. API key, Bedrock, Vertex users (`env` or `apiKeyHelper` in copied settings) supported? (Login)

## Git and network inside the container

68. No `~/.gitconfig`, SSH agent, or `gh` credentials, so `git commit` and push fail. Expected to work? (mount table)

## Scope and acceptance

69. Must the "use an existing tool" evaluation happen before building? (Alternatives)
70. Acceptance criteria: which hosts and engines must pass, startup time target for "feels like running Claude" (`plugins/` streamed every run), which behaviors need automated tests? (Problem, step 5)

## Contradictions and security gaps (highest impact)

71. If `.claude/settings.json`, `settings.local.json`, `.git/hooks` or `.git/config` does not exist at creation, nothing is mounted there. The agent can create the file or `git init` and plant hooks or `core.fsmonitor`. A repo initialized later stays unprotected until `ward reset`. Placeholders? (mount table)
72. RO `.git/config` and `.git/hooks` mounts do not cover: `mv .git .git-old && git init` (rename only refuses when the renamed path itself is a mount point), writing `.git/commondir` to redirect config lookup, a nested `subdir/.git/config` with `core.fsmonitor` that a host shell prompt's git status runs. The rename also applies to `.claude/`. Verified? Intended protection? (mount table)
73. When the project is `$HOME`, the project mount collides with the home volume and `~/.claude` rows. The agent would get host `~/.claude` and `~/.config/ward` RW. Mount layout for this case is undefined. (What the agent can see)
74. Under rootless Docker, a container UID matching the host UID maps to a host subuid, so files the agent writes are not owned by the user. Run as container root under rootless? How is rootless detected? (Limits, step 4)
75. On Docker Desktop `host.docker.internal` reaches host loopback services, and on Linux the bridge gateway reaches host services. Contradicts "exposes nothing on the host that the container could connect to on its own". Blocking in scope? (Bridge, Limits)
76. Step 4 starts the container before step 5 copies `ward-agent`, but `init` must exist before first start. After an upgrade, the running `init` stays old while the bridge is new. Accepted, or restart? How is a running binary overwritten? (steps 4 and 5)
77. Docker does not kill exec'd processes when the client disconnects. If ward is killed, Claude keeps running and counts as a session, so the container never idles. Contradicts "works even if ward is killed". What ends orphaned sessions? (Container lifecycle, Terminal relay)
78. On Linux, a resize to the same size does not send SIGWINCH, so the redraw resize does nothing. What forces the redraw? (Terminal relay)
79. Is testing Claude's Linux clipboard-image path a precondition for implementation or an implementer task? Which variables (`DISPLAY`, `WAYLAND_DISPLAY`) must be set? Same question for URL opening (`xdg-open` vs `BROWSER`). (Limits)
