# Brief Section Definitions

This is the **single source of truth** for the 10 competitor brief sections.
Referenced by the main workflow (Step 6: Write Section Files) and the parallel
agent prompt template.

## Section Template

Each section file follows this template:

```markdown
# {Section Title}
> Competitor: {name} | vs: {project_name} | Last Updated: {date} | Confidence: {high/medium/low}

{Section content with inline citations [S#-N] where S# is section number, N is source number}

---
*Sources: See 10-sources-and-data-quality.md for full citation list*
```

## Section 1: Company Overview
What the company does, when founded, headquarters, team size, leadership, mission statement,
and a 2-3 sentence positioning summary. Include the company's own description of itself
(from their website/app store listing) alongside an objective characterization.

## Section 2: Funding & Financials
All known funding rounds with dates, amounts, and lead investors. Total raised, most recent
valuation, estimated revenue (with confidence level), burn rate signals (hiring velocity,
office changes), and financial health indicators. Note which figures are confirmed vs estimated.
**Critical:** Flag any funding data that may be from a different company with the same name.

## Section 3: Product & Features
Core feature set organized by category. Platform availability (iOS, Android, web).
Key UX patterns and design philosophy. Recent feature launches (last 12 months).
Technology signals (job postings, blog posts). Integration ecosystem.
*Note: Comparisons to the user's product reflect product vision/design goals, not shipped product.*

## Section 4: Pricing & Monetization
Pricing tiers with exact amounts. Free vs paid feature breakdown. Subscription model
(monthly/annual, family plans). Estimated conversion rate (free-to-paid) if available.
ARPU signals. Price change history.

## Section 5: User Base & Traction
Downloads (App Store + Google Play), DAU/MAU estimates, app store ratings and review counts,
growth trajectory, geographic distribution, engagement metrics (session length, retention).
Note data freshness — app store numbers change quickly. Include growth rate analysis where data permits.
Label single-source estimates clearly.

## Section 6: Target Market & Positioning
Primary and secondary audiences. Age demographics. Market segment
(kids fintech, gamified wellness, family management, etc.). Brand voice and messaging strategy.
How they position against competitors. Marketing channels observed.
Include TAM/market size context if data available.

## Section 7: User Sentiment & Reviews
Themes from app store reviews, Reddit, social media. What users love most (top 3).
What users complain about most (top 3). Feature requests users frequently mention.
Churn signals and reasons. NPS or satisfaction data if available.
Quote 3-5 representative user comments verbatim with attribution.

## Section 8: Strengths & Weaknesses
SWOT-style analysis relative to the user's product. What this competitor does better.
Where the user's product has an advantage. Neutral differences.
Be honest — the point is to inform strategy, not to feel good.
*Note: User product comparisons reflect product vision, not shipped product.*

## Section 9: Threat Assessment
Is this a direct, indirect, or potential competitor? Likelihood of entering the user's
specific niche. Their defensive moats (network effects, brand, data, partnerships, capital).
Scenarios where they become a serious threat. Recommended strategic response.

## Section 10: Sources & Data Quality
Complete citation list with section citation map, source quality tiers (primary/secondary/tertiary),
wrong-company exclusions, blocked sources, and a data conflict resolution log.
Include confidence ratings per section with limiting factors.

## Writing Guidelines

- **Be specific**: "$35M estimated ARR" not "significant revenue"
- **Cite with section prefixes**: Use [S#-N] format for unambiguous citations
- **State confidence**: Each section header includes a confidence level
- **Note unknowns**: "No public data available" is better than omitting
- **Label single-source estimates**: Mark data from one source as "single-source estimate"
- **Product vision caveat**: When comparing to user's product, note these reflect
  product vision/design goals, not shipped product
- **Keep sections focused**: 300-800 words each
