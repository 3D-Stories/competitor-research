---
name: competitor-research
description: >-
  Comprehensive competitor research pipeline that consolidates scattered sources into curated market briefs.
  Pulls existing NotebookLM sources, searches the web for additional data via google-ai-mode, verifies accuracy,
  and produces structured per-section markdown files plus a combined brief for NotebookLM upload.
  Use this skill whenever the user wants to research a competitor, build a competitor profile, create a market brief,
  consolidate competitor sources, audit competitive intelligence, or mentions "competitor research", "market brief",
  "competitive analysis", or "competitor profile". Also trigger when the user asks to clean up or consolidate
  NotebookLM sources about a specific company.
---

# Competitor Research Pipeline

Build comprehensive, verified competitor briefs from multiple source types — existing NotebookLM content,
web research, and local project documents — then consolidate everything into structured markdown files
organized by section.

The goal is dual-purpose output: briefs that serve **investor pitch materials** (market sizing, funding,
threat assessment) AND **product strategy decisions** (feature gaps, user sentiment, positioning).

## Prerequisites

Before running this skill, verify these dependencies **in order**. If any fail, attempt
auto-install. If auto-install fails, stop and tell the user what to install manually.

### 1. NotebookLM CLI (`notebooklm`)

Check: `notebooklm --version`

If missing, install from PyPI and set up the skill:
```bash
pip install notebooklm-py
notebooklm skill install
```
The package is also available from GitHub: `https://github.com/teng-lin/notebooklm-py`
(always install from the latest release tag, NOT the main branch).

After install, authenticate: `notebooklm login` (opens browser for Google OAuth).
Verify with: `notebooklm status`

### 2. NotebookLM notebook selection

After auth is confirmed, ask the user whether to use an existing notebook or create a new one:

```
NotebookLM is ready. Do you want to:
  1. Use an existing notebook (I'll list them)
  2. Create a new notebook for this research

Which option?
```

**Option 1 — Existing notebook:**
1. Run `notebooklm list` to show all notebooks
2. Ask the user to pick one by number or name
3. Set context: `notebooklm use <selected_id>`

**Option 2 — New notebook:**
1. Suggest a title: "Market Research: {project_name}" (user can override)
2. Create it: `notebooklm create "<title>"`
3. Set context with the new notebook ID

Store the notebook ID for the rest of the workflow. All subsequent `notebooklm` commands
should use this notebook.

### 3. Google AI Mode skill

Check: `test -f ~/.claude/skills/google-ai-mode/SKILL.md`

If missing, inform the user:
"The google-ai-mode skill is required for web research. Install it before continuing.
See the plugin README for installation instructions."

### 4. Project path

The active rawgentic project must have a configured path (for the default output directory).
If no project is active, ask the user to specify an output directory manually.

## Input

The skill accepts:
- **Competitor name** (required) — e.g., "Finch", "Greenlight", "BusyKid"
- **Notebook ID** (optional) — defaults to current NotebookLM context
- **Project path** (optional) — defaults to active rawgentic project path
- **Product clarifier** (optional) — disambiguation when a company name is ambiguous,
  e.g., "Finch self-care app" vs "Finch payroll API". Ask the user if unclear.

## Output Structure

```
{project_path}/competitors/{competitor_name}/
├── 00-index.md                              # Table of contents with section summaries
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
    └── web/                                 # Google AI mode research results
        ├── search-{topic}.md
        └── ...
```

## Section Definitions

Each section file follows this template:

