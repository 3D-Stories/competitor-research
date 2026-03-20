---
name: competitor-research
description: >-
  Comprehensive competitor research pipeline that consolidates scattered sources into curated market briefs.
  Pulls existing NotebookLM sources, searches the web via Google AI Mode MCP, verifies accuracy,
  and produces structured per-section markdown files plus a combined brief for NotebookLM upload.
  Use this skill whenever the user wants to research a competitor, build a competitor profile, create a market brief,
  consolidate competitor sources, audit competitive intelligence, or mentions "competitor research", "market brief",
  "competitive analysis", or "competitor profile". Also trigger when the user asks to clean up or consolidate
  NotebookLM sources about a specific company. Includes a setup mode (/competitor-research setup) for first-run
  dependency configuration.
---

# Competitor Research Pipeline

Build comprehensive, verified competitor briefs from multiple source types — existing NotebookLM content,
web research, and local project documents — then consolidate everything into structured markdown files
organized by section.

The goal is dual-purpose output: briefs that serve **investor pitch materials** (market sizing, funding,
threat assessment) AND **product strategy decisions** (feature gaps, user sentiment, positioning).

## Prerequisites Check (Auto-Run)

**Before ANY research run**, check if the config file exists:

```bash
test -f ~/.config/competitor-research/config.json
```

**If missing:** Inform the user and run `/competitor-research setup` automatically:
"No configuration found. Running first-time setup..."
Then invoke the setup workflow below. Do NOT proceed with research until setup completes.

**If present:** Read the config and verify key dependencies are still working:
1. `notebooklm status` — if auth expired, call `/vnc-service:run` and re-auth
2. Check MCP `mcp__google-ai-search__search_ai` is in available tools — if missing, inform user to restart session
3. Proceed to research workflow

## Setup Mode (`/competitor-research setup`)

Run this once to configure all dependencies. Can also be re-run to reconfigure.

### Setup Step 1: Environment Detection

```bash
if [ -n "$DISPLAY" ] && xdpyinfo >/dev/null 2>&1; then
  HEADLESS=false
  echo "Graphical display detected"
else
  HEADLESS=true
  echo "Headless server detected — VNC will be needed for browser auth"
fi
```

### Setup Step 2: Virtual Display (headless only)

If headless:
1. Check if `/vnc-service:setup` skill is available
2. If available: invoke `/vnc-service:setup` to install and configure the virtual display + VNC
3. If not available: provide manual instructions for installing Xvfb + x11vnc
4. After setup, invoke `/vnc-service:run` to ensure it's running and get connection info
5. Wait for user to confirm VNC connection before proceeding

### Setup Step 3: NotebookLM CLI

1. Check: `notebooklm --version`
2. If missing, install via **pipx** (not pip — modern Ubuntu/Debian use PEP 668 which blocks
   direct pip install into the system Python):
   ```bash
   # Install the CLI
   pipx install notebooklm-py

   # Inject Playwright (needed for browser-based auth and source management)
   pipx inject notebooklm-py playwright

   # Install Chromium browser for Playwright (must use the venv's playwright binary)
   # Find the actual venv path:
   pipx list  # Look for the notebooklm-py venv location
   # Then install chromium using that path:
   $(pipx environment --value PIPX_LOCAL_VENVS)/notebooklm-py/bin/playwright install chromium
   ```

   Then install the Claude Code skill and reload:
   ```bash
   notebooklm skill install
   ```
   ```
   /reload-plugins
   ```

3. Check: `notebooklm status` for auth
4. If not authenticated:
   - If headless: invoke `/vnc-service:run`, then run login with DISPLAY=:99
   - If graphical: run login directly

   **Important:** The `notebooklm login` command opens a browser and waits for ENTER after
   OAuth completes. When running from Claude Code's Bash tool, stdin sends EOF immediately
   which aborts the login. Use this pattern instead:

   ```bash
   # Poll-based login: keeps process alive until auth file appears, then auto-confirms
   (while [ ! -f ~/.notebooklm/storage_state.json ] || \
     [ $(( $(date +%s) - $(stat -c %Y ~/.notebooklm/storage_state.json) )) -gt 10 ]; do \
     sleep 2; done; sleep 3; echo "") | notebooklm login
   ```

   This polls for the auth file to be written/updated, waits 3 seconds for it to finalize,
   then sends ENTER automatically. No fixed timeout — completes as soon as the user finishes
   OAuth.

   For headless servers, prefix with `DISPLAY=:99`:
   ```bash
   (while [ ! -f ~/.notebooklm/storage_state.json ] || \
     [ $(( $(date +%s) - $(stat -c %Y ~/.notebooklm/storage_state.json) )) -gt 10 ]; do \
     sleep 2; done; sleep 3; echo "") | DISPLAY=:99 notebooklm login
   ```

   Tell the user: "Complete OAuth in the browser. The login will auto-confirm once done."

