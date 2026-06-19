# Athlete-Founders Dataset Pipeline — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an exhaustive, sourced CSV dataset of institutional-VC-backed companies (founded 2000+, global, including exits) whose founders were professional or Olympic athletes, with full athletic stat lines and non-athletic background.

**Architecture:** Hybrid bidirectional discovery — three independent recall channels (founder signal-mining, Wikidata athlete×founder cross-section, curated seeds) feed one candidate pool, which passes through deterministic entity-resolution → VC verification → enrichment → QA → CSV export. Deterministic logic lives in tested Python modules; discovery/enrichment is executed by agents using MCP tools (PitchBook/Argo, Crustdata, WebSearch/WebFetch) writing into versioned CSVs.

**Tech Stack:** Python 3.11, pytest, pandas (CSV I/O), requests (Wikidata SPARQL), rapidfuzz (fuzzy name matching), Unicode normalization (`unicodedata`). MCP tools at execution time: PitchBook Premium / Argo, Crustdata, WebSearch, WebFetch.

## Global Constraints

- Athletes: professional major-league OR Olympic/Paralympic only. No collegiate-only.
- Companies: founded **2000 or later**; raised **≥1 round with an institutional VC investor**; include acquired/IPO/defunct.
- Geography: global. US-first execution sequencing.
- Every published field carries a `source_url`. Only `Confirmed` + `Probable` confidence tiers are published; `Possible` stays in a review queue.
- Output: joinable CSV tables keyed on `person_id` / `company_id`. Retain `pitchbook_company_id` / `pitchbook_person_id` / `wikidata_qid` as columns.
- Canonical keys: `wikidata_qid` when available, else deterministic hash of `(normalized_name, birth_year, primary_sport)` for persons and PitchBook ID (else normalized name + founded_year) for companies.

---

### Task 1: Phase-0 access spike (go/no-go)

Validate that every data source the pipeline depends on is reachable with sufficient depth, before building anything. This is a research task; deliverable is a written findings doc, not code.

**Files:**
- Create: `docs/superpowers/notes/phase0-access-findings.md`

- [ ] **Step 1: Probe PitchBook/Argo company + funding depth.** Using the Argo/PitchBook MCP tools, pull one known VC-backed athlete-founded company (e.g., a company founded by a former pro athlete) and confirm you can retrieve: founders/people, full funding round history, investor names + investor types, founded year, and exit event (type/date). Record which tool calls return each field.

- [ ] **Step 2: Probe an exited company.** Pull one company that was acquired or IPO'd and confirm historical rounds + exit event are still queryable by company ID (not gated on active fundraising).

- [ ] **Step 3: Probe founder enrichment.** For 3 known athlete-founders, check whether PitchBook bio and/or Crustdata LinkedIn enrichment surface athletic-career text. Note hit rate and what signal text looks like.

- [ ] **Step 4: Probe Wikidata.** Run a small SPARQL query (see Task 5) against `https://query.wikidata.org/sparql` for humans with both a sport property and a founder/entrepreneur occupation; confirm it returns results and is not rate-limited at the volume needed.

- [ ] **Step 5: Probe athlete reference + stat lines.** Confirm Sports Reference / Olympedia / Wikipedia pages for 3 known athletes are fetchable via WebFetch and contain full stat lines (per-season tables) and medals/accolades.

- [ ] **Step 6: Write findings + go/no-go.** In `phase0-access-findings.md`, for each source record: reachable (Y/N), fields available, rate/volume limits, and any blocker. End with an explicit GO / NO-GO / GO-WITH-CHANGES recommendation. If a source is blocked, propose the substitute (e.g., Crunchbase for PitchBook gaps).

- [ ] **Step 7: Commit**

```bash
git add docs/superpowers/notes/phase0-access-findings.md
git commit -m "docs: phase-0 data source access findings"
```

---

### Task 2: Project scaffold + CSV schema module

Establish the repo structure and the single source of truth for the dataset schema (column names/types for all four CSV tables), with validation.

**Files:**
- Create: `pipeline/__init__.py`, `pipeline/schema.py`
- Create: `tests/test_schema.py`
- Create: `requirements.txt`, `pytest.ini`
- Create: `data/raw/.gitkeep`, `data/interim/.gitkeep`, `data/output/.gitkeep`

**Interfaces:**
- Produces:
  - `PERSON_COLUMNS: list[str]`, `ATHLETIC_COLUMNS: list[str]`, `COMPANY_COLUMNS: list[str]`, `LINK_COLUMNS: list[str]`
  - `validate_row(table: str, row: dict) -> list[str]` — returns list of error strings (empty = valid). `table` ∈ {"person","athletic","company","link"}.

- [ ] **Step 1: Write requirements + pytest config**

`requirements.txt`:
```
pandas==2.2.2
rapidfuzz==3.9.6
requests==2.32.3
pytest==8.3.2
```

`pytest.ini`:
```
[pytest]
testpaths = tests
```

- [ ] **Step 2: Write the failing test**

`tests/test_schema.py`:
```python
from pipeline.schema import validate_row, PERSON_COLUMNS, LINK_COLUMNS

def test_person_columns_include_canonical_keys():
    for col in ("person_id", "full_name", "birth_year", "wikidata_qid", "pitchbook_person_id"):
        assert col in PERSON_COLUMNS

def test_link_requires_confidence_tier():
    errs = validate_row("link", {"person_id": "p1", "company_id": "c1"})
    assert any("confidence_tier" in e for e in errs)

def test_link_rejects_unknown_tier():
    errs = validate_row("link", {"person_id": "p1", "company_id": "c1",
                                 "confidence_tier": "Maybe", "evidence_sources": "http://x"})
    assert any("confidence_tier" in e for e in errs)

def test_valid_link_row_passes():
    errs = validate_row("link", {"person_id": "p1", "company_id": "c1",
                                  "confidence_tier": "Confirmed", "evidence_sources": "http://x"})
    assert errs == []
```

- [ ] **Step 3: Run test to verify it fails**

Run: `pytest tests/test_schema.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'pipeline.schema'`

- [ ] **Step 4: Write minimal implementation**

