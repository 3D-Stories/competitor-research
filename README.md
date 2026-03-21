# Competitor Research Pipeline

Comprehensive competitor research pipeline for Claude Code. Consolidates scattered
NotebookLM sources, enriches with live web research via Google AI Mode MCP, cross-references
for accuracy, and produces structured 10-section market briefs with automated quality review.

Output is dual-purpose: briefs that serve **investor pitch materials** (market sizing, funding,
threat assessment) AND **product strategy decisions** (feature gaps, user sentiment, positioning).

## Installation

Remove any stale marketplace entry, add fresh, install, and reload:

```bash
claude plugin marketplace remove competitor-research
claude plugin marketplace add 3D-Stories/competitor-research
claude plugin install competitor-research@competitor-research
```

Then reload plugins in your session:

```
/reload-plugins
```

After installing, run `/competitor-research setup` to configure dependencies.

## Setup (`/competitor-research setup`)

Run once to configure all dependencies. Can be re-run to reconfigure. The setup wizard
walks through 8 steps and saves configuration to `~/.config/competitor-research/config.json`.

If you skip setup and run `/competitor-research <name>` directly, the skill auto-detects
the missing config and runs setup automatically before proceeding.

### Step 0: Node.js & npm

Checks if Node.js v20+ and npm are installed (required for MCP servers and npx commands).
If missing or too old: installs latest LTS via NodeSource (`curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -`)

### Step 1: Environment Detection

Checks for a graphical display (`$DISPLAY` + `xdpyinfo`). If headless, flags that VNC will
be needed for browser auth flows (OAuth, CAPTCHA).

### Step 2: Virtual Display (headless only)

Checks if `/vnc-service:setup` is available. If not, installs the vnc-service plugin:

```bash
claude plugin marketplace add 3D-Stories/vnc-service
claude plugin install vnc-service@vnc-service
```

User must run `/reload-plugins` to activate it. After reload, invokes `/vnc-service:setup`
to install Xvfb + x11vnc, then `/vnc-service:run` to start the display and get connection info.
Waits for user to confirm VNC connection before proceeding.

### Step 3: NotebookLM CLI

Installs `notebooklm-py` via **pipx** (not pip -- modern Ubuntu/Debian use PEP 668 which
blocks direct pip install into system Python):

```bash
pipx install notebooklm-py
pipx inject notebooklm-py playwright
$(pipx environment --value PIPX_LOCAL_VENVS)/notebooklm-py/bin/playwright install chromium
# Install system dependencies for Chromium (libatk, libcairo, libpango, etc.)
# Without this, Chromium crashes with "cannot open shared object file" errors
sudo $(pipx environment --value PIPX_LOCAL_VENVS)/notebooklm-py/bin/playwright install-deps chromium
```

If sudo is unavailable for `install-deps`, the user must install the packages manually.
Run without sudo to see the list of required packages.

Then installs the Claude Code skill and reloads:

```bash
notebooklm skill install
```
```
/reload-plugins
```

Handles OAuth login (via VNC with `DISPLAY=:99` if headless, direct browser if graphical).
The `notebooklm login` command waits for ENTER after OAuth completes, but Claude Code's Bash
tool sends EOF immediately which aborts login. Uses a two-step pattern instead:

**Step A:** Launch login in background so the user can interact with the browser:
```bash
DISPLAY=:99 notebooklm login &
LOGIN_PID=$!
```
User completes OAuth in the browser (via VNC for headless).

**Step B:** After user confirms OAuth is complete, kill the background process and re-run
with immediate ENTER to save auth from the persistent browser profile:
```bash
kill $LOGIN_PID 2>/dev/null
echo "" | DISPLAY=:99 notebooklm login
```

The persistent browser profile (`~/.notebooklm/browser_profile`) retains the Google session,
so the second run opens an already-authenticated browser and immediately saves.

**If login fails** (e.g., missing dependencies, Chromium crash): kill the background process
before retrying to avoid orphaned browser instances:
```bash
kill $LOGIN_PID 2>/dev/null
# Fix the issue, then retry from Step A
```

### Step 4: Google AI Mode MCP

