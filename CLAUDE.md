# ward

ward runs Claude Code inside a per-project Docker container so it can run with permission prompts off without being able to damage the host.

design_doc_path: docs/designs/

## Status

Design phase. There is no code yet. Do not write code until `docs/designs/ward.md` has passed `/goldfish` and `human_review_gate` in `docs/egm/ward/GOLDFISH.md` is `passed` or `skipped-solo`. Implementation goes through `/egm-implement docs/designs/ward.md`.

## Where things are

- `docs/designs/ward.md` is the source of truth for the design.
- `docs/egm/ward/ELEPHANT.md` is the design decision trace. Read it before resuming design work.
- `docs/egm/ward/GOLDFISH.md` and `goldfish-*.md` are the review rounds.
- `docs/egm/ward/proposal-draft-*.md` are superseded drafts, kept as history. Do not implement from them.

## Rules that protect the host

These come from the design doc. Changing any of them is a design change, not an implementation detail.

1. Nothing ward trusts is read from the project folder. Configuration lives only in `~/.config/ward/`.
2. The bridge accepts a fixed list of requests. Never add one that runs a command on the host.
3. The container is never privileged and never gets the Docker socket.
4. Host `~/.claude` and `~/.claude.json` are never mounted writable. Only `~/.claude/projects/<key>` is bound live, with its `memory` folder read-only on top.
5. Files the host runs later are mounted read-only: `.git/hooks`, `.git/config`, and the project's `.claude/settings.json` and `settings.local.json`.
6. Forwarded ports listen on `127.0.0.1` only.
7. The login token never appears on a command line.

## Scope

v1 is Claude Code only, on a local macOS or Linux host with Docker. Remote sessions, other harnesses, per-session containers, and per-project Dockerfiles are deferred. Before proposing any of them, or any other idea the design rejected, read the Alternatives section of the design doc.