5. Verify: `notebooklm status` shows authenticated

### Setup Step 4: Google AI Mode MCP

1. Check if `mcp__google-ai-search__search_ai` is in the available tools list
2. If missing: install the MCP server using the session `/mcp` command:
   ```
   /mcp add google-ai-search npx google-ai-mode-mcp@latest
   ```
   Note: Do NOT use `xvfb-run` wrapper — use the persistent display from `/vnc-service:setup`.
   The MCP server will use `DISPLAY=:99` from the environment.

   Inform user: "Google AI Mode MCP installed. You'll need to **restart the session** for the
   MCP server to initialize. MCP servers only load at session start."
   **HARD STOP — Do NOT proceed to Step 5 or any later step.** Steps 5-7 require the MCP
   server to be running, which only happens after a session restart. If you skip ahead, the
   user will be asked setup questions out of order and the config file will be written before
   MCP is verified. Tell the user to restart and re-run `/competitor-research setup` — it will
   detect that Steps 1-3 are already complete and resume from Step 4 verification.

3. If present: test with a simple search:
   ```
   mcp__google-ai-search__search_ai(query="test", headless=true)
   ```

4. If the test search fails with **"Chromium distribution 'chrome' is not found"**:
   Chrome/patchright browser is not installed. Tell the user:
   ```
   The Google AI Mode MCP needs Chrome installed. Run this in a separate terminal
   (Claude Code cannot handle interactive sudo):

     sudo npx patchright install chrome

   Note: `apt install google-chrome-stable` does NOT work — it's not in default Ubuntu repos.
   The patchright installer downloads the correct Chromium build.

   Let me know when it's done and I'll retry the test search.
   ```
   Wait for user confirmation, then retry the test search.

5. If the test search fails with **CAPTCHA required**:
   - Invoke `/vnc-service:run` for connection info
   - Retry with `headless: false` so browser renders on VNC display
   - User solves CAPTCHA in VNC
   - Verify a headless search works after CAPTCHA is cleared

### Setup Step 5: NotebookLM Notebook Selection

Ask the user:
```
Which NotebookLM notebook should competitor research use?
  1. Use an existing notebook (I'll list them)
  2. Create a new notebook

Which option?
```

**Option 1:** `notebooklm list` → user picks → `notebooklm use <id>`
**Option 2:** `notebooklm create "Market Research: {project_name}"` → set context

### Setup Step 6: Critique Tool Detection

Check for critique tools in priority order:

1. **BMAD Party Mode** — search for installation (check common paths first, then broad search):
   ```bash
   # Check common locations first (fast)
   for d in ~/bmad ~/BMAD ~/.bmad; do
     [ -f "$d/_bmad/_config/agent-manifest.csv" ] && echo "$d/_bmad" && break
   done
   # Fallback: broader search with depth limit (slower but thorough)
   find ~ -maxdepth 4 -name "agent-manifest.csv" -path "*/_bmad/*" 2>/dev/null | head -1
   ```
   If found: store the **parent directory of `_config/`** as `bmad_path` (e.g., `/home/user/bmad/_bmad`).
   Step 7 will reference the agent manifest at `{bmad_path}/_config/agent-manifest.csv`.

2. **reflexion:critique** — check if available in skills list.
   If available: use as critique tool.

3. **Neither found** — offer to install reflexion:
   ```
   No critique tool found. Would you like to install reflexion?
   This provides multi-judge quality review for briefs.

   Install commands (run in session):
     /plugin marketplace add NeoLabHQ/context-engineering-kit
     /plugin install reflexion@NeoLabHQ/context-engineering-kit
     /reload-plugins

   Install now? (y/n)
   ```

4. **Self-critique** as last resort — always available, no install needed.

### Setup Step 7: Save Configuration

