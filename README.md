# REA on Apple Silicon ARM64 — Field Test Notes

Independent field notes on [REA — Reverse Engineer Anything](https://github.com/morluto/rea) (`morluto/rea`, MIT), tested in October 2026. Not affiliated with the REA project. This repo documents a reproducible setup, what actually worked on ARM64, and the safety limits that matter before you point it at anything untrusted.

> Tested on trusted binaries only (e.g. `/bin/ls`). No proprietary code, no malware, no third-party firmware is analysed or published here.

## Verdict in one paragraph

REA is real, and it works — as an MCP/CLI wrapper that drives existing reverse-engineering engines, not as a decompiler of its own. Its October 2026 star explosion is real too (see below), but the hype hides three facts that change how you should use it: it is **not a sandbox**, it does **not patch binaries**, and it is **not fully local** with a cloud-hosted agent model. Pin a version, run dubious targets in a VM, and treat its output as decompiler evidence, not recovered source.

## What was tested

| Item | Mac (native, trusted binaries) | Linux VM (sandbox, dubious targets) |
|---|---|---|
| Host | Mac mini, Apple Silicon, 24 GB | UTM VM, Ubuntu 24.04.5 ARM64, 8 GB RAM, 64 GB disk, native virtualisation, no shared folder |
| Node / JDK | Node 24.21.0, npm 11.19.0, OpenJDK 21.0.12.1 | Node 24.21.0, npm 11.19.0, Temurin JDK 21.0.12.1+1 |
| Ghidra | 12.1.4 PUBLIC | 12.1.4 PUBLIC |
| REA | `rea-agents@6.4.0` pinned (`npx -y rea-agents@6.4.0 mcp`) | `rea-agents@6.5.0` pinned (npm global) |
| Agent skill | Claude Code skill v37, 170 tools; `~/.claude.json` "rea" MCP entry sets `GHIDRA_INSTALL_DIR` and `JAVA_HOME` | Same toolchain, SSH env exports (see pitfall below) |

Smoke test, both sides, passed end to end:

```
rea analyze /bin/ls
# -> evidence_id generated, ELF ARM64, decompiler active
# /bin/ls: 523 procedures, 502 strings, 30 segments, Ghidra 12.1.4
# Ghidra headless: Import / Analysis / Save succeeded
```

## The ARM64 gotcha (the part most tutorials skip)

Ghidra 12.1.4 ships **no `mac_arm_64/decompile` binary**, and the same is true for Linux ARM64. The native decompiler has to be **compiled from source** on both hosts before REA's Ghidra lane works. Budget for that step; it is the single biggest setup friction on Apple Silicon / ARM64 Linux, and it recurs on every Ghidra upgrade (keep a user-space toolchain handy — here `g++ 13`, ~200 MB, for rebuilds).

## Usage rule adopted

- **Host (Mac): trusted binaries only.**
- **VM: everything dubious.** No shared folder, by design. If the VM gets dirty, reinstall from the ISO (~5 min) — no snapshot dependency.
- **Static analysis only** for the tests documented here. Dynamic capture executes the target with your user permissions (see below).

## Pitfall: SSH non-interactive environment

On Ubuntu, `~/.bashrc` exits early for non-interactive SSH, so `JAVA_HOME`, `GHIDRA_INSTALL_DIR` and the Node path are **not** loaded, and `rea analyze` fails with `GHIDRA_INSTALL_DIR is not set`. Prefix SSH commands explicitly:

```bash
export PATH=$HOME/.nvm/versions/node/v24.21.0/bin:$PATH \
  JAVA_HOME=$HOME/jdk \
  GHIDRA_INSTALL_DIR=$HOME/Applications/ghidra_12.1.4_PUBLIC
```

## What REA actually does (and does not do)

- **Does:** MCP server + CLI (`rea-agents` on npm) orchestrating Ghidra / Hopper / IDA Pro (plus JADX for Android, Binwalk/Unblob for firmware extraction only). Decompiles to pseudocode + assembly, lists/searches strings, symbols and functions, follows calls and xrefs, and returns an **Evidence record** (result + `evidence_id` + stated limitations/unknowns).
- **Does not patch binaries.** Annotation/rename in the Ghidra lane is limited to the **ephemeral session database**; executable bytes are unchanged and metadata is discarded on close. The IDA lane is read-only.
- **Is not a sandbox.** REA's own `SECURITY.md` says so plainly: bridge sessions are not a sandbox, parsing an untrusted binary runs with your user permissions, and runtime capture **executes the target**. Hence the VM rule above.
- **Is not fully local by default.** REA does not upload your binary, but tool results (decompiled code, strings, findings) go to your agent, and a cloud-hosted agent model sends them to its provider. Fully local requires a local-model agent setup, at a quality cost.

## The 100k-stars claim, checked (12 October 2026)

Verified live on GitHub on 12 Oct 2026 ~16:14 UTC: **~142,000 stars, 32,600 forks**.

- Repo created **14 April 2026** (initially `betterBinaryMCP`, a Hopper MCP server).
- ~647 stars at end of September 2026, then a viral spike **4–11 October**: crossed 100,000 stars on 11 October.
- No evidence found of purchased stars; X virality plausibly explains the curve (notably a 2.8M-view post on 8 October). Treat stars as a virality metric, not an audit.
- Velocity warning: v4.0.0 (5 Oct) → v4.1.0 (6 Oct) → v5.0.0 (7 Oct) → v6.0.0 (8 Oct) → v6.4.0 (11 Oct). **Pin a version**; `@latest` today is not tomorrow's code. 153 open issues / 134 open PRs at the 12 Oct check.

## Alternatives in the same space

- **GhidraMCP** (LaurieWired, Apache-2.0): Ghidra only, mature, persistent rename/annotation inside your Ghidra project.
- **ida-pro-mcp** (mrexodia, MIT): IDA Pro only, deeper in IDA; REA's IDA lane is built on it, restricted to read-only.
- **ReVa** (cyberkaida): Ghidra extension/MCP built around many small tools to limit context rot on long tasks.
- **Binary Ninja MCP** (fosdickio, GPL-3.0) and **radare2-mcp** (radareorg, official): engine-specific, REA has neither as a provider (roadmap only).

REA's differentiator is breadth and packaging: many engines and formats behind one installer, one tool vocabulary, and an evidence/unknowns output format.

## Scope and disclaimer

These notes cover setup and smoke tests on trusted binaries only. They are not a benchmark of REA on large real-world binaries (none was found in October 2026), and they do not cover automotive or proprietary firmware analysis. Use REA only on software you own or are authorised to examine, and check the target's licence/EULA and your jurisdiction's anti-circumvention rules.

## Sources

- https://github.com/morluto/rea — repo, README, releases (v6.4.0, 11 Oct 2026), SECURITY.md
- https://rea.tools/ — project site
- Field setup and smoke-test results: October 2026, as tabulated above