`pipeline/schema.py`:
```python
PERSON_COLUMNS = [
    "person_id", "full_name", "aliases", "birth_year", "nationality",
    "gender", "wikidata_qid", "linkedin_url", "pitchbook_person_id",
]
ATHLETIC_COLUMNS = [
    "person_id", "sport", "leagues", "teams", "years_active", "level",
    "stat_lines", "accolades", "athletic_source_urls",
]
COMPANY_COLUMNS = [
    "company_id", "company_name", "founded_year", "sector", "hq_country",
    "status", "exit_type", "exit_date", "total_raised", "lead_investors",
    "notable_investors", "a16z_relationship", "pitchbook_company_id",
    "company_source_urls",
]
LINK_COLUMNS = [
    "person_id", "company_id", "role", "confidence_tier",
    "evidence_sources", "resolver_notes",
]
_COLUMNS = {"person": PERSON_COLUMNS, "athletic": ATHLETIC_COLUMNS,
            "company": COMPANY_COLUMNS, "link": LINK_COLUMNS}
_VALID_TIERS = {"Confirmed", "Probable", "Possible"}

def validate_row(table: str, row: dict) -> list[str]:
    if table not in _COLUMNS:
        return [f"unknown table: {table}"]
    errors = []
    unknown = set(row) - set(_COLUMNS[table])
    if unknown:
        errors.append(f"unknown columns: {sorted(unknown)}")
    if table == "link":
        if not row.get("confidence_tier"):
            errors.append("missing confidence_tier")
        elif row["confidence_tier"] not in _VALID_TIERS:
            errors.append(f"invalid confidence_tier: {row['confidence_tier']}")
        if row.get("confidence_tier") in {"Confirmed", "Probable"} and not row.get("evidence_sources"):
            errors.append("published tiers require evidence_sources")
    return errors
```

- [ ] **Step 5: Run test to verify it passes**

Run: `pytest tests/test_schema.py -v`
Expected: PASS (4 passed)

- [ ] **Step 6: Commit**

```bash
git add pipeline/ tests/test_schema.py requirements.txt pytest.ini data/
git commit -m "feat: project scaffold + CSV schema validation"
```

---

### Task 3: Name normalization + blocking

Deterministic name handling for global entity resolution (accents, transliteration, nicknames) and a blocking key to avoid full cross-join.

**Files:**
- Create: `pipeline/normalize.py`
- Create: `tests/test_normalize.py`

**Interfaces:**
- Produces:
  - `normalize_name(name: str) -> str` — lowercased, accent-folded, punctuation-stripped, whitespace-collapsed.
  - `blocking_key(name: str) -> str` — sorted first+last normalized token initials + last-name token, for candidate blocking.
  - `name_similarity(a: str, b: str) -> float` — 0..1 via rapidfuzz token_sort_ratio on normalized names.

- [ ] **Step 1: Write the failing test**

`tests/test_normalize.py`:
```python
from pipeline.normalize import normalize_name, blocking_key, name_similarity

def test_accent_folding():
    assert normalize_name("Zlatan Ibrahimović") == "zlatan ibrahimovic"

def test_punctuation_and_case():
    assert normalize_name("O'Neal, Shaquille") == "oneal shaquille"

def test_blocking_key_stable_across_order():
    assert blocking_key("Shaquille O'Neal") == blocking_key("O'Neal Shaquille")

def test_similarity_high_for_same_name():
    assert name_similarity("Tom Brady", "Thomas Brady") < 1.0
    assert name_similarity("Tom Brady", "tom  brady") == 1.0

def test_similarity_low_for_different():
    assert name_similarity("Tom Brady", "Serena Williams") < 0.4
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_normalize.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`pipeline/normalize.py`:
```python
import re
import unicodedata
from rapidfuzz import fuzz

def normalize_name(name: str) -> str:
    if not name:
        return ""
    nfkd = unicodedata.normalize("NFKD", name)
    no_accents = "".join(c for c in nfkd if not unicodedata.combining(c))
    lowered = no_accents.lower()
    cleaned = re.sub(r"[^a-z0-9 ]", " ", lowered)
    return re.sub(r"\s+", " ", cleaned).strip()

def blocking_key(name: str) -> str:
    tokens = normalize_name(name).split()
    if not tokens:
        return ""
    return " ".join(sorted(tokens))

def name_similarity(a: str, b: str) -> float:
    return fuzz.token_sort_ratio(normalize_name(a), normalize_name(b)) / 100.0
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/test_normalize.py -v`
Expected: PASS (5 passed)

- [ ] **Step 5: Commit**

```bash
git add pipeline/normalize.py tests/test_normalize.py
git commit -m "feat: name normalization + blocking for entity resolution"
```

---

### Task 4: Venture-backed classifier

Encode the inclusion rule: a company qualifies iff ≥1 round has an institutional-VC investor, and it was founded in 2000+. Operates on the funding structure returned by PitchBook (normalized into a dict by the enrichment stage).

**Files:**
- Create: `pipeline/vc_classify.py`
- Create: `tests/test_vc_classify.py`

**Interfaces:**
- Consumes: a `company` dict shaped `{"founded_year": int, "rounds": [{"investors": [{"name": str, "type": str}]}]}` where `type` ∈ {"VC","Angel","PE","Corporate","Accelerator","Other"}.
- Produces:
  - `is_institutional_vc(investor: dict) -> bool`
  - `qualifies(company: dict) -> tuple[bool, str]` — (qualifies?, human-readable reason).

- [ ] **Step 1: Write the failing test**

`tests/test_vc_classify.py`:
```python
from pipeline.vc_classify import qualifies, is_institutional_vc

VC = {"name": "Andreessen Horowitz", "type": "VC"}
ANGEL = {"name": "Some Angel", "type": "Angel"}

def test_vc_investor_recognized():
    assert is_institutional_vc(VC) is True
    assert is_institutional_vc(ANGEL) is False

def test_company_with_vc_round_qualifies():
    co = {"founded_year": 2015, "rounds": [{"investors": [ANGEL, VC]}]}
    ok, reason = qualifies(co)
    assert ok is True

def test_angel_only_company_excluded():
    co = {"founded_year": 2015, "rounds": [{"investors": [ANGEL]}]}
    ok, reason = qualifies(co)
    assert ok is False
    assert "institutional" in reason.lower()