Google AI Mode is an **MCP server** (not a skill). Installed via:

```bash
claude mcp add google-ai-search npx google-ai-mode-mcp@latest
```

This registers the `mcp__google-ai-search__search_ai` tool in the session.

**HARD STOP after install.** MCP servers only load at session start. The user must restart
the session and re-run `/competitor-research setup`. Setup detects that Steps 1-3 are
already complete and resumes from Step 4 verification.

After restart, the skill tests with a sample search. If the test fails with
"Chromium distribution 'chrome' is not found", Chrome/patchright must be installed:

```bash
sudo npx patchright install chrome
```

Note: `apt install google-chrome-stable` does NOT work -- it is not in default Ubuntu repos.
The patchright installer downloads the correct Chromium build. This requires interactive sudo,
so the user runs it in a separate terminal.

If the test search triggers a CAPTCHA, the skill invokes `/vnc-service:run` and retries with
`headless: false` so the user can solve the CAPTCHA in VNC.

### Step 5: Notebook Selection

Lists existing NotebookLM notebooks or creates a new one. User picks which notebook to use
for competitor research.

### Step 6: Critique Tool Detection

Searches for critique tools in priority order:

1. **BMAD Party Mode** (preferred) -- searches common paths for `agent-manifest.csv`.
   If not found, offers to install: `npx bmad-method install --directory . --tools claude-code --yes` (non-interactive)
2. **reflexion:critique** -- checks if available in skills list. If not found, offers to install:
   ```bash
   claude plugin marketplace add NeoLabHQ/context-engineering-kit
   claude plugin install reflexion@NeoLabHQ/context-engineering-kit
   ```
   User runs `/reload-plugins` to activate.
3. **Self-critique** -- always available, no install needed.

### Step 7: Save Configuration

Writes all settings to `~/.config/competitor-research/config.json`. Supports **named profiles**
for users researching competitors across multiple projects or notebooks. Setup creates a default
profile. Users can add more by re-running setup or manually editing the config.

```json
{
  "version": 2,
  "headless": true,
  "critique_tool": "bmad|reflexion|self",
  "bmad_path": "/home/user/bmad/_bmad",
  "mcp_installed": true,
  "vnc_configured": true,
  "default_profile": "chorestory",
  "profiles": {
    "chorestory": {
      "notebook_id": "uuid",
      "notebook_title": "ChoreStory Investor Research",
      "project_path": "/home/user/projects/chorestory_business"
    }
  }
}
```

Global settings (headless, critique_tool, mcp_installed, vnc_configured) are shared across
profiles. Per-profile settings are notebook_id, notebook_title, and project_path.

## Project Context

Step 2 of the pipeline gathers context about the user's own product so that competitor briefs
produce strategic comparisons rather than generic profiles. This grounds Sections 3 (Product &
Features), 6 (Target Market & Positioning), 8 (Strengths & Weaknesses), and 9 (Threat
Assessment) in actual product reality.

**Auto-detection from NotebookLM:** The skill searches the active notebook for project documents
(executive summaries, pitch decks, PRDs, READMEs) and extracts structured context: product name,
audience, JTBD, differentiators, monetization model, and stage.

**Quick briefing (7 questions):** If no project documents are found in NotebookLM, the skill
offers a quick briefing -- 7 questions about the product (name, audience, problem, differentiators,
monetization, stage, category). Answers are used to generate a Project Context Brief that is
saved locally and uploaded to NotebookLM for future runs.

**Upload existing docs:** Users can point to local files (pitch deck, PRD, README) and the skill
extracts context from those instead.

**Skip option:** Proceeding without project context is possible, but Sections 8 and 9 will
contain generic competitive analysis without product-specific SWOT comparisons. The index and
affected section headers are annotated with a "no project context" caveat.

**Cached in config profile:** Once gathered, project context is saved to
`config.profiles[profile].project_context` and loaded automatically on subsequent runs. The skill
offers a refresh option if the cached context is stale.

**Section header template** includes the project name for quick identification:

```
> Competitor: {name} | vs: {project_name} | Last Updated: {date} | Confidence: {level}
```

## Usage

### Research a Competitor

```
/competitor-research <name>
```