```markdown
# {Section Title}
> Competitor: {name} | Last Updated: {date} | Confidence: {high/medium/low}

{Section content with inline citations [1][2][3]}

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

### Section 3: Product & Features
Core feature set organized by category. Platform availability (iOS, Android, web).
Key UX patterns and design philosophy. Recent feature launches (last 12 months).
Technology signals (job postings, blog posts). Integration ecosystem.
Compare feature coverage to ChoreStory where relevant.

### Section 4: Pricing & Monetization
Pricing tiers with exact amounts. Free vs paid feature breakdown. Subscription model
(monthly/annual, family plans). Estimated conversion rate (free-to-paid) if available.
ARPU signals. Price change history. How pricing compares to ChoreStory's $5/$10 model.

### Section 5: User Base & Traction
Downloads (App Store + Google Play), DAU/MAU estimates, app store ratings and review counts,
growth trajectory, geographic distribution, engagement metrics (session length, retention).
Note data freshness — app store numbers change quickly.

### Section 6: Target Market & Positioning
Primary and secondary audiences. Age demographics. Market segment
(kids fintech, gamified wellness, family management, etc.). Brand voice and messaging strategy.
How they position against competitors. Marketing channels observed.

### Section 7: User Sentiment & Reviews
Themes from app store reviews, Reddit, social media. What users love most (top 3).
What users complain about most (top 3). Feature requests users frequently mention.
Churn signals and reasons. NPS or satisfaction data if available.
Quote 3-5 representative user comments verbatim with attribution.

### Section 8: Strengths & Weaknesses
SWOT-style analysis *relative to ChoreStory*. What this competitor does better than ChoreStory.
Where ChoreStory has an advantage. Neutral differences (different approach, neither better).
Be honest — the point is to inform strategy, not to feel good.

### Section 9: Threat Assessment
Is this a direct, indirect, or potential competitor? Likelihood of entering ChoreStory's
specific niche (cooperative family RPG + chores). Their defensive moats (network effects,
brand, data, partnerships, capital). Scenarios where they become a serious threat.
Recommended strategic response.

### Section 10: Sources & Data Quality
Complete citation list with:
- Source title, URL, and access date
- Source type (official, third-party, user-generated, estimated)
- Confidence rating per source (high/medium/low)
- Data freshness (when was this information current?)
- Any sources that were attempted but blocked/unavailable

This section exists so that every claim in the brief is traceable. If someone in a pitch
meeting asks "where did you get that $35M ARR number?", you can point here.

## Workflow

Execute these steps in order. Each step has a clear checkpoint before proceeding.

### Step 1: Setup & Disambiguation

1. Confirm the competitor name with the user
2. If the name is ambiguous (e.g., "Finch" could be the self-care app or the payroll API),
   ask the user to clarify which company/product
3. **Suggest output directory** and let the user override:
   ```
   I'll save the competitor brief to:
     {project_path}/competitors/{competitor_name}/

   Change directory? (Enter a path or press Enter to accept)
   ```
   Default: `{active_rawgentic_project_path}/competitors/{competitor_name}/`
   The user may specify any absolute or relative path. If relative, resolve against the
   workspace root. Store the chosen path — all subsequent steps use it.
4. Create the directory structure: `{output_dir}/raw/notebooklm/` and `{output_dir}/raw/web/`
5. Confirm the NotebookLM notebook context is set

**Checkpoint:** Directory exists, notebook context confirmed, competitor unambiguous, output path confirmed.

### Step 2: Pull Existing NotebookLM Sources

1. Run `notebooklm source list --json` to get all sources
2. Filter sources whose title or URL contains the competitor name (case-insensitive)
3. Also identify sources that are likely related but don't contain the exact name
   (e.g., app store URLs, known product URLs)
4. Present the filtered list to the user: "I found {N} sources that appear to be about {competitor}.
   Here they are: {list}. Should I include/exclude any?"
5. For each confirmed source, run `notebooklm source fulltext {source_id} --json`
6. Save each fulltext to `raw/notebooklm/{source_id_first8chars}.md` with a YAML header:
   ```yaml
   ---
   source_id: {full_id}
   title: {title}
   url: {url or "uploaded file"}
   type: {source_type}
   pulled_at: {ISO timestamp}
   ---
   ```
7. Track which source IDs were pulled (needed for Step 7 deletion)

**Checkpoint:** All relevant fulltexts saved locally. Source ID list recorded.

### Step 3: Web Research via Google AI Mode

Run targeted searches to fill gaps that NotebookLM sources might not cover. Execute these
searches using the google-ai-mode skill (`/google-ai-mode`), saving results to `raw/web/`.

**Required searches** (adapt the competitor name into each query):
1. `"{competitor} app company overview founding team funding 2026"` → company-overview.md
2. `"{competitor} app revenue users downloads metrics 2026"` → traction-metrics.md
3. `"{competitor} app pricing subscription model free vs paid 2026"` → pricing.md
4. `"{competitor} app reviews user sentiment Reddit complaints 2026"` → user-sentiment.md
5. `"{competitor} app competitors market position comparison 2026"` → market-position.md

**Conditional searches** (run if NotebookLM sources are thin in these areas):
6. `"{competitor} app features product updates roadmap 2026"` → product-features.md
7. `"{competitor} funding rounds investors valuation crunchbase 2026"` → funding-details.md
8. `"{competitor} hiring jobs careers team growth 2026"` → hiring-signals.md

Save each result to `raw/web/search-{topic}.md`.

**Checkpoint:** At least 5 web research files saved. User informed of what was found.

### Step 4: Analyze & Verify

This is the critical thinking step. Read ALL raw sources (both NotebookLM and web) and:

1. **Company verification**: Confirm all sources are about the correct company/product.
   Flag any that are about a different entity with the same name.
2. **Fact extraction**: For each section, pull out specific data points with their source.
   Prefer primary sources (company website, app store) over secondary (blog posts, estimates).
3. **Cross-reference**: When multiple sources cite the same figure, note agreement.
   When they disagree, note the discrepancy and which source is more authoritative.
4. **Freshness check**: Flag any data older than 12 months as potentially stale.
5. **Gap identification**: Note which sections have strong data and which are thin.
   Report gaps to the user before writing.

**Checkpoint:** User informed of any wrong-company sources, data gaps, and conflicting data.

### Step 5: Write Section Files

Write each section file following the section definitions above. Guidelines:

- **Be specific**: "$35M estimated ARR" not "significant revenue"
- **Cite everything**: Use [1][2][3] inline citations keyed to Section 10
- **State confidence**: Each section header includes a confidence level
- **Note unknowns**: "No public data available" is better than omitting the topic
- **Keep sections focused**: Each file should be 300-800 words. If a section is thin
  (under 150 words of real content), that's fine — say what you know honestly
- **ChoreStory comparisons**: Sections 3, 4, 8, and 9 should include ChoreStory-relative analysis

Write the files in order (01 through 10), then:
- Create `00-index.md` with a table of contents and one-line summary per section
- Create `{competitor_name}-full-brief.md` that concatenates all sections with `---` dividers
  and includes the index at the top

### Step 6: Save to Project

All files should already be in `{project_path}/competitors/{competitor_name}/`.
Verify the directory structure matches the Output Structure above.

Report to the user:
```
Competitor brief complete: {competitor_name}
  Sections: 10 files + index + combined brief
  Raw sources: {N} from NotebookLM, {M} from web research
  Location: {project_path}/competitors/{competitor_name}/