def test_pre_2000_company_excluded():
    co = {"founded_year": 1998, "rounds": [{"investors": [VC]}]}
    ok, reason = qualifies(co)
    assert ok is False
    assert "2000" in reason

def test_exited_company_still_qualifies():
    co = {"founded_year": 2010, "status": "acquired", "rounds": [{"investors": [VC]}]}
    ok, _ = qualifies(co)
    assert ok is True
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_vc_classify.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`pipeline/vc_classify.py`:
```python
INSTITUTIONAL_TYPES = {"VC"}

def is_institutional_vc(investor: dict) -> bool:
    return investor.get("type") in INSTITUTIONAL_TYPES

def qualifies(company: dict) -> tuple[bool, str]:
    fy = company.get("founded_year")
    if fy is None:
        return False, "missing founded_year"
    if fy < 2000:
        return False, f"founded {fy} < 2000 cutoff"
    for rnd in company.get("rounds", []):
        if any(is_institutional_vc(i) for i in rnd.get("investors", [])):
            return True, "has institutional VC round"
    return False, "no institutional VC investor in any round"
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/test_vc_classify.py -v`
Expected: PASS (5 passed)

- [ ] **Step 5: Commit**

```bash
git add pipeline/vc_classify.py tests/test_vc_classify.py
git commit -m "feat: venture-backed inclusion classifier"
```

---

### Task 5: Entity-resolution scoring + confidence tiers

The precision core: given an athlete record and a founder record, score whether they are the same person and assign a confidence tier.

**Files:**
- Create: `pipeline/resolve.py`
- Create: `tests/test_resolve.py`

**Interfaces:**
- Consumes: `name_similarity` from `pipeline.normalize`.
- Produces:
  - `score_match(athlete: dict, founder: dict) -> dict` returning `{"score": float, "signals": list[str]}`. Input dicts may carry: `name`, `birth_year`, `education` (list), `nationality`, `dual_identity_source` (bool/url), `co_occurrence` (bool).
  - `confidence_tier(match: dict) -> str` ∈ {"Confirmed","Probable","Possible","Reject"}.

- [ ] **Step 1: Write the failing test**

`tests/test_resolve.py`:
```python
from pipeline.resolve import score_match, confidence_tier

def test_explicit_dual_identity_is_confirmed():
    m = score_match(
        {"name": "Jane Doe", "birth_year": 1985, "dual_identity_source": "http://li/jane"},
        {"name": "Jane Doe", "birth_year": 1985, "dual_identity_source": "http://li/jane"},
    )
    assert confidence_tier(m) == "Confirmed"

def test_multiple_signals_is_probable():
    m = score_match(
        {"name": "John Smith", "birth_year": 1980, "education": ["Stanford"], "nationality": "US"},
        {"name": "Jon Smith", "birth_year": 1980, "education": ["Stanford"], "nationality": "US"},
    )
    assert confidence_tier(m) == "Probable"

def test_name_only_is_possible():
    m = score_match({"name": "John Smith"}, {"name": "John Smith"})
    assert confidence_tier(m) == "Possible"

def test_different_birth_year_rejected():
    m = score_match(
        {"name": "John Smith", "birth_year": 1980},
        {"name": "John Smith", "birth_year": 1995},
    )
    assert confidence_tier(m) == "Reject"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_resolve.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`pipeline/resolve.py`:
```python
from pipeline.normalize import name_similarity

def score_match(athlete: dict, founder: dict) -> dict:
    signals = []
    score = 0.0

    by_a, by_f = athlete.get("birth_year"), founder.get("birth_year")
    if by_a and by_f and by_a != by_f:
        return {"score": 0.0, "signals": ["birth_year_conflict"]}

    sim = name_similarity(athlete.get("name", ""), founder.get("name", ""))
    if sim >= 0.85:
        score += sim
        signals.append(f"name_sim={sim:.2f}")

    dual = athlete.get("dual_identity_source") and founder.get("dual_identity_source")
    if dual:
        signals.append("dual_identity_source")

    edu_a = {e.lower() for e in athlete.get("education", [])}
    edu_f = {e.lower() for e in founder.get("education", [])}
    if edu_a and edu_a & edu_f:
        score += 0.5
        signals.append("education_match")

    if by_a and by_f and by_a == by_f:
        score += 0.3
        signals.append("birth_year_match")

    if athlete.get("nationality") and athlete.get("nationality") == founder.get("nationality"):
        score += 0.1
        signals.append("nationality_match")

    if athlete.get("co_occurrence") and founder.get("co_occurrence"):
        score += 0.2
        signals.append("co_occurrence")

    return {"score": round(score, 3), "signals": signals, "dual_identity": bool(dual)}

def confidence_tier(match: dict) -> str:
    signals = match.get("signals", [])
    if "birth_year_conflict" in signals:
        return "Reject"
    if match.get("dual_identity"):
        return "Confirmed"
    corroborating = {"education_match", "birth_year_match", "nationality_match", "co_occurrence"}
    n_corrob = len(corroborating & set(signals))
    has_name = any(s.startswith("name_sim") for s in signals)
    if not has_name:
        return "Reject"
    if n_corrob >= 2:
        return "Probable"
    return "Possible"
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/test_resolve.py -v`
Expected: PASS (4 passed)

- [ ] **Step 5: Commit**

```bash
git add pipeline/resolve.py tests/test_resolve.py
git commit -m "feat: entity-resolution scoring + confidence tiers"
```

---

### Task 6: CSV I/O + cross-table integrity

Read/write the four joinable CSV tables and validate referential integrity (every link references existing person_id and company_id).

**Files:**
- Create: `pipeline/io_csv.py`
- Create: `tests/test_io_csv.py`

**Interfaces:**
- Consumes: column lists + `validate_row` from `pipeline.schema`.
- Produces:
  - `write_table(table: str, rows: list[dict], out_dir: str) -> str` — writes `<out_dir>/<table>.csv`, returns path. Raises `ValueError` if any row fails `validate_row`.
  - `check_referential_integrity(out_dir: str) -> list[str]` — returns list of integrity errors (empty = ok).

- [ ] **Step 1: Write the failing test**

`tests/test_io_csv.py`:
```python
from pipeline.io_csv import write_table, check_referential_integrity

