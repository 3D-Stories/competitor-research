# Competitor Research Plugin

Comprehensive competitor research pipeline for Claude Code. Consolidates scattered NotebookLM sources, enriches with web research, and produces structured market briefs with automated quality review.

## What It Does

Given a competitor name, this plugin:

1. Pulls all existing NotebookLM sources about that competitor
2. Searches the web for additional data via Google AI Mode
3. Analyzes and cross-references all sources for accuracy
4. Produces a structured 10-section market brief (individual files + combined)
5. Cleans up scattered sources from NotebookLM
6. Uploads the consolidated brief back to NotebookLM
7. Runs a critique/quality review on the final output

## Prerequisites

### 1. NotebookLM CLI (`notebooklm-py`)

The plugin uses [notebooklm-py](https://github.com/teng-lin/notebooklm-py) for programmatic
access to Google NotebookLM.

**Install:**
```bash
pip install notebooklm-py
```

**Or from GitHub (latest release tag):**
```bash
LATEST_TAG=$(curl -s https://api.github.com/repos/teng-lin/notebooklm-py/releases/latest | grep '"tag_name"' | cut -d'"' -f4)
pip install "git+https://github.com/teng-lin/notebooklm-py@${LATEST_TAG}"
```

**Install the Claude Code skill:**
```bash
notebooklm skill install
```

**Authenticate (required before first use):**
```bash
notebooklm login
```
This opens a browser window for Google OAuth. Sign in with the Google account that has access
to your NotebookLM notebooks. The auth token is stored at `~/.notebooklm/storage_state.json`.

**Verify auth works:**
```bash
notebooklm status
notebooklm list
```

**Auth troubleshooting:**
- If commands fail with auth errors: `notebooklm login` (re-authenticate)
- Check auth details: `notebooklm auth check --test`
- For CI/headless environments: set `NOTEBOOKLM_AUTH_JSON` env var with contents of `storage_state.json`

**Re-authentication:** Google OAuth tokens expire periodically. If the skill fails with auth
errors during a run, it will prompt you to re-run `notebooklm login`.

### 2. Google AI Mode Skill

Required for web research. Must be installed as a standalone Claude Code skill.

**Check if installed:**
```bash
test -f ~/.claude/skills/google-ai-mode/SKILL.md && echo "Installed" || echo "Not installed"
```

**If not installed:** Follow the google-ai-mode skill's installation instructions. The skill
uses a headless browser to query Google's AI Search mode — no API key required, but it may
occasionally encounter CAPTCHAs that require `--show-browser` for manual solving.

### 3. Critique Tool (optional but recommended)

The plugin runs a quality review after generating each brief. It uses the first available tool:

1. `/bmad-party-mode` — multi-perspective adversarial critique (best)
2. `/reflexion:critique` — multi-judge review with consensus (good)
3. Self-critique — inline analysis (always available as fallback)

No setup required — the plugin auto-detects which tools are available.

## Installation

```bash
claude plugin install /path/to/competitor-research
```

Or if published to a marketplace:
```bash
claude plugin install competitor-research
```

## Usage

```
/competitor-research Finch
```

The skill will:
- Ask for disambiguation if needed ("Finch self-care app or Finch payroll API?")
- Suggest an output directory (default: `{project}/competitors/{name}/`)
- Walk through each step with checkpoints for your review

### Multiple Competitors

```
/competitor-research
```
Then mention you want to run for all competitors. The skill will suggest processing order
(heaviest source count first) and run them one at a time.

## Output Structure

```
competitors/finch/
├── 00-index.md                    # Table of contents
├── 01-company-overview.md         # What they do, founded, team
├── 02-funding-and-financials.md   # Rounds, valuation, revenue
├── 03-product-and-features.md     # Feature set, platforms, UX
├── 04-pricing-and-monetization.md # Tiers, freemium, ARPU
├── 05-user-base-and-traction.md   # Downloads, DAU, ratings
├── 06-target-market-and-positioning.md  # Audience, segment, messaging
├── 07-user-sentiment-and-reviews.md     # Love/hate, Reddit, churn
├── 08-strengths-and-weaknesses.md       # SWOT relative to your product
├── 09-threat-assessment.md              # Direct/indirect, moats, scenarios
├── 10-sources-and-data-quality.md       # Citations, confidence ratings
├── finch-full-brief.md            # Combined file (uploaded to NotebookLM)
└── raw/                           # Audit trail
    ├── notebooklm/                # Original source fulltexts
    └── web/                       # Google AI Mode research results
```

## Brief Sections

| # | Section | Serves |
|---|---------|--------|
| 1 | Company Overview | Pitch + Product |
| 2 | Funding & Financials | Pitch |
| 3 | Product & Features | Product |
| 4 | Pricing & Monetization | Pitch + Product |
| 5 | User Base & Traction | Pitch |
| 6 | Target Market & Positioning | Pitch + Product |
| 7 | User Sentiment & Reviews | Product |
| 8 | Strengths & Weaknesses | Pitch + Product |
| 9 | Threat Assessment | Pitch |
| 10 | Sources & Data Quality | Pitch + Product |

## Version History

- **0.1.0** — Initial release. 9-step pipeline with 10 sections, critique cascade, configurable output directory.
