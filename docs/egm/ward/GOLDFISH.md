feature: ward
design_doc_path: docs/designs/ward.md
date_opened: 2026-10-14
human_review_gate: pending
rounds: []

## Round 1

- round: 1
- date: 2026-10-14
- scope: Problem, Technical Plan, Alternatives. Detailed Implementation not yet written.
- agent_type: A generalist, B reviewer, C architect
- prompts: the standard Goldfish preamble, verbatim from the /goldfish skill, followed by the skill's Goldfish A, B, and C task prompts verbatim, with `<PATH>` = `/Users/rafe/other/ward/docs/designs/ward.md`.
- reports: `goldfish-A-comprehension.md`, `goldfish-B-critic.md`, `goldfish-C-readiness.md`
- human_review_gate: pending
- readiness: not-yet

### Orchestrator synthesis

- Comprehension: passed. A's account of goal, threat model, and mechanism matches the design.
- Design-level problems needing user decisions: G1 read-only sub-mounts inside the writable project are bypassable (B F1-F6, F10, C Q71-72). G2 nothing protects the project folder itself from deletion (B F59). G3 Linux root and setuid files (B F7, F36, C Q74). G4 network reach to host and LAN services (B F11, C Q75). G5 terminal escape sequences (B F12). G6 orphaned exec sessions and per-session identity (B F13, F32, C Q53-54, Q77). G7 platform-specific build output in the shared folder (B F55).
- Correctness fixes with no real decision: listed in the round 1 chat summary and to be applied after user approval.
- C's build, packaging, naming, schema, CLI parsing, and acceptance questions are Section 4 material.