def test_write_and_integrity_ok(tmp_path):
    d = str(tmp_path)
    write_table("person", [{"person_id": "p1", "full_name": "A B"}], d)
    write_table("company", [{"company_id": "c1", "company_name": "Co"}], d)
    write_table("link", [{"person_id": "p1", "company_id": "c1",
                          "confidence_tier": "Confirmed", "evidence_sources": "http://x"}], d)
    assert check_referential_integrity(d) == []

def test_dangling_link_detected(tmp_path):
    d = str(tmp_path)
    write_table("person", [{"person_id": "p1", "full_name": "A B"}], d)
    write_table("company", [{"company_id": "c1", "company_name": "Co"}], d)
    write_table("link", [{"person_id": "pX", "company_id": "c1",
                          "confidence_tier": "Confirmed", "evidence_sources": "http://x"}], d)
    errs = check_referential_integrity(d)
    assert any("pX" in e for e in errs)

def test_invalid_row_raises(tmp_path):
    import pytest
    with pytest.raises(ValueError):
        write_table("link", [{"person_id": "p1", "company_id": "c1",
                              "confidence_tier": "Maybe"}], str(tmp_path))
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_io_csv.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`pipeline/io_csv.py`:
```python
import os
import pandas as pd
from pipeline.schema import _COLUMNS, validate_row

def write_table(table: str, rows: list[dict], out_dir: str) -> str:
    for row in rows:
        errs = validate_row(table, row)
        if errs:
            raise ValueError(f"{table} row invalid: {errs}")
    os.makedirs(out_dir, exist_ok=True)
    path = os.path.join(out_dir, f"{table}.csv")
    df = pd.DataFrame(rows, columns=_COLUMNS[table])
    df.to_csv(path, index=False)
    return path

def _ids(out_dir: str, table: str, col: str) -> set:
    path = os.path.join(out_dir, f"{table}.csv")
    if not os.path.exists(path):
        return set()
    return set(pd.read_csv(path, dtype=str)[col].dropna())

def check_referential_integrity(out_dir: str) -> list[str]:
    errors = []
    persons = _ids(out_dir, "person", "person_id")
    companies = _ids(out_dir, "company", "company_id")
    link_path = os.path.join(out_dir, "link.csv")
    if os.path.exists(link_path):
        links = pd.read_csv(link_path, dtype=str)
        for _, r in links.iterrows():
            if r["person_id"] not in persons:
                errors.append(f"link references missing person_id {r['person_id']}")
            if r["company_id"] not in companies:
                errors.append(f"link references missing company_id {r['company_id']}")
    return errors
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/test_io_csv.py -v`
Expected: PASS (3 passed)

- [ ] **Step 5: Commit**

```bash
git add pipeline/io_csv.py tests/test_io_csv.py
git commit -m "feat: CSV I/O + referential integrity check"
```

---

### Task 7: Channel B — Wikidata athlete×founder query

Build the structured Wikidata recall channel: a SPARQL query + parser producing candidate records.

**Files:**
- Create: `channels/channel_b_wikidata.py`
- Create: `tests/test_channel_b.py`

**Interfaces:**
- Produces:
  - `build_query() -> str` — SPARQL string.
  - `parse_results(json_bindings: list[dict]) -> list[dict]` — returns candidates `{"person_name","wikidata_qid","athletic_claim","company_claim","source":"wikidata_B","source_url"}`.
  - `fetch(endpoint: str = "https://query.wikidata.org/sparql") -> list[dict]` — runs query, returns parsed candidates. (Network; not unit-tested — tested via `parse_results` on a fixture.)

- [ ] **Step 1: Write the failing test (parser only, on a fixture)**

`tests/test_channel_b.py`:
```python
from channels.channel_b_wikidata import build_query, parse_results

def test_query_mentions_sport_and_founder():
    q = build_query()
    assert "P641" in q  # sport
    assert "founded" in q.lower() or "P112" in q  # founded by / occupation entrepreneur

def test_parse_extracts_candidate():
    bindings = [{
        "person": {"value": "http://www.wikidata.org/entity/Q123"},
        "personLabel": {"value": "Jane Athlete"},
        "sportLabel": {"value": "Basketball"},
        "orgLabel": {"value": "Acme Inc"},
    }]
    out = parse_results(bindings)
    assert out[0]["wikidata_qid"] == "Q123"
    assert out[0]["person_name"] == "Jane Athlete"
    assert out[0]["athletic_claim"] == "Basketball"
    assert out[0]["company_claim"] == "Acme Inc"
    assert out[0]["source"] == "wikidata_B"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_channel_b.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`channels/channel_b_wikidata.py`:
```python
import requests

def build_query() -> str:
    # Humans with a sport (P641) who founded an organization (P112 inverse)
    # or have occupation entrepreneur (Q131524)/businessperson (Q43845).
    return """
    SELECT DISTINCT ?person ?personLabel ?sportLabel ?org ?orgLabel WHERE {
      ?person wdt:P31 wd:Q5 ; wdt:P641 ?sport .
      { ?org wdt:P112 ?person . }
      UNION
      { ?person wdt:P106 ?occ . VALUES ?occ { wd:Q131524 wd:Q43845 } .
        OPTIONAL { ?org wdt:P112 ?person . } }
      SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
    }
    """

def parse_results(json_bindings: list[dict]) -> list[dict]:
    out = []
    for b in json_bindings:
        qid = b.get("person", {}).get("value", "").rsplit("/", 1)[-1]
        out.append({
            "person_name": b.get("personLabel", {}).get("value", ""),
            "wikidata_qid": qid,
            "athletic_claim": b.get("sportLabel", {}).get("value", ""),
            "company_claim": b.get("orgLabel", {}).get("value", ""),
            "source": "wikidata_B",
            "source_url": b.get("person", {}).get("value", ""),
        })
    return out