Save all configuration to `~/.config/competitor-research/config.json`.

The config supports **named profiles** for users researching competitors across multiple
projects or notebooks. Setup creates a `default` profile. Users can add more by re-running
setup or manually editing the config.

```json
{
  "version": 2,
  "created_at": "ISO timestamp",
  "updated_at": "ISO timestamp",
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

```bash
mkdir -p ~/.config/competitor-research
```

Report:
```
Setup complete!
  ✓ Virtual display (Xvfb :99 + VNC on 5999)
  ✓ NotebookLM (authenticated)
  ✓ Google AI Mode MCP (CAPTCHA cleared)
  ✓ Notebook: {notebook_title}
  ✓ Critique: {bmad|reflexion|self}

Run /competitor-research <name> to start researching.
```

## Input

The skill accepts:
- **Competitor name** (required) — e.g., "Finch", "Greenlight", "BusyKid"
- **Domain keywords** (optional) — additional search terms specific to this competitor's market,
  e.g., "kids fintech debit card banking" for Greenlight, "gamified wellness self-care mental health"
  for Finch. If not provided, the skill will infer domain keywords from the competitor name and
  initial source analysis.
- **Notebook ID** (optional) — defaults to configured notebook from setup
- **Project path** (optional) — defaults to active rawgentic project path
- **Product clarifier** (optional) — disambiguation when a company name is ambiguous,
  e.g., "Finch self-care app" vs "Finch payroll API". Ask the user if unclear.

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

## Section Definitions

Each section file follows this template:

```markdown
# {Section Title}
> Competitor: {name} | Last Updated: {date} | Confidence: {high/medium/low}

