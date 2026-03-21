# Parallel Agent Prompt Template

Instantiate this template per competitor, replacing all `{placeholder}` values.
Each agent runs Steps 3-6 independently. Steps 7-9 run in the main session afterward.

The section definitions referenced below are in `references/section-definitions.md`.
Read that file and include the full section definitions in the agent prompt so the
agent is fully self-contained.

---

## Template

```
You are researching competitor "{competitor_name}" ({product_clarifier}).
Domain keywords: {domain_keywords}
Output directory: {output_dir}
NotebookLM notebook ID: {notebook_id}
Current year: {current_year}
Project context: {project_context_summary}

Available tools: Bash, Read, Write, Edit, Glob, Grep, mcp__google-ai-search__search_ai
Do NOT run Steps 7-9 (critique, upload, delete). Only Steps 3-6.

FIRST: Set NotebookLM context:
  notebooklm use {notebook_id}

STEP 3: Pull NotebookLM Sources
- Run: notebooklm source list --json
- Filter for sources matching "{competitor_name}" (case-insensitive) in title or URL
- Also check for domain keyword matches and known product URLs
- For each match, run: notebooklm source fulltext {source_id} --json
- Save to {output_dir}/raw/notebooklm/{source_id_first8chars}.md with YAML header
- Track source IDs in {output_dir}/raw/notebooklm/_metadata.json
- Note which sections have strong data vs gaps
- Update {output_dir}/.status.json: {"step": 3, "completed": true}

STEP 4: Web Research
- Use mcp__google-ai-search__search_ai for these queries:
  General:
  1. "{competitor_name} {domain_keywords} company overview founding team funding {current_year}"
  2. "{competitor_name} {domain_keywords} revenue users downloads metrics {current_year}"
  3. "{competitor_name} {domain_keywords} pricing subscription model {current_year}"
  4. "{competitor_name} {domain_keywords} reviews user sentiment Reddit {current_year}"
  5. "{competitor_name} {domain_keywords} competitors market position {current_year}"
  LinkedIn:
  6. "{competitor_name} site:linkedin.com/company employees team size headquarters"
  7. "{competitor_name} site:linkedin.com CEO founder CTO leadership executive"
  8. "{competitor_name} site:linkedin.com hiring jobs open positions engineering product"
  9. "{competitor_name} site:linkedin.com funding raised series investors announcement"
  10. "{competitor_name} {domain_keywords} site:linkedin.com product launch update announcement"
- Run additional gap-targeted searches based on Step 3 section gap analysis
- Save results to {output_dir}/raw/web/search-{topic}.md
- If CAPTCHA: write "captcha" to {output_dir}/.captcha_blocked and skip remaining
  searches. Do NOT retry — the main session will handle CAPTCHA resolution.
- Update {output_dir}/.status.json: {"step": 4, "completed": true, "searches_completed": N,
  "captcha_blocked": true/false}

STEP 5: Analyze & Verify
- Read ALL raw sources (both NotebookLM and web)
- Flag wrong-company sources (common with similar names)
- Cross-reference conflicting data between sources
- Check data freshness (flag >12 months old)
- Check for AI search contamination from same-name companies
- Identify section gaps
- Update {output_dir}/.status.json: {"step": 5, "completed": true}

STEP 6: Write Section Files
{INCLUDE_SECTION_DEFINITIONS_HERE}

Guidelines: be specific ($35M not "significant"), cite everything [S#-N], state confidence,
note unknowns honestly, 300-800 words per section.

- Write 01-10 section files
- Write 00-index.md with executive summary and reading paths
- Write {competitor_name}-full-brief.md (combined)
- Update {output_dir}/.status.json: {"step": 6, "completed": true, "word_count": N,
  "sections_written": 10}

FINAL STATUS: Write complete {output_dir}/.status.json:
{"competitor": "{competitor_name}", "completed": true, "steps_completed": [3,4,5,6],
 "word_count": N, "sections_written": 10, "captcha_blocked": false,
 "errors": [], "started_at": "ISO", "finished_at": "ISO"}
```

---

## Template Usage Notes

- The `{INCLUDE_SECTION_DEFINITIONS_HERE}` placeholder must be replaced with the full
  section definitions from `references/section-definitions.md` (the section template and
  all 10 section descriptions). Agents cannot read reference files — they need the content
  inline in their prompt.
- The `{project_context_summary}` should be a condensed version of the project context
  from Step 2 (product name, audience, JTBD, differentiators, stage).
- Agents share the same MCP server — web searches queue through one browser (sequential
  but fast). Analysis and writing are truly parallel.
- Agents share the same NotebookLM session — source list calls may interleave but this is
  safe (read-only). Each agent calls `notebooklm use {notebook_id}` first (idempotent).
- Each agent writes to its own `{output_dir}` — no file conflicts.
- No agent should delete or upload NotebookLM sources.
