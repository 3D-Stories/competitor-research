# Competitor Research Pipeline

Comprehensive competitor research pipeline for Claude Code. Consolidates scattered
NotebookLM sources, enriches with live web research via Google AI Mode MCP, cross-references
for accuracy, and produces structured 10-section market briefs with automated quality review.

Output is dual-purpose: briefs that serve **investor pitch materials** (market sizing, funding,
threat assessment) AND **product strategy decisions** (feature gaps, user sentiment, positioning).

## Prerequisites & Setup

Run `/competitor-research setup` once to configure all dependencies. The setup wizard walks
through 7 steps and saves configuration to `~/.config/competitor-research/config.json`.

If you skip setup and run `/competitor-research <name>` directly, the skill auto-detects the
missing config and runs setup automatically before proceeding.

### Setup Steps

| Step | What It Does |
|------|-------------|
| **1. Environment Detection** | Checks for a graphical display (`$DISPLAY` + `xdpyinfo`). If headless, flags that VNC will be needed for browser auth flows. |
| **2. Virtual Display (headless only)** | Invokes `/vnc-service:setup` to install Xvfb + x11vnc. Falls back to manual instructions if the vnc-service skill is unavailable. Runs `/vnc-service:run` and waits for user to confirm VNC connection. |
| **3. NotebookLM CLI** | Installs `notebooklm-py` from PyPI (`pip install notebooklm-py`), registers the Claude Code skill (`notebooklm skill install`), then reloads plugins (`/reload-plugins`). Handles OAuth login (via VNC if headless, direct browser if graphical). |
| **4. Google AI Mode MCP** | Installs the MCP server: `/mcp add google-ai-search npx google-ai-mode-mcp@latest`. Requires a **session restart** after install (MCP servers only load at session start). Tests with a sample search and handles CAPTCHA via VNC if needed. |
| **5. Notebook Selection** | Lists existing NotebookLM notebooks or creates a new one. User picks which notebook to use for competitor research. |
| **6. Critique Tool Detection** | Searches for critique tools in priority order: (1) BMAD Party Mode (searches common paths for `agent-manifest.csv`), (2) reflexion:critique skill, (3) offers to install reflexion via `/plugin marketplace add NeoLabHQ/context-engineering-kit` then `/plugin install reflexion@NeoLabHQ/context-engineering-kit` then `/reload-plugins`, (4) self-critique as last resort. |
| **7. Save Configuration** | Writes all settings to `~/.config/competitor-research/config.json` (headless flag, notebook ID, critique tool, BMAD path, MCP status, VNC status). Reports a summary of what was configured. |

## Installation

Install the skill into Claude Code, then run setup:

```bash
claude plugin install /path/to/competitor-research
```

After installing, run `/competitor-research setup` in your first session to configure
dependencies (NotebookLM CLI, Google AI Mode MCP, critique tools).

## Usage

### First-Time Setup

```
/competitor-research setup
```

### Research a Competitor

```
/competitor-research <name>
```

Example: `/competitor-research Greenlight`

### Inputs

| Input | Required | Description | Example |
|-------|----------|-------------|---------|
| **Competitor name** | Yes | The company or product to research | `Greenlight`, `Finch`, `BusyKid` |
| **Domain keywords** | No | Additional search terms for the competitor's market. Inferred from name if not provided. | `kids fintech debit card banking` |
| **Product clarifier** | No | Disambiguation when a name is ambiguous. The skill asks if unclear. | `Finch self-care app` vs `Finch payroll API` |
| **Notebook ID** | No | Defaults to the notebook configured during setup | UUID from NotebookLM |
| **Project path** | No | Defaults to active rawgentic project path | `/home/user/my-project` |

### Multiple Competitors

Run one at a time. The skill suggests processing order (heaviest source count first to free
NotebookLM slots early). Each competitor reuses the same config from setup.

## Pipeline Workflow

The pipeline executes 9 steps in order, each with a clear checkpoint before proceeding.

| Step | Name | What Happens |
|------|------|-------------|
| **1** | Prerequisites Check | Validates config exists. If missing, runs setup automatically. Quick-checks NotebookLM auth and MCP availability. |
| **2** | Disambiguation & Directory Setup | Confirms competitor name (asks if ambiguous). Collects domain keywords. Creates output directory structure. Sets NotebookLM notebook context. |
| **3** | Pull Existing NotebookLM Sources | Lists all sources, filters by competitor name + domain keywords. User confirms include/exclude. Pulls fulltext for each source. Performs quick section gap analysis to inform targeted web research. |
| **4** | Web Research via Google AI Mode MCP | Runs 5 required searches (overview, traction, pricing, sentiment, market position) plus gap-targeted searches based on Step 3 analysis. Uses `mcp__google-ai-search__search_ai` MCP tool. Handles CAPTCHA via VNC. Saves results to `raw/web/`. |
| **5** | Analyze & Verify | Reads all raw sources. Flags wrong-company data, cross-references facts, checks freshness (>12 months), identifies gaps, checks for AI search contamination from same-name companies. Reports conflicts to user. |
| **6** | Write Section Files | Writes 10 section files (01-10), the index (00), and the combined full brief. Follows section templates with confidence headers and inline citations. |
| **7** | Critique & Quality Review | **Runs BEFORE deleting NotebookLM sources** so originals remain if fatal flaws are found. Uses configured critique tool (BMAD / reflexion / self-critique). Offers to revise affected sections. Regenerates combined file if revisions are made. |
| **8** | Upload Consolidated Brief | **Uploads BEFORE deleting old sources** to eliminate data loss window. Uploads combined brief to NotebookLM. Verifies it appears in source list. If upload fails, informs user of local path -- does NOT proceed to deletion. |
| **9** | Clean Up NotebookLM Sources | Requires explicit user confirmation. Deletes old sources (unique + duplicates + wrong-company). Reports count freed. Handles mid-batch failures gracefully (logs failed IDs, continues, retries auth if expired). |