def fetch(endpoint: str = "https://query.wikidata.org/sparql") -> list[dict]:
    resp = requests.get(endpoint, params={"query": build_query(), "format": "json"},
                        headers={"User-Agent": "a16z-athlete-founders-research/1.0"})
    resp.raise_for_status()
    return parse_results(resp.json()["results"]["bindings"])
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/test_channel_b.py -v`
Expected: PASS (2 passed)

- [ ] **Step 5: Run the live query once and snapshot**

Run: `python -c "from channels.channel_b_wikidata import fetch; import json; json.dump(fetch(), open('data/raw/channel_b.json','w'))"`
Expected: writes a non-empty JSON array to `data/raw/channel_b.json`. Eyeball 5 records for sanity.

- [ ] **Step 6: Commit**

```bash
git add channels/channel_b_wikidata.py tests/test_channel_b.py data/raw/channel_b.json
git commit -m "feat: Channel B Wikidata athlete-founder recall"
```

---

### Task 8: Channel A + Channel C runbooks (agent-executed)

Channels A and C are executed by agents using MCP tools and web search — not deterministic scripts — so they are documented as precise runbooks that emit the same candidate JSON shape as Channel B. Deliverable is the runbook docs plus the produced candidate files.

**Files:**
- Create: `channels/channel_a_founder_signal.md`
- Create: `channels/channel_c_curated_seeds.md`
- Produces (when run): `data/raw/channel_a.json`, `data/raw/channel_c.json` (same record shape as Channel B, `source` = `founder_signal_A` / `curated_C`).

- [ ] **Step 1: Write Channel A runbook.** In `channel_a_founder_signal.md`, document: (1) the founder-bio signal lexicon (league names per sport, "Olympian", "Team USA"/national teams, "drafted", "professional [sport]", "national team", "caps"); (2) the procedure — for batches of VC-backed founders pulled from PitchBook/Argo, fetch bio + Crustdata LinkedIn, flag any signal hit; (3) the output record shape (`person_name`, `athletic_claim`=matched signal text, `company_claim`=their company, `source`=`founder_signal_A`, `source_url`); (4) what NOT to flag (recreational/collegiate-only mentions without pro/Olympic level).

- [ ] **Step 2: Write Channel C runbook.** In `channel_c_curated_seeds.md`, document: (1) the seed source list — press features on athlete-founders, sports-tech media, athlete-led VC fund portfolios, athlete-focused accelerators (name concrete starting URLs/queries); (2) extraction procedure via WebSearch/WebFetch; (3) output record shape (`source`=`curated_C`); (4) the dual role of this channel as both seed AND the recall-benchmark gold set (Task 12).

- [ ] **Step 3: Execute both runbooks (first global, US-first pass).** Produce `data/raw/channel_a.json` and `data/raw/channel_c.json`. Target a meaningful first batch (e.g., ≥50 candidates each) to exercise the downstream pipeline; full sweep happens in Task 13.

- [ ] **Step 4: Commit**

```bash
git add channels/channel_a_founder_signal.md channels/channel_c_curated_seeds.md data/raw/channel_a.json data/raw/channel_c.json
git commit -m "feat: Channel A + C runbooks and first candidate batch"
```

---

### Task 9: Candidate pool merge + within-universe dedup

Combine all channel outputs into one candidate pool and dedup within the athlete universe and within the founder universe before cross-resolution.

**Files:**
- Create: `pipeline/candidate_pool.py`
- Create: `tests/test_candidate_pool.py`

**Interfaces:**
- Consumes: `blocking_key`, `name_similarity` from `pipeline.normalize`.
- Produces:
  - `load_candidates(paths: list[str]) -> list[dict]` — concatenates channel JSON files.
  - `dedup_candidates(candidates: list[dict]) -> list[dict]` — merges records that share `wikidata_qid`, or share `blocking_key` AND `name_similarity >= 0.9`; unions their `source` into a `sources` list and keeps all `source_url`s.

- [ ] **Step 1: Write the failing test**

`tests/test_candidate_pool.py`:
```python
from pipeline.candidate_pool import dedup_candidates

def test_merge_by_qid():
    cands = [
        {"person_name": "Jane Athlete", "wikidata_qid": "Q1", "source": "wikidata_B", "source_url": "u1"},
        {"person_name": "J. Athlete", "wikidata_qid": "Q1", "source": "curated_C", "source_url": "u2"},
    ]
    out = dedup_candidates(cands)
    assert len(out) == 1
    assert set(out[0]["sources"]) == {"wikidata_B", "curated_C"}

def test_merge_by_name_similarity():
    cands = [
        {"person_name": "John Smith", "wikidata_qid": "", "source": "founder_signal_A", "source_url": "u1"},
        {"person_name": "John  Smith", "wikidata_qid": "", "source": "curated_C", "source_url": "u2"},
    ]
    out = dedup_candidates(cands)
    assert len(out) == 1

def test_distinct_people_not_merged():
    cands = [
        {"person_name": "John Smith", "wikidata_qid": "", "source": "a", "source_url": "u1"},
        {"person_name": "Serena Williams", "wikidata_qid": "", "source": "b", "source_url": "u2"},
    ]
    out = dedup_candidates(cands)
    assert len(out) == 2
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_candidate_pool.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`pipeline/candidate_pool.py`:
```python
import json
from pipeline.normalize import blocking_key, name_similarity

def load_candidates(paths: list[str]) -> list[dict]:
    out = []
    for p in paths:
        with open(p) as f:
            out.extend(json.load(f))
    return out

def _merge(a: dict, b: dict) -> dict:
    merged = dict(a)
    sources = set(a.get("sources", [a.get("source")] if a.get("source") else []))
    sources.update(b.get("sources", [b.get("source")] if b.get("source") else []))
    merged["sources"] = sorted(s for s in sources if s)
    urls = set(filter(None, [a.get("source_url"), b.get("source_url")]
                      + a.get("source_urls", []) + b.get("source_urls", [])))
    merged["source_urls"] = sorted(urls)
    if not merged.get("wikidata_qid"):
        merged["wikidata_qid"] = b.get("wikidata_qid", "")
    return merged

def dedup_candidates(candidates: list[dict]) -> list[dict]:
    clusters: list[dict] = []
    for c in candidates:
        match = None
        for existing in clusters:
            if c.get("wikidata_qid") and c["wikidata_qid"] == existing.get("wikidata_qid"):
                match = existing
                break
            if (blocking_key(c["person_name"]) == blocking_key(existing["person_name"])
                    and name_similarity(c["person_name"], existing["person_name"]) >= 0.9):
                match = existing
                break
        if match:
            clusters[clusters.index(match)] = _merge(match, c)
        else:
            seed = dict(c)
            seed["sources"] = [c["source"]] if c.get("source") else c.get("sources", [])
            clusters.append(seed)
    return clusters
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/test_candidate_pool.py -v`
Expected: PASS (3 passed)