{Section content with inline citations [S#-N] where S# is section number, N is source number}

---
*Sources: See 10-sources-and-data-quality.md for full citation list*
```

### Section 1: Company Overview
What the company does, when founded, headquarters, team size, leadership, mission statement,
and a 2-3 sentence positioning summary. Include the company's own description of itself
(from their website/app store listing) alongside an objective characterization.

### Section 2: Funding & Financials
All known funding rounds with dates, amounts, and lead investors. Total raised, most recent
valuation, estimated revenue (with confidence level), burn rate signals (hiring velocity,
office changes), and financial health indicators. Note which figures are confirmed vs estimated.
**Critical:** Flag any funding data that may be from a different company with the same name.

### Section 3: Product & Features
Core feature set organized by category. Platform availability (iOS, Android, web).
Key UX patterns and design philosophy. Recent feature launches (last 12 months).
Technology signals (job postings, blog posts). Integration ecosystem.
*Note: Comparisons to the user's product reflect product vision/design goals, not shipped product.*

### Section 4: Pricing & Monetization
Pricing tiers with exact amounts. Free vs paid feature breakdown. Subscription model
(monthly/annual, family plans). Estimated conversion rate (free-to-paid) if available.
ARPU signals. Price change history.

### Section 5: User Base & Traction
Downloads (App Store + Google Play), DAU/MAU estimates, app store ratings and review counts,
growth trajectory, geographic distribution, engagement metrics (session length, retention).
Note data freshness — app store numbers change quickly. Include growth rate analysis where data permits.
Label single-source estimates clearly.

### Section 6: Target Market & Positioning
Primary and secondary audiences. Age demographics. Market segment
(kids fintech, gamified wellness, family management, etc.). Brand voice and messaging strategy.
How they position against competitors. Marketing channels observed.
Include TAM/market size context if data available.

### Section 7: User Sentiment & Reviews
Themes from app store reviews, Reddit, social media. What users love most (top 3).
What users complain about most (top 3). Feature requests users frequently mention.
Churn signals and reasons. NPS or satisfaction data if available.
Quote 3-5 representative user comments verbatim with attribution.

### Section 8: Strengths & Weaknesses
SWOT-style analysis relative to the user's product. What this competitor does better.
Where the user's product has an advantage. Neutral differences.
Be honest — the point is to inform strategy, not to feel good.
*Note: User product comparisons reflect product vision, not shipped product.*

### Section 9: Threat Assessment
Is this a direct, indirect, or potential competitor? Likelihood of entering the user's
specific niche. Their defensive moats (network effects, brand, data, partnerships, capital).
Scenarios where they become a serious threat. Recommended strategic response.

### Section 10: Sources & Data Quality
Complete citation list with section citation map, source quality tiers (primary/secondary/tertiary),
wrong-company exclusions, blocked sources, and a data conflict resolution log.
Include confidence ratings per section with limiting factors.

## Workflow

Execute these steps in order. Each step has a clear checkpoint before proceeding.

### Step 1: Prerequisites Check

1. Check for `~/.config/competitor-research/config.json`
2. If missing: run setup mode (see above), then return here
3. If present: quick validation (notebooklm status, MCP available)
4. Load config values (notebook_id, critique_tool, bmad_path, etc.)

### Step 2: Profile, Notebook & Directory Setup

1. **Select profile:** Load the default profile from config. If multiple profiles exist,
   or if the user's current context doesn't match the default, ask:
   ```
   Using profile: "chorestory" (notebook: ChoreStory Investor Research)
   Use this profile, or switch? Available profiles: chorestory, [other...]
   Or: create a new profile for this research
   ```
   If no profile matches the current work, create a new one inline (ask for notebook + path).

2. **Verify notebook:** Set NotebookLM context to the profile's notebook:
   `notebooklm use {profile.notebook_id}`
   If the notebook no longer exists (deleted or shared access revoked), ask the user to
   pick a new one and update the profile.

3. **Verify project path:** Check if `{profile.project_path}` exists on disk.
   If not set or missing: ask the user for the output base directory.
   If the active rawgentic project path is detected and differs from the profile,
   ask: "Use {rawgentic_path} or {profile.project_path}?"

4. Confirm the competitor name with the user
5. If the name is ambiguous, ask the user to clarify which company/product
6. Ask for **domain keywords** (or infer from name): "Any specific search terms for this
   competitor's market? (e.g., 'kids fintech debit card' for Greenlight)"
7. Suggest output directory, let user override:
   ```
   Output: {project_path}/competitors/{competitor_name}/
   Change? (Enter path or press Enter to accept)
   ```
8. Create directory structure: `{output_dir}/raw/notebooklm/` and `{output_dir}/raw/web/`

**Checkpoint:** Profile loaded, notebook context set, directory exists, competitor unambiguous, domain keywords set.

### Step 3: Pull Existing NotebookLM Sources

1. Run `notebooklm source list --json` to get all sources
2. Filter sources by competitor name AND domain keywords (case-insensitive)
3. Also identify likely-related sources (app store URLs, known product URLs)
4. Present the filtered list: "I found {N} sources about {competitor}. Should I include/exclude any?"
5. For each confirmed source, run `notebooklm source fulltext {source_id} --json`
6. Save each fulltext to `raw/notebooklm/{source_id_first8chars}.md` with YAML header
7. Track all source IDs: unique sources, duplicate IDs, wrong-company IDs
8. **Quick section mapping**: Before web research, scan pulled sources and note which sections
   have strong data vs. gaps. This informs targeted web research in Step 4.

**Checkpoint:** Fulltexts saved. Source IDs recorded. Section gap analysis complete.

### Step 4: Web Research via Google AI Mode MCP

Run targeted searches using the `mcp__google-ai-search__search_ai` MCP tool.
Use domain keywords from Step 2 to make searches competitor-specific.

**Required searches** (adapt competitor name + domain keywords):
1. `"{competitor} {domain_keywords} company overview founding team funding {year}"` → company-overview.md
2. `"{competitor} {domain_keywords} revenue users downloads metrics {year}"` → traction-metrics.md
3. `"{competitor} {domain_keywords} pricing subscription model free vs paid {year}"` → pricing.md
4. `"{competitor} {domain_keywords} reviews user sentiment Reddit complaints {year}"` → user-sentiment.md
5. `"{competitor} {domain_keywords} competitors market position comparison {year}"` → market-position.md

**Gap-targeted searches** (run based on Step 3 section gap analysis):
6-8. Additional searches targeting specific sections that are data-thin

Save each result to `raw/web/search-{topic}.md` with YAML header (query, date, source).

**CAPTCHA handling:** If a search returns `captchaRequired: true`:
1. Invoke `/vnc-service:run` for connection info
2. Retry the search with `headless: false`
3. User solves CAPTCHA in VNC
4. Retry remaining searches with `headless: true`

**Checkpoint:** At least 5 web research files saved.

### Step 5: Analyze & Verify

Read ALL raw sources (both NotebookLM and web) and:

1. **Company verification**: Flag any sources about a different company with the same name
2. **Fact extraction**: Pull specific data points with source attribution per section
3. **Cross-reference**: Note agreements and discrepancies between sources
4. **Freshness check**: Flag data older than 12 months
5. **Gap identification**: Report remaining gaps to user
6. **Web research contamination check**: Flag any AI-generated web research that contains
   data from a wrong-company (this is common — AI search results conflate same-name companies)

**Checkpoint:** User informed of wrong-company sources, gaps, and conflicts.

### Step 6: Write Section Files

Write each section file following the section definitions above. Guidelines:

- **Be specific**: "$35M estimated ARR" not "significant revenue"
- **Cite with section prefixes**: Use [S#-N] format for unambiguous citations
- **State confidence**: Each section header includes a confidence level
- **Note unknowns**: "No public data available" is better than omitting
- **Label single-source estimates**: Mark data from one source as "single-source estimate"
- **Product vision caveat**: When comparing to user's product, note these reflect
  product vision/design goals, not shipped product
- **Keep sections focused**: 300-800 words each

Write files 01 through 10, then:
- Create `00-index.md` with executive summary, reading paths (investor track / product track),
  table of contents, and key numbers at a glance
- Create `{competitor_name}-full-brief.md` concatenating all sections with `---` dividers

### Step 7: Critique & Quality Review

**Run BEFORE deleting NotebookLM sources** — if the critique finds fatal flaws, we still have
the original sources to work with.

Use the critique tool from config:

1. **BMAD Party Mode** (if bmad_path configured): Read the agent manifest at
   `{bmad_path}/_config/agent-manifest.csv` and orchestrate a multi-agent review
   focusing on: factual accuracy, source quality, analytical rigor, completeness,
   and actionability.

2. **reflexion:critique**: Invoke with the full brief path and evaluation criteria.

3. **Self-critique**: Re-read the brief and evaluate against: factual accuracy, source quality,
   completeness, analytical rigor, actionability, bias check.

**After critique:** If significant issues found, offer to revise affected sections.
Regenerate the combined file if revisions are made.

**Checkpoint:** Critique complete. Brief finalized.

### Step 8: Upload Consolidated Brief FIRST

**Upload BEFORE deleting old sources** — this eliminates the data loss window. If upload
fails, the original sources are still intact.

1. Upload: `notebooklm source add {path_to_full_brief}`
2. Wait: `notebooklm source wait {source_id}`
3. Verify it appears: `notebooklm source list`

If upload fails: inform user the brief is available locally at `{path}` and can be manually
uploaded later. Do NOT proceed to deletion.

### Step 9: Clean Up NotebookLM Sources

**Requires user confirmation. Only proceed after Step 8 upload is verified.**

1. Present the list of source IDs to delete (unique + duplicates + wrong-company)
2. Ask: "The consolidated brief is uploaded and verified. Ready to delete these {N} old
   sources? (y/n)"
3. On confirmation: delete each source via `notebooklm source delete {source_id}`
4. If any deletion fails mid-batch: log the failed ID, continue with remaining deletions,
   report "Deleted {N}/{total}. Failed: {list}" at end. If auth expired mid-batch, re-auth
   via `/vnc-service:run` and retry failed deletions.

Report:
```
Uploaded: {competitor_name}-full-brief.md
Status: ready
NotebookLM sources: was {old_count}, now {new_count} (freed {delta} slots)
```

## Running for Multiple Competitors

Suggest processing one at a time, ordered by source count (heaviest first to free slots early).
Each competitor reuses the same config from setup.

## Error Handling

| Error | Response |
|-------|----------|
| NotebookLM auth expired | Invoke `/vnc-service:run`, then `DISPLAY=:99 notebooklm login` |
| Google AI Mode CAPTCHA | Invoke `/vnc-service:run`, retry with `headless: false` |
| MCP not available | Inform user to restart session (MCP initializes at session start) |
| Source fulltext fails | Log source ID, skip, note in Section 10 as "content unavailable" |
| Ambiguous competitor name | Always ask, never guess |
| Thin data | Write honestly. Flag: "Section {N} is thin — only {X} words" |
| Chrome singleton lock | Clear `/path/to/chrome_profile/Singleton*` and retry |
| Config missing | Auto-run `/competitor-research setup` |