Example: `/competitor-research Greenlight`

Example (multiple): `/competitor-research Finch, Greenlight, BusyKid`

When multiple names are provided, runs in parallel mode.

### Inputs

| Input | Required | Description | Example |
|-------|----------|-------------|---------|
| **Competitor name(s)** | Yes | The company or product to research (comma-separated for multiple; commas required for multi-word names like `Hello Fresh, Greenlight`) | `Greenlight`, `Finch, Greenlight, BusyKid` |
| **Domain keywords** | No | Additional search terms for the competitor's market. Inferred from name if not provided. | `kids fintech debit card banking` |
| **Product clarifier** | No | Disambiguation when a name is ambiguous. The skill asks if unclear. | `Finch self-care app` vs `Finch payroll API` |
| **Notebook ID** | No | Defaults to the notebook configured during setup | UUID from NotebookLM |
| **Project path** | No | Defaults to active rawgentic project path | `/home/user/my-project` |

### Parallel Mode

When 2+ competitors are provided, the skill uses a fan-out / fan-in pattern:

**Phase 1: Shared Setup + CAPTCHA Pre-Warming (interactive)**
- Prerequisites check, profile/notebook verification, and CAPTCHA pre-warming run once
- Each competitor is disambiguated with domain keywords
- User confirms all competitors before agents launch

