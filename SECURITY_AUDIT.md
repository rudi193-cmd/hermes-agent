---
b17: HERM1
title: Security Audit — hermes-agent
date: 2026-05-06
auditor: Hanuman (Claude Code, Sonnet 4.6)
status: open
---

# Security Audit — hermes-agent

Part of Level 2 full-fleet security audit. hermes-agent — feature-rich AI agent CLI with gateway, memory, tool, and model-switching subsystems.

## Rubric Results

| # | Check | Status | Notes |
|---|---|---|---|
| R1 | SQL injection | N/A | No direct SQL queries found |
| R2 | Shell injection | ⚠️ WARN | See P2: shell=True in two places, both fed from local config, not external input |
| R3 | Path traversal | ✅ PASS | No user-controlled path operations identified in critical paths |
| R4 | Hardcoded credentials | ✅ PASS | Sentinel values only ("no-key-required", "aws-sdk", "not-used") — not real secrets |
| R5 | CORS wildcard | N/A | CLI/agent, no web server |
| R6 | XSS | N/A | No HTML rendering |
| R7 | Unsigned code execution | ✅ PASS | No eval(); no exec(); no dynamic imports with user-controlled module names |
| R8 | Missing auth on APIs | N/A | No webhook/API receiver |
| R9 | Bare except swallowing errors | ⚠️ WARN | Multiple `except Exception: pass` in gateway and CLI subsystems |
| R10 | Predictable temp paths | ✅ PASS | No temp file creation identified |
| R11 | Race conditions | ✅ PASS | No shared mutable state across async boundaries identified |
| R12 | safe_integration.py status() | ❌ MISSING | No safe_integration.py present |
| R13 | Entry point importable | ✅ PASS | pyproject.toml entry points defined; imports clean |
| R14 | requirements.txt pinned | ⚠️ WARN | requirements.txt exists but most deps unpinned (no `==` versions) |
| R15 | No hardcoded dev paths | ✅ PASS | Paths use env vars or dynamic resolution |

## Findings

### P2: HA-SHL-01 — shell=True in quick_commands and memory_setup

**Severity:** P2
**Status:** Open
**Files:** `cli.py:5739`, `hermes_cli/memory_setup.py:137`

```python
# cli.py
result = subprocess.run(exec_cmd, shell=True, capture_output=True, text=True, timeout=30)

# memory_setup.py
subprocess.run(check_cmd, shell=True, capture_output=True, timeout=5)
```

Both commands originate from the user's local config files (quick_commands config and plugin dependency definitions), not from external input. Blast radius is the user's own shell. However, if config files are written by a compromised plugin or shared config, shell injection is trivially achievable.

**Recommended fix:** For quick_commands: validate that `exec_cmd` is a string and consider using `shlex.split(exec_cmd)` with `shell=False` where possible. For memory_setup: same — list args preferred.

---

### P2: HA-DEP-01 — requirements.txt unpinned

**Severity:** P2
**Status:** Open

`requirements.txt` exists but most dependencies have no version pins (e.g., `openai`, `rich`, `requests`). A `pip install -r requirements.txt` can pull breaking changes or unpatched security releases. `pyproject.toml` is canonical per the file's own comment — ensure it also pins or uses lock files (`uv.lock` present).

---

### P2: HA-SAP-01 — No safe_integration.py

**Severity:** P2
**Status:** Open

No `safe_integration.py` present. hermes-agent executes tools and model calls — SAFE integration would provide authorization gates for sensitive operations.

---

## Strengths

- **No hardcoded secrets.** All API keys from env vars; sentinel strings ("no-key-required") are non-functional placeholders.
- **Dynamic imports are safe.** `importlib.import_module` in `run_agent.py`/`batch_runner.py` uses hardcoded module names from internal registries, not user input.
- **uv.lock present.** Reproducible installs available via uv even though requirements.txt is unpinned.
- **No eval() or exec().** No dynamic code execution from user input.
