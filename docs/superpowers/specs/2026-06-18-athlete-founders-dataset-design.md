# Athlete-Founders Dataset — Design

**Date:** 2026-06-18
**Owner:** jsaltzman@a16z.com
**Status:** Draft for review

## Goal

Build as exhaustive a dataset as possible of companies founded by former **professional or Olympic athletes** that received **institutional venture backing** (including companies that have since exited). For each person, capture which league/sport they came from, their athletic performance, and their non-athletic background. The dataset must be rigorous enough to (a) serve as a living internal research/sourcing asset and (b) back a published a16z editorial piece.

## Scope (decided)

- **Athlete population:** Professional (major leagues, global) + Olympic/Paralympic athletes only. *Not* general collegiate. Elite-college athletes appear only when they also reached pro/Olympic level. (Collegiate-only is an explicit out-of-scope item, revisitable as a future phase.)
- **Geography:** Global. US-first in execution sequencing, but the schema and sources support international leagues and all national Olympic teams.
- **Venture-backed definition:** Company raised **≥1 funding round with at least one institutional VC investor participating.** Excludes pure friends-and-family / angel-only / bootstrapped. Includes companies that have since been acquired, gone public, or shut down.
- **Deliverable:** Both — a rigorous, refreshable dataset *and* editorial-ready aggregates.
- **Founding-era cutoff:** Companies **founded in 2000 or later**. Earlier ventures by qualifying athletes are out of scope.
- **Performance depth:** **Full stat lines per sport** (not just summary markers).
- **Storage:** **CSV** for now (structured tables exported as CSV; CRM-linkable IDs retained as columns for later import).

## Out of scope

- Collegiate-only athletes (no pro/Olympic appearance).
- Companies with no institutional VC round (bootstrapped, angel-only, franchise/restaurant/CPG with no VC, personal brand licensing deals).
- Athletes as *investors only* (LP/angel) with no founder/co-founder operating role — captured as a flag but not a qualifying record.

---

## Architecture: hybrid bidirectional discovery (Approach 3)

The intersection of two large, messy, global populations — athletes (**U_A**) and VC-backed founders (**U_F**) — is found by running several independent **recall channels** into one shared candidate pool, then concentrating all **precision** work in a single entity-resolution + verification stage.

Design stance: **do not exhaustively ingest U_A.** Athlete databases are used as an on-demand verification/enrichment *reference*, not bulk-ingested. Discovery is driven from the VC side, Wikidata, and curated seeds.

```
                ┌─────────────────────────────────────────┐
  Channel A ───►│                                           │
  (founder      │                                           │
   signal-mine) │                                           │
                │            CANDIDATE POOL                 │
  Channel B ───►│   (person ⇄ possible company link)        │
  (Wikidata     │                                           │
   athlete×     │                                           │
   founder)     │                                           │
                │                                           │
  Channel C ───►│                                           │
  (curated      └───────────────────┬───────────────────────┘
   seeds)                           │
                                     ▼
                      ┌──────────────────────────────┐
                      │ 3. ENTITY RESOLUTION          │
                      │    (athlete-rec ⇄ founder-rec │
                      │     → one person + confidence)│
                      └───────────────┬───────────────┘
                                      ▼
                      ┌──────────────────────────────┐
                      │ 4. VC VERIFICATION            │
                      │    (PitchBook deal history,   │
                      │     incl. exits)              │
                      └───────────────┬───────────────┘
                                      ▼
                      ┌──────────────────────────────┐
                      │ 5. ENRICHMENT                 │
                      │    (performance + background) │
                      └───────────────┬───────────────┘
                                      ▼
                      ┌──────────────────────────────┐
                      │ 6. QA / dedup / review queue  │
                      └───────────────┬───────────────┘
                                      ▼
                          7. OUTPUTS (dataset + aggregates)
```

---

## Stage 1 — Universe construction

**U_F (VC-backed founders) — the primary frame.** Build from PitchBook (via the PitchBook Premium / Argo MCP integration) as the authoritative source of venture-backed companies and their founders, cross-checked with Crunchbase. This universe is bounded and high-quality and is needed anyway for the venture-backed filter and company/funding fields. We do **not** materialize all of U_F upfront; it is queried per candidate and per recall channel.

**U_A (athletes) — reference, not ingested.** Used on demand for verification and performance enrichment:
- **Sports Reference family** — Pro-Football-Reference, Basketball-Reference, Baseball-Reference, Hockey-Reference, Sports-Reference / Olympedia (Olympics). Structured rosters, stats, draft data.
- **Olympedia** + Olympic.org / IOC and national Olympic committee databases — Olympic/Paralympic participation and medals.
- **Wikidata / Wikipedia** — global coverage of notable athletes; Wikipedia articles frequently document post-athletic business ventures.
- **League official sites / ESPN / Transfermarkt (soccer)** — international leagues.