```

### Step 7: Clean Up NotebookLM Sources

**This step requires user confirmation before proceeding.**

1. Present the list of NotebookLM source IDs that were pulled in Step 2
2. Ask: "Ready to delete these {N} individual sources from NotebookLM and replace
   them with the consolidated brief? (y/n)"
3. On confirmation, delete each source: `notebooklm source delete {source_id}`
4. Also identify and delete any exact duplicate sources (same URL, added twice)
   that relate to this competitor

**Checkpoint:** Old scattered sources removed. User confirmed.

### Step 8: Upload Consolidated Brief

1. Upload the combined file: `notebooklm source add {project_path}/competitors/{competitor_name}/{competitor_name}-full-brief.md`
2. Wait for processing: `notebooklm source wait {source_id}`
3. Verify it appears in the source list: `notebooklm source list`

Report:
```
Uploaded: {competitor_name}-full-brief.md
Status: ready
NotebookLM sources: was {old_count}, now {new_count} (freed {delta} slots)
```

### Step 9: Critique & Quality Review

Run a structured critique on the completed brief to catch factual errors, unsupported claims,
missing context, and analytical blind spots before the user relies on it.

**Critique tool cascade** — use the first available option:

1. **`/bmad-party-mode`** (preferred) — Check if the `bmad-party-mode` skill is in the available
   skills list. If present, invoke it with the full brief file as input.
2. **`/reflexion:critique`** (fallback) — If bmad-party-mode is not installed, use the reflexion
   critique skill. Invoke it with context about what was produced:
   ```
   /reflexion:critique
   Review the competitor market brief at {path_to_full_brief}.
   Evaluate for: factual accuracy, source quality, analytical rigor,
   completeness across all 10 sections, and actionability for both
   investor pitch and product strategy contexts.
   ```
3. **Self-critique** (last resort) — If neither skill is available, perform an inline critique
   by re-reading the full brief and evaluating against these criteria:
   - **Factual accuracy**: Are any claims unsupported or contradicted by the raw sources?
   - **Source quality**: Are critical claims backed by primary sources, not just blog posts?
   - **Completeness**: Does every section have substantive content, or are some just filler?
   - **Analytical rigor**: Does the threat assessment actually assess threats, or just describe?
   - **Actionability**: Could someone make a pitch or product decision based on this brief?
   - **Bias check**: Is the brief too generous or too dismissive of the competitor?

   Write a short critique summary (5-10 bullet points) and present to the user.

**After the critique:** If significant issues are found (factual errors, missing critical data,
weak analysis), offer to revise the affected sections before considering the brief complete.
Update the combined file and re-upload to NotebookLM if revisions are made.

**Checkpoint:** Critique complete. User informed of any issues. Brief finalized.

## Running for Multiple Competitors

When the user wants to process all competitors, suggest running them one at a time
rather than in parallel, because:
- Each competitor's Step 2 (source identification) benefits from user review
- Web research via google-ai-mode is sequential (browser-based)
- The user should validate each brief before the sources are deleted

Suggest a processing order based on source count (heaviest first, to free the most slots early).

## Error Handling

- **NotebookLM auth expired**: Prompt user to run `notebooklm login`
- **Google AI mode CAPTCHA**: Suggest `--show-browser` flag for manual solving
- **Source fulltext fails**: Log the source ID, skip it, note in Section 10 as "content unavailable"
- **Ambiguous competitor name**: Always ask rather than guess
- **Thin data**: Write the section honestly with what you have. Don't pad with fluff.
  Flag to the user: "Section {N} is thin — only {X} words of substantive content"
