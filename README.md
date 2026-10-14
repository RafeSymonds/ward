# ward

Run Claude Code with permission prompts off, inside a container that can only damage the project you are working in.

**Status: design phase. Nothing is built yet.** The design is in [docs/designs/ward.md](docs/designs/ward.md).

## What it will do

You type `ward` in a project folder where you would have typed `claude`. ward starts a container for that folder and runs Claude inside it, connected to your terminal.

- The project folder is writable. Your home folder is not visible to the agent.
- Your Claude settings, skills, agents, and plugins are copied in each run. The agent cannot change your host config.
- Sessions started in ward can be resumed outside ward, and the other way around.
- The agent has `sudo` inside its container, and what it installs persists for that project.
- Dev servers the agent starts are forwarded to `127.0.0.1` on your machine and shown in Claude's status line.
- Links Claude opens land in your browser. Image paste works.
- Each project gets one container, which stops itself when no session is using it.
- CPU, memory, and process limits are configurable.

## Requirements

macOS or Linux, Docker, and a Claude subscription for `claude setup-token`.

## Planned commands

| Command | Does |
|---|---|
| `ward [-- args]` | Run Claude in the current folder. |
| `ward login` | Store a token from `claude setup-token`. |
| `ward shell` | Open a shell in the project's container. |
| `ward ls` | List ward containers. |
| `ward stop` | Stop the project's container. |
| `ward reset` | Rebuild the container, keeping its home folder. |
| `ward rm` | Delete the container and its home folder. |
| `ward prune --older-than <age>` | Delete containers not used within that age. |
| `ward build` | Rebuild the base image. |

## What it does not protect against

ward prevents damage to your machine. It does not prevent the agent from reading the project folder and sending it over the network, and it cannot protect you from code the agent writes that you later run yourself. See the Limits section of the design doc.