## Stage 2 — Recall channels

**Channel A — Founder-first signal mining.** Over PitchBook/Crunchbase founders, scan enrichment text (PitchBook bios, LinkedIn via Crustdata, web/news) for athletic-career signals: league names, "Olympian / Olympic," "Team USA / [national team]," "drafted by," "played professional," "former [sport] player," national-team caps, etc. Hits become candidates. *Strength:* directly mines the venture-backed set. *Weakness:* misses founders whose bios omit athletics — covered by B and C.

**Channel B — Wikidata athlete × founder cross-section.** Query Wikidata for humans who hold **both** an athletic property (e.g., sport / member of sports team / participant in Olympics) **and** a founder/entrepreneur/CEO-type occupation, or who are linked to an organization via "founded by." This yields a high-recall, *structured* seed of athlete-entrepreneurs globally without scraping all of U_A. Each becomes a candidate carrying both athletic and (claimed) company identifiers.

**Channel C — Curated seeds.** Ingest existing human-curated lists: press features on athlete-founders, sports-tech/athlete-business media, athlete-led VC funds and their operating-company portfolios, athlete-focused accelerators. Used two ways: (1) as high-precision seed candidates, and (2) as a **recall benchmark** — the fraction of Channel-C known cases the automated channels independently rediscover is our coverage metric.

All channels emit a uniform candidate record: `{person_name, aliases, athletic_claim, company_claim, source, source_url}`.

## Stage 3 — Entity resolution (the dedup core)

This stage answers "is athlete-record X the same human as founder-record Y, and are these duplicate records of one person/one company?"

**De-dup within each universe first.**
- *Within U_A:* one athlete may appear in multiple databases and across college→pro→Olympic. Collapse to a **canonical person** (Wikidata QID where available; else name + birth-year + sport key).
- *Within U_F:* one founder may have multiple ventures, and one company may have duplicate records across sources. Collapse to **canonical person** and **canonical company** IDs (PitchBook IDs as spine).

**Then resolve across universes (athlete-rec ⇄ founder-rec).**
- **Blocking** to avoid full cross-join: normalize names (Unicode-fold accents, handle transliteration/romanization for global names, nickname tables) and block on name tokens + coarse attributes.
- **Disambiguating signals** (weighted; common-name collisions are the central risk):
  - *Explicit dual-identity source* (strongest): a single source asserts both roles — a LinkedIn profile listing pro career + founder role; a Wikipedia "business career" section naming the company; a news profile. This is the backbone of confirmation.
  - *Education match:* athlete's college == founder's college (strong for college→pro athletes).
  - *Birth-year / age consistency.*
  - *Timeline plausibility:* founding date vs. athletic career window (overlap is allowed — active athletes do found companies).
  - *Geography / nationality consistency.*
  - *Co-occurrence:* both identities named on the same web page/article.
  - *Photo match* (optional, expensive — reserved for tie-breaking on high-value candidates).
- **Confidence tiers:**
  - **Confirmed** — an explicit dual-identity source exists. → publishable.
  - **Probable** — no single dual-identity source, but multiple independent corroborating signals. → publishable, flagged.
  - **Possible** — name match + weak/single signal. → manual-review queue, *not* published until adjudicated.

Output of this stage: a deduped set of **person ⇄ company** links each carrying a confidence tier and its evidence URLs.

## Stage 4 — Venture-backed verification (incl. exits)

For every resolved company, verify the funding filter against **PitchBook deal history** (Argo/PitchBook MCP), cross-checked with Crunchbase.

- **Rule:** company has ≥1 funding round in which an investor classified as institutional VC participated. Classify investor type (VC vs. angel vs. PE vs. corporate/strategic vs. accelerator) to enforce the "institutional VC" bar.
- **Exited companies are queried by company, not by current status.** PitchBook retains acquired / IPO'd / defunct companies with their historical rounds and exit events. Inclusion = "*ever* received an institutional VC round," independent of whether the company still exists or is still raising. Capture exit type (M&A / IPO / SPAC / defunct), exit date, and valuation/amount where available.
- **Edge cases (explicit rulings):**
  - *Athlete-led VC/PE funds:* the fund-management firm is **not** a qualifying company; its operating-company portfolio is in scope only if an athlete is a founder/operator there. Flag fund-founder relationships separately.
  - *CPG / apparel / restaurant / media brands:* qualify **only** if they took an institutional VC round (many do not).
  - *Holding companies / personal brands / licensing vehicles:* excluded unless they are operating companies with a VC round.
  - *Reverse mergers / SPACs:* treat the operating company's funding history as the test.