**Phase 2: Parallel Research (concurrent agents)**
- All agents spawn in a single message and run concurrently
- The orchestrator blocks until ALL agents complete (no incremental notification --
  this is how Claude Code's Agent tool works)
- Each agent prompt is fully self-contained: includes competitor name, domain keywords,
  output directory, notebook ID, current year, complete section definitions,
  notebooklm use command, available tools list, and .status.json protocol
- Web searches queue through the shared MCP browser (sequential but fast);
  analysis and writing are truly parallel
- Each agent writes to its own output directory -- no file conflicts
- CAPTCHA in agents: if triggered, the agent writes a `.captcha_blocked` marker file
  and skips remaining searches. The main session backfills after all agents complete.

**Phase 3: Sequential Completion (interactive)**
- Processes ALL competitors after ALL agents complete (not first-done)
- For each competitor, reads `.status.json` and runs Steps 7-9:
  - Critique using configured tool (BMAD/reflexion/self)
  - Upload consolidated brief to NotebookLM
  - Clean up old sources (with user confirmation)
- For 3+ competitors with BMAD critique: offers a batch option -- self-critique all briefs
  first, then BMAD deep-dive on the weakest. Avoids N interactive party mode sessions.

**Error Recovery**
- Reads `.status.json` per competitor to determine which step failed
- Partial failures (e.g., Steps 3-4 done, 5-6 failed): offers to run remaining steps
  from the main session using existing raw files
- Missing `.status.json` (agent timeout): informs user, offers manual completion
- CAPTCHA-blocked agents: resolves CAPTCHA interactively, backfills missing searches

Progress tracking:
```
Competitor Research Progress:
  ● Finch        — Steps 3-6 running (agents active, waiting for all to complete)
  ● Greenlight   — Steps 3-6 running
  ● BusyKid      — Steps 3-6 running
  ---
  All agents complete. Processing Steps 7-9:
  ✓ Finch        — Steps 7-9 done
  ● Greenlight   — Step 7 (critique) in progress
  ○ BusyKid      — Pending
```

## Pipeline (9 Steps)

| Step | Name | What Happens |
|------|------|-------------|
| **1** | Prerequisites, Profile & Directory Setup | Validates config, quick-checks NotebookLM auth and MCP, CAPTCHA pre-warming, selects profile, verifies notebook, confirms competitor names, collects domain keywords, checks for resume from partial runs, creates output directories. |
| **2** | Project Context Familiarization | Loads or creates project context (product name, audience, JTBD, differentiators, monetization, stage). Searches NotebookLM for project docs, offers quick briefing if none found, caches in config profile. Grounds Sections 3, 6, 8, 9 in actual product reality. |
| **3** | Pull Existing NotebookLM Sources | Lists all sources, filters by competitor name + domain keywords. User confirms include/exclude. Pulls fulltext for each source. Performs quick section gap analysis to inform targeted web research. |
| **4** | Web Research via Google AI Mode MCP | Runs 5 required searches (overview, traction, pricing, sentiment, market position) plus gap-targeted searches based on Step 3 analysis. Handles CAPTCHA via VNC. Saves results to `raw/web/`. |
| **5** | Analyze & Verify | Reads all raw sources. Flags wrong-company data, cross-references facts, checks freshness (>12 months), identifies gaps, checks for AI search contamination from same-name companies. Reports conflicts to user. |
| **6** | Write Section Files | Writes 10 section files (01-10), the index (00), and the combined full brief. Follows section definitions from `references/section-definitions.md` with confidence headers and inline citations `[S#-N]`. |
| **7** | Critique & Quality Review | **Runs BEFORE deleting NotebookLM sources** so originals remain if fatal flaws are found. Uses configured critique tool with graceful fallback cascade (configured → self-critique). Offers to revise affected sections. |
| **8** | Upload Consolidated Brief | **Uploads BEFORE deleting old sources** to eliminate data loss window. Uploads combined brief to NotebookLM. Verifies it appears in source list. If upload fails, informs user of local path -- does NOT proceed to deletion. |
| **9** | Clean Up NotebookLM Sources | Requires explicit user confirmation. Deletes old sources (unique + duplicates + wrong-company). Reports count freed. Handles mid-batch failures gracefully. |

Key ordering: critique runs BEFORE delete (Step 7 before 9), upload runs BEFORE delete
(Step 8 before 9). This ensures no data loss if critique finds fatal flaws or upload fails.

In parallel mode, Steps 3-6 run concurrently across all competitors via background agents. Steps 7-9 run sequentially in the main session.

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
├── .status.json                             # Pipeline progress (for resume capability)
└── raw/                                     # Raw source texts (for audit trail)
    ├── notebooklm/                          # Fulltexts pulled from NotebookLM
    │   ├── {source_id_short}.md
    │   └── ...
    └── web/                                 # Google AI Mode MCP research results
        ├── search-{topic}.md
        └── ...
```

## Brief Sections

Each section file follows this template:

```markdown
# {Section Title}
> Competitor: {name} | vs: {project_name} | Last Updated: {date} | Confidence: {high/medium/low}

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
| 7 | User Sentiment & Reviews | Product | Review themes, top loves/complaints, feature requests, churn signals. 3-5 verbatim quotes. |
| 8 | Strengths & Weaknesses | Pitch + Product | SWOT-style analysis. Honest assessment. Notes that user product comparisons reflect vision, not shipped product. |
| 9 | Threat Assessment | Pitch | Direct/indirect/potential classification, defensive moats, threat scenarios, strategic response |
| 10 | Sources & Data Quality | Pitch + Product | Full citation list, source quality tiers, wrong-company exclusions, data conflicts, per-section confidence |

## Skill Architecture

```
skills/competitor-research/
├── SKILL.md                                 # Main skill prompt (482 lines)
└── references/
    ├── setup.md                             # Setup wizard (loaded only during /competitor-research setup)
    ├── section-definitions.md               # Single source of truth for 10 section definitions
    └── agent-prompt-template.md             # Parallel agent prompt template (loaded only for parallel mode)
```

Section definitions are maintained in one place (`references/section-definitions.md`) and
referenced by both the main workflow and parallel agent prompts. This prevents definition
drift between single-competitor and parallel modes — a bug class that affected v0.5.0.

## Dependencies

### /vnc-service Plugin (headless servers)

Required for browser-based interactions (OAuth, CAPTCHA) on headless servers. If not installed,
the setup wizard installs it automatically:

```bash
claude plugin marketplace add 3D-Stories/vnc-service
claude plugin install vnc-service@vnc-service
```

User runs `/reload-plugins` to activate. The skill calls `/vnc-service:run` whenever browser
interaction is needed during research runs.

### Dependency Installation

Skills auto-install their dependencies via the `claude` CLI in Bash. The user only needs
to run `/reload-plugins` when prompted -- this is a session command that cannot be called
from Bash.

## Troubleshooting

| Error | Response |
|-------|----------|
| NotebookLM auth expired | Invokes `/vnc-service:run`, then runs `DISPLAY=:99 notebooklm login` |
| Google AI Mode CAPTCHA | Invokes `/vnc-service:run`, retries search with `headless: false` for user to solve in VNC |
| MCP not available | Inform user to restart session -- MCP servers only initialize at session start |
| Chromium distribution 'chrome' is not found | User runs `sudo npx patchright install chrome` in a separate terminal |
| Source fulltext fails | Logs source ID, skips it, notes in Section 10 as "content unavailable" |
| Ambiguous competitor name | Always asks the user to clarify, never guesses |
| Thin data | Writes honestly with what is available. Flags: "Section N is thin -- only X words" |
| Chrome singleton lock | Clears `/path/to/chrome_profile/Singleton*` and retries |
| Config missing | Auto-runs `/competitor-research setup` before proceeding |

## Version History

- **0.6.0** -- Current release. Major restructure driven by multi-agent critique review:
  - **Architecture**: Extracted setup wizard, section definitions, and agent prompt template
    to `references/` directory. SKILL.md reduced from 959 to 482 lines (under 500-line target).
    Section definitions now maintained in single source of truth, preventing definition drift.
  - **Bug fixes**: Fixed agent prompt template step numbering (was using pre-v0.5.0 numbers),
    fixed all stale step cross-references (lines 500, 619, 635, 747, 937-939, 945 in v0.5.0),
    resolved contradictory critique tool enforcement (now graceful fallback cascade instead of
    strict-then-fallback contradiction), fixed BMAD path indirection.
  - **Merged Steps 1+2**: Prerequisites Check and Profile/Notebook/Directory Setup merged into
    a single Step 1. Pipeline is now 9 steps (was 10); parallel agents run Steps 3-6 (was 4-7),
    sequential completion runs Steps 7-9 (was 8-10).
  - **Resume capability**: `.status.json` written after each step in both single-competitor and
    parallel pipelines. Detects partial completion and offers to resume from last completed step.
  - **CAPTCHA pre-warming**: Moved from parallel-mode-only to Step 1 prerequisites, benefiting
    both single-competitor and parallel paths.
  - **Description optimization**: Added trigger phrases for "competitive landscape", "market
    intelligence", "compare product to company", "who are our competitors", "analyze competition".
- **0.5.0** -- Added Project Context Familiarization (Step 3) to ground competitor
  briefs in the user's own product reality. Auto-detects project docs from NotebookLM, offers
  7-question quick briefing or doc upload if none found, caches context in config profile for
  subsequent runs. Sections 3, 6, 8, 9 now produce strategic comparisons instead of generic profiles.
  Section header template includes `vs: {project_name}`. Pipeline was 10 steps; parallel
  agents ran Steps 4-7, sequential completion ran Steps 8-10.
- **0.4.0** -- Parallel mode rewrite: CAPTCHA pre-warming in Phase 1, self-contained
  agent prompts with full section definitions and .status.json protocol, orchestrator blocks until all
  agents complete (no incremental notification), CAPTCHA-blocked agents write marker files for main
  session backfill, Phase 3 processes all competitors after all agents finish with batch critique option
  (self-critique all + BMAD deep-dive on weakest) for 3+ competitors, structured error recovery via
  .status.json. Inputs now require comma separation for multi-word competitor names.
- **0.3.5** -- Added Playwright system dependency install step (`playwright install-deps chromium`).
  Replaced poll-based login with two-step background/save pattern. Added orphan process cleanup guidance.
- **0.3.4** -- Previous release.
- **0.3.x** -- Incremental fixes and refinements to the pipeline, setup flow, and error handling.
- **0.2.0** -- Major rewrite. Added setup wizard (7-step first-run configuration), Google AI
  Mode MCP integration (replacing skill-based web research), BMAD/reflexion critique cascade,
  domain keywords for targeted searches, section gap analysis, wrong-company contamination
  checks, upload-before-delete safety ordering, and persistent config file with named profiles.
- **0.1.0** -- Initial release. 9-step pipeline with 10 sections, critique cascade,
  configurable output directory.