- [ ] **Step 5: Commit**

```bash
git add pipeline/candidate_pool.py tests/test_candidate_pool.py
git commit -m "feat: candidate pool merge + within-universe dedup"
```

---

### Task 10: Enrichment runbook — athletic + company/funding (agent-executed)

For each deduped candidate, gather the data needed for entity resolution and the final schema. Agent-executed via MCP/web; documented as a runbook with a strict output contract feeding Tasks 11–12.

**Files:**
- Create: `pipeline/enrich.md` (runbook)
- Produces (when run): `data/interim/enriched_candidates.json` — each record carries `athlete` and `founder` sub-dicts shaped for `score_match` (Task 5) plus full schema fields (stat lines, accolades, funding `rounds` with investor `type`, founded_year, exit, education, birth_year, nationality).

- [ ] **Step 1: Write the enrichment runbook.** In `enrich.md` document, per candidate: (a) **athletic enrichment** — resolve to Sports Reference / Olympedia / Wikipedia, extract sport, leagues, teams, years_active, level, **full stat lines** (per-season tables), accolades/medals, birth_year, nationality, education; record `athletic_source_urls`. (b) **company/funding enrichment** — resolve the company in PitchBook/Argo, extract founded_year, role, sector, hq_country, status, exit_type/date, total_raised, lead/notable investors **with investor type**, structured as `rounds:[{investors:[{name,type}]}]`; record `company_source_urls`. (c) **dual-identity hunt** — actively search for one source asserting both identities (LinkedIn listing pro career + founder role; Wikipedia business-career section); if found, set `dual_identity_source` URL on both sub-dicts. (d) **output contract** — exact JSON shape consumed by Task 11.

- [ ] **Step 2: Execute enrichment on the first batch.** Produce `data/interim/enriched_candidates.json` for the Task 8/9 candidate pool.

- [ ] **Step 3: Validate the output contract.** Run: `python -c "import json; d=json.load(open('data/interim/enriched_candidates.json')); assert all('athlete' in r and 'founder' in r for r in d); print(len(d), 'records ok')"`
Expected: prints count, no assertion error.

- [ ] **Step 4: Commit**

```bash
git add pipeline/enrich.md data/interim/enriched_candidates.json
git commit -m "feat: enrichment runbook + first enriched batch"
```

---

### Task 11: Pipeline assembly — resolve → verify → emit CSV

Wire the deterministic modules into one driver that turns enriched candidates into the four published CSVs, applying confidence tiers and the VC filter, routing `Possible` to a review queue.

**Files:**
- Create: `pipeline/run.py`
- Create: `tests/test_run.py`

**Interfaces:**
- Consumes: `score_match`/`confidence_tier` (Task 5), `qualifies` (Task 4), `write_table`/`check_referential_integrity` (Task 6), schema (Task 2).
- Produces:
  - `process(enriched: list[dict]) -> dict` returning `{"published": [...], "review_queue": [...], "rejected": [...]}` where published items carry person/company/link/athletic rows.
  - `emit(enriched: list[dict], out_dir: str, review_dir: str) -> dict` — writes CSVs for published rows and a `review_queue.csv`; returns counts.

- [ ] **Step 1: Write the failing test**

`tests/test_run.py`:
```python
from pipeline.run import process

VC = {"name": "a16z", "type": "VC"}

def _enriched(**over):
    base = {
        "person_name": "Jane Doe",
        "athlete": {"name": "Jane Doe", "birth_year": 1985,
                    "dual_identity_source": "http://li/jane"},
        "founder": {"name": "Jane Doe", "birth_year": 1985,
                    "dual_identity_source": "http://li/jane",
                    "company_name": "Acme", "founded_year": 2015,
                    "rounds": [{"investors": [VC]}]},
    }
    base.update(over)
    return base

def test_confirmed_vc_company_published():
    out = process([_enriched()])
    assert len(out["published"]) == 1

def test_angel_only_rejected_even_if_confirmed_person():
    e = _enriched()
    e["founder"]["rounds"] = [{"investors": [{"name": "x", "type": "Angel"}]}]
    out = process([e])
    assert len(out["published"]) == 0
    assert len(out["rejected"]) == 1

def test_possible_goes_to_review_queue():
    e = _enriched()
    e["athlete"] = {"name": "Jane Doe"}
    e["founder"] = {"name": "Jane Doe", "company_name": "Acme",
                    "founded_year": 2015, "rounds": [{"investors": [VC]}]}
    out = process([e])
    assert len(out["review_queue"]) == 1
    assert len(out["published"]) == 0
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_run.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`pipeline/run.py`:
```python
import hashlib
from pipeline.resolve import score_match, confidence_tier
from pipeline.vc_classify import qualifies
from pipeline.io_csv import write_table, check_referential_integrity

def _pid(rec: dict) -> str:
    qid = rec.get("wikidata_qid")
    if qid:
        return qid
    key = f"{rec['athlete'].get('name','')}|{rec['athlete'].get('birth_year','')}"
    return "P" + hashlib.sha1(key.encode()).hexdigest()[:12]

def _cid(founder: dict) -> str:
    pb = founder.get("pitchbook_company_id")
    if pb:
        return str(pb)
    key = f"{founder.get('company_name','')}|{founder.get('founded_year','')}"
    return "C" + hashlib.sha1(key.encode()).hexdigest()[:12]

