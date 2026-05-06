# Sovereign Digest: Emacs Edition

A local-first, LLM-powered digest generator that aggregates a month of Emacs news into a single Org file — no cloud APIs, no subscriptions, everything runs on your hardware.

## What It Does

Sovereign Digest pulls from **nine sources**, triages each item with a local LLM, deduplicates across sources, and compiles a richly annotated Org-mode digest:

| Source | What's Captured |
|--------|----------------|
| **Git Savannah** | High-confidence commits to GNU Emacs (≥ 4/5 significance) |
| **r/emacs** | Top posts with 4-6 sentence summaries, community quotes, and engagement data |
| **Hacker News** | Emacs stories via the Algolia search API (no auth needed) |
| **Lobste.rs** | Curated posts from the invite-only community's `emacs` tag |
| **Planet Emacslife** | Curated blog posts from the Emacs blogosphere |
| **Sacha Chua's Weekly** | Gap-filling items from the most comprehensive Emacs news aggregation |
| **MELPA** | New packages added to the archive |
| **Tracked Releases** | Version bumps for major packages (Magit, Org, Consult, Vertico, etc.) |
| **emacs-devel** | Significant proposals and discussions from the mailing list |

## Example Output

Each digest entry includes structured metadata as Org properties:

```org
** Feature: Support 24-bit TrueColor on MS-Windows console
:PROPERTIES:
:COMMIT: 2bca4ac0ed7
:DATE: 2026-04-08
:CONFIDENCE: 5/5
:END:
- *Author:* ewantown
- *Note:* Adds significant new functionality for 24-bit TrueColor on MS-Windows console.

** Package: emskin: a nested Wayland compositor in Rust that embeds any app into Emacs windows
:PROPERTIES:
:OP: u/bilikai
:DATE: 2026-04-17
:SCORE: 84 ↑
:COMMENTS: 50
:END:
- *Link:* https://www.reddit.com/r/emacs/comments/1sooz6l/...
- *Why it matters:* emskin offers a novel way to integrate native applications into Emacs.
- *Summary:* emskin is a nested Wayland compositor written in Rust that allows Emacs to
  treat arbitrary graphical applications as buffers within its windows...
- *Community voice:* "Firefox opens as a Wayland client owned by the compositor."
```

## Requirements

- **WSL2 / Linux** — tested on Debian 13
- **Ollama** with a CUDA-capable GPU (RTX 5070 Ti tested)
- **Python 3.12+** via pyenv
- **Emacs** with Org-Babel (for literate execution)

### Models

Two-model strategy for best results:

| Model | Role | Pull Command |
|-------|------|-------------|
| `qwen2.5-coder:14b-instruct-q5_K_M` | Structured JSON triage & classification | `ollama pull qwen2.5-coder:14b-instruct-q5_K_M` |
| `qwen2.5:14b-instruct-q5_K_M` | Nuanced prose summaries | `ollama pull qwen2.5:14b-instruct-q5_K_M` |

Both fit comfortably in 16 GB VRAM at Q5_K_M quantization.

## Quick Start

```bash
# 1. Clone
git clone https://github.com/YOUR_USERNAME/sovereign-digest.git
cd sovereign-digest

# 2. Create Python environment
pyenv virtualenv 3.12.3 digest-env
~/.pyenv/versions/digest-env/bin/pip install ollama requests feedparser python-dotenv

# 3. Pull models
ollama pull qwen2.5-coder:14b-instruct-q5_K_M
ollama pull qwen2.5:14b-instruct-q5_K_M

# 4. (Optional) Reddit OAuth — for higher rate limits
cp .env.example .env
# Edit .env with your Reddit API credentials

# 5. Open in Emacs and run
emacs digest-pipeline.org
```

In Emacs, execute blocks in order with `C-c C-c`:
1. **§2 Configuration** — always run first (initializes session state)
2. **§3–§10** — source collection (can be run individually or in sequence)
3. **§11 Compilation** — deduplicates and writes the digest

## Pipeline Architecture

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Git        │  │  Reddit     │  │ Hacker News │  │  Lobste.rs  │
│  Savannah   │  │  (OAuth or  │  │  (Algolia)  │  │  (RSS tag)  │
│             │  │   public)   │  │             │  │             │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │                │
       │  MODEL_TRIAGE  │          MODEL_SUMMARY          │
       │  (coder 14B)   │          (general 14B)          │
       │                │                │                │
       ▼                ▼                ▼                ▼
┌────────────────────────────────────────────────────────────────┐
│                  seen_ids.json (TTL: 90d)                     │
└────────────────────────────────────────────────────────────────┘
       │                │                │                │
┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐
│   Planet    │  │  Sacha Chua │  │   MELPA +   │  │ emacs-devel │
│  Emacslife  │  │  Weekly     │  │  Releases   │  │ (Atom feed) │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │                │
       ▼                ▼                ▼                ▼
┌────────────────────────────────────────────────────────────────┐
│            Cross-Source URL Deduplication                      │
│   (Reddit > HN > Lobsters > Planet > Sacha priority)          │
└───────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
                   digest-YYYY-MM-DD.org
                   latest-digest.org
```

## Configuration

All constants live in **§2** of the pipeline:

| Constant | Default | Description |
|----------|---------|-------------|
| `MODEL_TRIAGE` | `qwen2.5-coder:14b-instruct-q5_K_M` | Model for JSON classification |
| `MODEL_SUMMARY` | `qwen2.5:14b-instruct-q5_K_M` | Model for prose summaries |
| `DAYS_AGO` | `30` | Lookback window |
| `CACHE_TTL_DAYS` | `90` | Auto-prune cache entries older than this |
| `TRACKED_PACKAGES` | 11 packages | GitHub release feeds to monitor |

### Reddit API (Optional)

The pipeline works out of the box using Reddit's public JSON endpoints (~10 req/min). For higher rate limits, create a [Reddit script app](https://www.reddit.com/prefs/apps/) and add credentials to `.env`:

```env
REDDIT_CLIENT_ID=your_client_id
REDDIT_CLIENT_SECRET=your_client_secret
```

## Features

- **Local-first** — no cloud LLM APIs; everything runs on Ollama
- **Literate programming** — the pipeline *is* the documentation
- **Dual-model triage** — coder model for structured classification, general model for nuanced summaries
- **Cross-source dedup** — items appearing in multiple feeds are merged, keeping the richest version
- **Crash recovery** — cache checkpoints every N items; resume where you left off
- **Response validation** — LLM output is schema-checked before use
- **Structured logging** — `digest.log` with timestamped DEBUG/INFO/WARNING/ERROR levels
- **Versioned output** — dated digest files for month-over-month comparison
- **Cache TTL** — automatic 90-day expiry prevents unbounded growth

## Bonus: Fortune File

Section 12 extracts tips from the fortnightly r/emacs tips threads into a `fortune`-compatible file:

```bash
strfile tips_and_tricks.fortune
fortune tips_and_tricks.fortune
```

## License

MIT