## Output Structure

```
{project_path}/competitors/{competitor_name}/
├── 00-index.md                              # Table of contents with reading paths
├── 01-company-overview.md                   # Section 1
├── 02-funding-and-financials.md             # Section 2
├── 03-product-and-features.md               # Section 3
├── 04-pricing-and-monetization.md           # Section 4
├── 05-user-base-and-traction.md             # Section 5
├── 06-target-market-and-positioning.md      # Section 6
├── 07-user-sentiment-and-reviews.md         # Section 7
├── 08-strengths-and-weaknesses.md           # Section 8
├── 09-threat-assessment.md                  # Section 9
├── 10-sources-and-data-quality.md           # Section 10
├── {competitor_name}-full-brief.md          # Combined file (all sections, for NotebookLM)
└── raw/                                     # Raw source texts (for audit trail)
    ├── notebooklm/                          # Fulltexts pulled from NotebookLM
    │   ├── {source_id_short}.md
    │   └── ...
    └── web/                                 # Google AI Mode MCP research results
        ├── search-{topic}.md
        └── ...
```

## Brief Sections

Each section file follows this template format:

```markdown
# {Section Title}
> Competitor: {name} | Last Updated: {date} | Confidence: {high/medium/low}

{Section content with inline citations [S#-N]}

---
*Sources: See 10-sources-and-data-quality.md for full citation list*
```

| # | Section | Serves | Content |
|---|---------|--------|---------|
| 1 | Company Overview | Pitch + Product | What they do, founding date, HQ, team size, leadership, mission, positioning summary |
| 2 | Funding & Financials | Pitch | Funding rounds, valuations, estimated revenue, burn rate signals. Flags same-name company confusion. |
| 3 | Product & Features | Product | Core features by category, platforms, UX patterns, recent launches, tech signals, integrations |
| 4 | Pricing & Monetization | Pitch + Product | Pricing tiers (exact amounts), free vs paid breakdown, subscription model, ARPU signals |
| 5 | User Base & Traction | Pitch | Downloads, DAU/MAU, app store ratings, growth trajectory, engagement metrics. Labels single-source estimates. |
| 6 | Target Market & Positioning | Pitch + Product | Primary/secondary audiences, demographics, market segment, brand voice, TAM/market size |
| 7 | User Sentiment & Reviews | Product | Review themes, top loves/complaints, feature requests, churn signals. Includes 3-5 verbatim quotes. |
| 8 | Strengths & Weaknesses | Pitch + Product | SWOT-style analysis. Honest assessment. Notes that user product comparisons reflect vision, not shipped product. |
| 9 | Threat Assessment | Pitch | Direct/indirect/potential classification, defensive moats, threat scenarios, strategic response |
| 10 | Sources & Data Quality | Pitch + Product | Full citation list, source quality tiers, wrong-company exclusions, data conflicts, per-section confidence |

## Troubleshooting

| Error | Response |
|-------|----------|
| NotebookLM auth expired | Invokes `/vnc-service:run`, then runs `DISPLAY=:99 notebooklm login` |
| Google AI Mode CAPTCHA | Invokes `/vnc-service:run`, retries search with `headless: false` for user to solve in VNC |
| MCP not available | Inform user to restart session -- MCP servers only initialize at session start |
| Source fulltext fails | Logs source ID, skips it, notes in Section 10 as "content unavailable" |
| Ambiguous competitor name | Always asks the user to clarify, never guesses |
| Thin data | Writes honestly with what is available. Flags: "Section N is thin -- only X words" |
| Chrome singleton lock | Clears `/path/to/chrome_profile/Singleton*` and retries |
| Config missing | Auto-runs `/competitor-research setup` before proceeding |

## VNC / Headless Environments

This skill relies on browser-based authentication (NotebookLM OAuth, Google AI Mode CAPTCHA
solving). On headless servers, it uses the `/vnc-service` plugin to provide a virtual display
(Xvfb on `:99`) and VNC server for remote browser interaction.

- Setup installs VNC automatically via `/vnc-service:setup`
- During research runs, `/vnc-service:run` is invoked whenever browser interaction is needed
- VNC binds to localhost only -- connect via SSH tunnel

## Web Research Architecture

Web research uses the **Google AI Mode MCP server** (`google-ai-mode-mcp`), not a skill.
The MCP server is installed into the session with:

```
/mcp add google-ai-search npx google-ai-mode-mcp@latest
```

This registers the `mcp__google-ai-search__search_ai` tool, which queries Google's AI Search
mode via a headless browser. The MCP approach means the tool is available directly in Claude's
tool list and can be called programmatically during the research pipeline without skill
invocation overhead.

Key behaviors:
- Runs headless by default for speed
- Falls back to visible browser (on VNC display) when CAPTCHA is required
- Uses domain keywords from the disambiguation step to make searches competitor-specific
- Requires a session restart after initial installation (MCP servers load at session start)

## Version History

- **0.2.0** -- Major rewrite. Added setup wizard (7-step first-run configuration), Google AI
  Mode MCP integration (replacing skill-based web research), BMAD/reflexion critique cascade,
  domain keywords for targeted searches, section gap analysis, wrong-company contamination
  checks, upload-before-delete safety ordering, and persistent config file.
- **0.1.0** -- Initial release. 9-step pipeline with 10 sections, critique cascade,
  configurable output directory.