## Stage 5 — Enrichment

Fill the schema for every qualifying person/company:
- **Performance** (from U_A reference sources): sport, league(s), team(s), years active, level (pro / Olympic / both), plus **full stat lines per sport** — the complete career statistical record as published by the sport's reference DB (e.g., per-season batting/pitching lines, scoring/rebounding/assist totals, receiving/rushing yards, goals/caps, event times/marks), alongside accolades — Olympic/Paralympic medals, all-star/all-pro/awards, draft position, championships. Stat schemas are sport-specific; stored as a structured per-sport stats table keyed to `person_id`. Always sourced.
- **Background beyond athletics:** education, prior/parallel career, other ventures, notable non-athletic facts.
- **Company/funding:** sector, HQ country, founded year, role (founder / co-founder / CEO), total raised, lead & notable investors, current status & exit, and **a16z relationship** (portfolio / competitive-fund-backed / none) for sourcing value.

## Stage 6 — QA, dedup, review queue

- Run the Channel-C **recall benchmark**: report % of known athlete-founders rediscovered automatically; investigate misses to find systematic gaps.
- Adjudicate the **Possible** queue (human review).
- Final dedup pass across persons and companies; reconcile conflicting field values by source priority (PitchBook > Crunchbase > Wikidata > press) with provenance retained.
- Precision audit: sample Confirmed/Probable records and verify the dual identity holds.

## Stage 7 — Outputs

- **Master dataset** — exported as **CSV** (one file per logical table — persons, athletic/stat lines, companies, linkages — joinable on `person_id` / `company_id`). One row per person×company link in the link file (a person with multiple qualifying companies gets multiple rows). Every value carries a `source_url`. PitchBook/Argo IDs retained as columns so the CSVs can be imported into a CRM-linked store later.
- **Editorial aggregates** — leagues/sports represented, performance distribution, sector breakdown, exit outcomes, background patterns (education, etc.), time trends — built only from Confirmed + Probable records, with caveats on coverage limits.

---

## Data schema (draft)

**Person:** `person_id` (canonical), `full_name`, `aliases[]`, `birth_year`, `nationality`, `gender`, `wikidata_qid`, `linkedin_url`, `pitchbook_person_id`.

**Athletic:** `sport`, `leagues[]`, `teams[]`, `years_active`, `level` (pro/Olympic/both), `performance` (medals, awards, draft, caps, key stats — structured + free-text summary), `athletic_source_urls[]`.

**Background:** `education[]`, `prior_career`, `other_ventures[]`, `notable_facts`, `background_source_urls[]`.

**Company:** `company_id` (canonical), `company_name`, `founded_year`, `role`, `sector`, `hq_country`, `status` (active/acquired/IPO/defunct), `exit_type`, `exit_date`, `total_raised`, `lead_investors[]`, `notable_investors[]`, `a16z_relationship`, `pitchbook_company_id`, `company_source_urls[]`.

**Linkage:** `confidence_tier` (Confirmed/Probable/Possible), `evidence_sources[]`, `resolver_notes`.

## Key risks & mitigations

- **Common-name false positives** → entity-resolution confidence tiers; only Confirmed/Probable published; explicit dual-identity source preferred as backbone.
- **Recall blind spots** (founders who hide athletic past) → three independent channels + Channel-C benchmark to *measure* coverage rather than assume it.
- **Global name/transliteration noise** → Unicode folding, romanization handling, Wikidata QIDs as language-neutral keys.
- **Source access constraints** → PitchBook/Crunchbase/LinkedIn coverage and rate limits validated in a Phase-0 spike before full build.
- **"Venture-backed" ambiguity at the margins** → explicit edge-case rulings (above); investor-type classification enforced.

## Resolved decisions

1. **Performance depth** — full stat lines per sport (sport-specific stats tables keyed to `person_id`).
2. **Founding-era cutoff** — companies founded **2000 or later**.
3. **Storage** — **CSV** files (joinable tables) for now; CRM IDs retained as columns for later import.
4. **Phase 0 spike** — yes. Validate source access (PitchBook/Argo MCP depth, Crustdata/LinkedIn enrichment, Wikidata query feasibility, Sports Reference access) before committing to the full pipeline build.
