# Agent-World

CLI tools for giving AI agents a better feel of the outside world.

Inspired by Microsoft's `playwright-cli`, which gives agents structured access to browser state via accessibility snapshots — Agent-World extends that philosophy to the rest of the world: social platforms, code forges, news feeds, and anything else an agent might need to navigate.

## Philosophy

Agents work best when they can interact with the world through structured, predictable interfaces. GUIs are for humans. APIs are for machines. CLIs sit in between — human-readable, machine-parseable, and composable via pipes and scripts.

Agent-World provides two kinds of tools:

### Custom Tools

Original CLIs built specifically for agent consumption. These are written for **Node.js** and **Deno**, and each lives in its own directory with a `CLAUDE.md` describing how to navigate the world using that tool.

**Interop format preference:**
1. **YAML** — most natural-language-adjacent, easiest for agents to read and produce
2. **XML** — when structure demands it
3. ~~JSON~~ — custom tools have no excuse to use JSON

### Pre-Existing Tool Guides

Wrappers and guides for existing CLIs that are already useful to agents but lack agent-oriented documentation. Each gets a directory with a `CLAUDE.md` focused on discovery and navigation patterns — not a rehash of `--help`, but a guide to *how an agent should think about using the tool*.

Example: GitHub CLI (`gh`) — the `CLAUDE.md` covers discovery practices, not just command syntax.

## Directory Structure

```
tools/
├── gh/                    ← pre-existing tool guide
│   └── CLAUDE.md          ← discovery-oriented usage guide
└── <tool-name>/           ← custom or pre-existing tool
    ├── CLAUDE.md          ← agent navigation guide
    └── src/               ← source (custom tools only)
```

## Development

This repo follows [VSDD](CLAUDE.md) (Verified Spec-Driven Development).