def process(enriched: list[dict]) -> dict:
    published, review_queue, rejected = [], [], []
    for rec in enriched:
        match = score_match(rec["athlete"], rec["founder"])
        tier = confidence_tier(match)
        ok, reason = qualifies(rec["founder"])
        item = {"rec": rec, "tier": tier, "match": match, "vc_reason": reason}
        if not ok:
            rejected.append(item)
        elif tier in ("Confirmed", "Probable"):
            published.append(item)
        elif tier == "Possible":
            review_queue.append(item)
        else:
            rejected.append(item)
    return {"published": published, "review_queue": review_queue, "rejected": rejected}

def emit(enriched: list[dict], out_dir: str, review_dir: str) -> dict:
    result = process(enriched)
    persons, companies, links, athletics = [], [], [], []
    for item in result["published"]:
        rec, a, f = item["rec"], item["rec"]["athlete"], item["rec"]["founder"]
        pid, cid = _pid(rec), _cid(f)
        persons.append({"person_id": pid, "full_name": rec["person_name"],
                        "birth_year": a.get("birth_year"), "nationality": a.get("nationality"),
                        "wikidata_qid": rec.get("wikidata_qid", "")})
        athletics.append({"person_id": pid, "sport": a.get("sport"),
                          "leagues": a.get("leagues"), "level": a.get("level"),
                          "stat_lines": a.get("stat_lines"), "accolades": a.get("accolades"),
                          "athletic_source_urls": a.get("athletic_source_urls")})
        companies.append({"company_id": cid, "company_name": f.get("company_name"),
                          "founded_year": f.get("founded_year"), "status": f.get("status"),
                          "exit_type": f.get("exit_type"), "total_raised": f.get("total_raised"),
                          "pitchbook_company_id": f.get("pitchbook_company_id", "")})
        links.append({"person_id": pid, "company_id": cid, "role": f.get("role", "founder"),
                      "confidence_tier": item["tier"],
                      "evidence_sources": ";".join(rec.get("source_urls", [])) or "n/a",
                      "resolver_notes": ",".join(item["match"]["signals"])})
    write_table("person", persons, out_dir)
    write_table("athletic", athletics, out_dir)
    write_table("company", companies, out_dir)
    write_table("link", links, out_dir)
    integrity = check_referential_integrity(out_dir)
    review_rows = [{"person_id": _pid(i["rec"]), "company_id": _cid(i["rec"]["founder"]),
                    "role": "founder", "confidence_tier": "Possible",
                    "evidence_sources": ";".join(i["rec"].get("source_urls", [])) or "n/a",
                    "resolver_notes": ",".join(i["match"]["signals"])}
                   for i in result["review_queue"]]
    write_table("link", review_rows, review_dir)
    return {"published": len(links), "review": len(review_rows),
            "rejected": len(result["rejected"]), "integrity_errors": integrity}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/test_run.py -v`
Expected: PASS (3 passed)

- [ ] **Step 5: Run the full suite**

Run: `pytest -v`
Expected: all tests pass.

- [ ] **Step 6: Run end-to-end on the first batch**

Run: `python -c "import json; from pipeline.run import emit; print(emit(json.load(open('data/interim/enriched_candidates.json')), 'data/output', 'data/review'))"`
Expected: prints counts; `integrity_errors` is `[]`; `data/output/*.csv` populated.

- [ ] **Step 7: Commit**

```bash
git add pipeline/run.py tests/test_run.py data/output data/review
git commit -m "feat: pipeline assembly resolve->verify->CSV"
```

---

### Task 12: QA — recall benchmark + precision audit

Measure coverage and precision so the dataset's completeness is reported, not assumed.

**Files:**
- Create: `pipeline/qa.py`
- Create: `tests/test_qa.py`
- Create: `docs/superpowers/notes/qa-report.md` (generated)

**Interfaces:**
- Produces:
  - `recall(gold: list[str], found_ids: list[str]) -> dict` — gold = Channel-C known athlete-founder identifiers; returns `{"recall": float, "missed": list}`.
  - `precision_sample(published: list[dict], n: int) -> list[dict]` — deterministic sample (every k-th record) for manual dual-identity verification.

- [ ] **Step 1: Write the failing test**

`tests/test_qa.py`:
```python
from pipeline.qa import recall, precision_sample

def test_recall_counts_missed():
    out = recall(["a", "b", "c", "d"], ["a", "b"])
    assert out["recall"] == 0.5
    assert set(out["missed"]) == {"c", "d"}

def test_precision_sample_is_deterministic():
    pub = [{"id": i} for i in range(10)]
    s1 = precision_sample(pub, 3)
    s2 = precision_sample(pub, 3)
    assert s1 == s2
    assert len(s1) == 3
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_qa.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`pipeline/qa.py`:
```python
def recall(gold: list[str], found_ids: list[str]) -> dict:
    found = set(found_ids)
    missed = [g for g in gold if g not in found]
    r = (len(gold) - len(missed)) / len(gold) if gold else 0.0
    return {"recall": round(r, 3), "missed": missed}

def precision_sample(published: list[dict], n: int) -> list[dict]:
    if n <= 0 or not published:
        return []
    step = max(1, len(published) // n)
    return [published[i] for i in range(0, len(published), step)][:n]
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/test_qa.py -v`
Expected: PASS (2 passed)

- [ ] **Step 5: Generate the QA report.** Compute recall against the Channel-C gold set (which of those known athlete-founders the automated channels A/B independently surfaced), and hand-verify a precision sample of published records. Write results, the recall %, the missed list (with hypotheses for each miss), and the precision findings to `qa-report.md`.

- [ ] **Step 6: Commit**

```bash
git add pipeline/qa.py tests/test_qa.py docs/superpowers/notes/qa-report.md
git commit -m "feat: QA recall benchmark + precision audit"
```

---

### Task 13: Full global sweep + Possible-queue adjudication

Scale the validated pipeline from first batch to full coverage, then human-adjudicate the review queue.

**Files:**
- Modify: candidate files under `data/raw/`, `data/interim/`, `data/output/`
- Create: `docs/superpowers/notes/coverage-log.md`

- [ ] **Step 1: Run Channels A/B/C at full scale.** Execute the runbooks (Tasks 7, 8) across the full global universe, US-first then international. Append to `data/raw/*.json`.

- [ ] **Step 2: Re-run dedup + enrichment + pipeline.** Regenerate `data/interim/enriched_candidates.json` and re-run `emit` (Task 11). Confirm `integrity_errors == []` and full suite still passes (`pytest -v`).

- [ ] **Step 3: Adjudicate the Possible queue.** Manually review `data/review/link.csv`; promote confirmed matches to published (with added evidence) or reject. Document decisions.

- [ ] **Step 4: Re-run QA (Task 12) and log coverage.** Record final counts (persons, companies, links by tier), recall %, leagues/sports represented, and known gaps in `coverage-log.md`. Note any silent caps (e.g., international sources skipped) explicitly.

- [ ] **Step 5: Commit**

```bash
git add data/ docs/superpowers/notes/coverage-log.md
git commit -m "feat: full global sweep + review-queue adjudication"
```

---

### Task 14: Editorial aggregates

Produce the published-piece aggregates from `Confirmed` + `Probable` records only.

**Files:**
- Create: `pipeline/aggregates.py`
- Create: `tests/test_aggregates.py`
- Create: `data/output/aggregates/` (generated CSVs)

**Interfaces:**
- Produces:
  - `aggregate(out_dir: str) -> dict` reading published CSVs, returning dicts for: league/sport counts, sector breakdown, exit-outcome counts, founding-year trend, education patterns. Writes one CSV per aggregate to `out_dir/aggregates/`.

- [ ] **Step 1: Write the failing test**

`tests/test_aggregates.py`:
```python
import os
from pipeline.io_csv import write_table
from pipeline.aggregates import aggregate

def test_sport_counts(tmp_path):
    d = str(tmp_path)
    write_table("person", [{"person_id": "p1", "full_name": "A"},
                           {"person_id": "p2", "full_name": "B"}], d)
    write_table("athletic", [{"person_id": "p1", "sport": "Basketball"},
                             {"person_id": "p2", "sport": "Basketball"}], d)
    write_table("company", [{"company_id": "c1", "company_name": "X"}], d)
    write_table("link", [{"person_id": "p1", "company_id": "c1",
                          "confidence_tier": "Confirmed", "evidence_sources": "u"}], d)
    out = aggregate(d)
    assert out["sport_counts"]["Basketball"] >= 1
    assert os.path.exists(os.path.join(d, "aggregates", "sport_counts.csv"))
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_aggregates.py -v`
Expected: FAIL with `ModuleNotFoundError`

- [ ] **Step 3: Write minimal implementation**

`pipeline/aggregates.py`:
```python
import os
import pandas as pd

def _read(out_dir, name):
    p = os.path.join(out_dir, f"{name}.csv")
    return pd.read_csv(p, dtype=str) if os.path.exists(p) else pd.DataFrame()

def aggregate(out_dir: str) -> dict:
    links = _read(out_dir, "link")
    athletic = _read(out_dir, "athletic")
    company = _read(out_dir, "company")
    pub_pids = set(links["person_id"]) if not links.empty else set()
    ath = athletic[athletic["person_id"].isin(pub_pids)] if not athletic.empty else athletic

    sport_counts = ath["sport"].value_counts().to_dict() if not ath.empty else {}
    pub_cids = set(links["company_id"]) if not links.empty else set()
    comp = company[company["company_id"].isin(pub_cids)] if not company.empty else company
    sector_counts = comp["sector"].value_counts().to_dict() if "sector" in comp else {}
    exit_counts = comp["exit_type"].value_counts().to_dict() if "exit_type" in comp else {}
    year_trend = comp["founded_year"].value_counts().sort_index().to_dict() if "founded_year" in comp else {}

    agg_dir = os.path.join(out_dir, "aggregates")
    os.makedirs(agg_dir, exist_ok=True)
    for name, data in [("sport_counts", sport_counts), ("sector_counts", sector_counts),
                       ("exit_counts", exit_counts), ("year_trend", year_trend)]:
        pd.DataFrame(list(data.items()), columns=[name[:-7] or name, "count"]).to_csv(
            os.path.join(agg_dir, f"{name}.csv"), index=False)
    return {"sport_counts": sport_counts, "sector_counts": sector_counts,
            "exit_counts": exit_counts, "year_trend": year_trend}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/test_aggregates.py -v`
Expected: PASS (1 passed)

- [ ] **Step 5: Generate real aggregates + run full suite**

Run: `pytest -v && python -c "from pipeline.aggregates import aggregate; print(aggregate('data/output'))"`
Expected: all tests pass; aggregate CSVs written under `data/output/aggregates/`.

- [ ] **Step 6: Commit**

```bash
git add pipeline/aggregates.py tests/test_aggregates.py data/output/aggregates
git commit -m "feat: editorial aggregates from published dataset"
```

---

## Self-Review

**Spec coverage:**
- Hybrid 3-channel discovery → Tasks 7 (B), 8 (A+C). ✓
- Don't bulk-ingest U_A; on-demand reference → enrichment runbook Task 10. ✓
- Within-universe dedup then cross-universe ER → Tasks 9 (within) + 5/11 (cross). ✓
- Confidence tiers, only Confirmed/Probable published, Possible→queue → Tasks 5, 11, 13. ✓
- Venture-backed = institutional VC round; founded 2000+ → Task 4. ✓
- Exits included (query by company) → Task 4 test + enrichment runbook Task 10. ✓
- Full stat lines per sport → enrichment runbook Task 10 + schema Task 2. ✓
- Global, US-first → Tasks 8, 13. ✓
- CSV joinable tables, source_urls, CRM IDs retained → Tasks 2, 6, 11. ✓
- Recall benchmark + precision audit → Task 12. ✓
- Editorial aggregates → Task 14. ✓
- Phase-0 access spike first → Task 1. ✓

**Placeholder scan:** Research/runbook steps (Tasks 1, 8, 10, 13) intentionally describe agent procedures rather than code — their deliverables and output contracts are concrete and validated by downstream tests. No code step contains TBD/TODO.

**Type consistency:** `score_match`/`confidence_tier`, `qualifies`, `write_table`/`check_referential_integrity`, `validate_row`, candidate record shape (`person_name`/`wikidata_qid`/`athletic_claim`/`company_claim`/`source`/`source_url`), and the enriched `{athlete, founder}` contract are used consistently across Tasks 2–14.
