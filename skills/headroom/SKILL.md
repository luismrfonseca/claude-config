---
name: headroom
description: Context compression layer — 60-95% fewer tokens for JSON, 15-20% fewer for coding agents
---

# Headroom: Context Compression Layer

Headroom compresses everything your AI agent reads — tool outputs, logs, RAG chunks, files, conversation history — before it reaches Claude. Same answers, fewer tokens.

## How it works

```
Your agent (Claude Code, Cursor, Codex, your app)
     ↓ prompts · tool outputs · logs · RAG results · files
     ↓
  ┌─────────────────────────────────────────┐
  │ Headroom (runs locally — your data stays)│
  │ ┌─────────────────────────────────────┐ │
  │ │ ContentRouter → SmartCrusher (JSON) │ │
  │ │               → CodeCompressor (AST)│ │
  │ │               → Kompress-v2-base   │ │
  │ │ CCR (reversible compression)       │ │
  │ │ Cross-agent memory                 │ │
  │ └─────────────────────────────────────┘ │
  └─────────────────────────────────────────┘
     ↓ compressed prompt
     ↓
  LLM provider (Anthropic, OpenAI, Bedrock, …)
```

## Features

- **Content-aware compression**: Detects JSON, code, or prose and selects the right compressor
- **SmartCrusher** — JSON compression (removes nesting, redundancy)
- **CodeCompressor** — Code/AST compression (removes boilerplate, preserves signatures)
- **Kompress-v2-base** — General text compression (HuggingFace model, local)
- **CCR (Reversible)** — Originals cached locally; LLM calls `headroom_retrieve` if needed
- **Cross-agent memory** — Share compression state across Claude, Codex, Gemini, Grok
- **Output token reduction** — Trims what the model writes back (drops ceremony, skips deep thinking on routine steps)

## Savings

Real-world measurements on agent workloads:

| Workload | Before | After | Savings |
|----------|-------:|------:|--------:|
| Code search (100 results) | 17,765 | 1,408 | **92%** |
| SRE incident debugging | 65,694 | 5,118 | **92%** |
| GitHub issue triage | 54,174 | 14,761 | **73%** |
| Codebase exploration | 78,502 | 41,254 | **47%** |

On coding agents (agentic workload): **15-20% fewer tokens**, **20% cost reduction**, **accuracy preserved**.

## Setup (already configured)

- ✓ Headroom CLI installed
- ✓ Proxy running on `http://127.0.0.1:8787`
- ✓ MCP retrieve tool registered (restart Claude Code if needed)
- ✓ Config at `~/.claude/headroom.config.toml`

**Serena MCP** (optional semantic code navigation): requires `uv` or `uvx`. Install with:
```bash
pip install uv
headroom wrap claude --serena
```

## Commands

```bash
# Health check — verify routing is working
headroom doctor

# Performance dashboard (proxy must be running)
headroom dashboard

# Per-model compression stats
headroom perf

# Live savings dashboard
headroom perf --live

# Export savings data
headroom export --format json

# Compare models before/after
headroom compare

# Find optimizations
headroom optimize

# Audit token usage
headroom audit

# Learn from failed sessions — writes corrections to CLAUDE.local.md
headroom learn

# MCP server (expose compression data to any MCP client)
headroom mcp
```

## Gotchas

**Claude Code 2.1.221+ limitations:**
- Remote Control (`/rc`) disabled when custom ANTHROPIC_BASE_URL is set
- On-demand tool loading kept on (Headroom doesn't disable it)
- 1M context window disabled — restore with `headroom wrap claude --1m`

**Anonymous telemetry**: compression stats only (never code, paths). Opt-out: `HEADROOM_BEACON=off`

## Unwrap

To remove Headroom and restore direct Anthropic API:
```bash
headroom unwrap claude
```

## Integration with other tools

- Works with **codeburn** for token cost tracking (both read the same session files)
- Complements **ponytail** (less code → fewer tokens to compress)
- Works with **graphify** (large graph outputs get compressed by Headroom)
