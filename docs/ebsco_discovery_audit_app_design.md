# EBSCO Discovery Service (EDS) Audit App Design

## 1) Goal
Build an **audit application** that validates EDS integration end-to-end by:
- Running searches that mimic real user behavior.
- Clicking result links and provider links.
- Verifying full-text and database access pathways.
- Producing actionable pass/fail evidence per database.

A key requirement is to **"reverse engineer" search strategies** that organically surface target databases/content in normal discovery flows.

---

## 2) What the app should test

### Core audit assertions
1. **Search quality / discoverability**
   - A query should return expected record types and sources within top N results.
2. **Link behavior**
   - "Full Text", "PDF Full Text", "HTML Full Text", and custom links should open correctly.
3. **Authentication/access path**
   - On-campus/off-campus SSO/proxy flow behaves as expected.
   - Access denied events are flagged with context.
4. **Database attribution**
   - Result metadata maps to expected database/provider.
5. **Coverage consistency**
   - Expected high-value titles/topics are discoverable via realistic queries.

### Output
- Test run report with:
  - Query used.
  - Result rank, title, source database.
  - Click path and final URL domain.
  - Access status (`SUCCESS`, `PAYWALL`, `BROKEN_LINK`, `AUTH_REQUIRED`, `METADATA_MISMATCH`).
  - Screenshot + HTML snippet on failures.

---

## 3) System architecture

## 3.1 Components
1. **Test Case Generator (Reverse Engineering Engine)**
   - Starts from known target journals/articles/databases.
   - Generates query variants (title keywords, author+year, DOI fragments, subject terms, Boolean variants).
   - Learns which query patterns most reliably surface each target.

2. **Execution Engine**
   - Two pluggable runners:
     - **Browser Runner** (Playwright/Selenium).
     - **API Runner** (EDS API calls).
   - Common normalized event model for apples-to-apples comparison.

3. **Validation Engine**
   - Rules for pass/fail on ranking, link integrity, domain allowlist, and access status.
   - Optional heuristics to detect interstitial auth pages.

4. **Evidence Collector**
   - HAR/network logs.
   - Screenshots on each click step.
   - HTML snapshots for failed states.

5. **Reporting + Dashboard**
   - Streamlit dashboard with trends by database/provider.
   - Export JSON/CSV/PDF audit packet.

## 3.2 Data model (simplified)
- `TargetAsset`: database, journal, exemplar article metadata.
- `QueryStrategy`: query string, fields, filters, expected confidence.
- `RunEvent`: timestamp, step type (search/click/redirect), metadata.
- `AssertionResult`: assertion id, status, evidence pointers.
- `AuditRun`: environment (on-campus/off-campus), versioned config.

---

## 4) Reverse engineering search workflows

## 4.1 Seed data
For each database to test, define 5–20 seed assets:
- Canonical journal/article titles.
- Known DOI(s).
- Subject heading terms.
- Author + year combos.

## 4.2 Query generation strategy
Generate and score variants:
- Exact phrase: `"journal title"`.
- Article title keywords + date.
- Author surname + key noun.
- Subject term + limiter (peer-reviewed, full text, publication date).
- Controlled expansions/synonyms.

## 4.3 Optimization loop
1. Execute candidate queries.
2. Score by:
   - Target found in top N.
   - Correct database attribution.
   - Successful full-text click-through.
3. Keep top-performing query recipes per target database.
4. Re-test periodically to detect discovery drift.

This produces a living library of **organic search recipes** that mimic patron behavior while still giving deterministic audit coverage.

---

## 5) API vs Browser: compare/contrast

## 5.1 API-based auditing
### Strengths
- Faster, stable, cheap to run at scale.
- Structured metadata is easier to parse.
- Great for regression checks and coverage baselines.

### Weaknesses
- May bypass UI layers where real issues occur (link resolver widgets, JS rendering, interstitial auth).
- Less representative of actual patron behavior.
- Some click behaviors and provider redirects are hard to reproduce faithfully.

## 5.2 Browser-based auditing
### Strengths
- Best simulation of real user journey.
- Catches UI/integration breakage the API can miss.
- Naturally validates click paths, resolver behavior, and auth redirects.

### Weaknesses
- Slower, flakier, more operationally complex.
- Sensitive to UI changes and timing.
- Requires more robust retry/wait strategies.

## 5.3 Recommendation (given your preference)
Use a **hybrid model with browser-first truth**:
- **Browser tests are authoritative** for user-experience validation.
- **API tests act as high-frequency guardrails** (metadata and index sanity checks).

Recommended split:
- Daily: lightweight API regressions + small browser smoke pack.
- Weekly/nightly: full browser audit matrix (databases × environments × auth modes).

This gives realistic validation without exploding runtime/cost.

---

## 6) Proposed test matrix

Dimensions:
- Environment: on-campus IP, off-campus proxy/SSO.
- User role: guest, authenticated patron.
- Content type: scholarly article, ebook, news, report.
- Database/provider family.
- UI pathway: basic search, advanced search, facets applied.

Minimum useful matrix:
- Top 10 strategic databases × 5 query recipes × 2 auth contexts = 100 core flows/run.

---

## 7) Reliability and anti-flake design
- Deterministic waits on semantic conditions (result count visible, expected DOM markers).
- Retry policy by failure class (network timeout vs assertion mismatch).
- Domain allowlist for expected provider redirects.
- Snapshot evidence on first failure.
- Quarantine mode for known external outages to avoid noisy failures.

---

## 8) Compliance and guardrails
- Respect EBSCO/provider terms of service and rate limits.
- Use dedicated audit accounts.
- Mask/avoid PII in logs.
- Add configurable crawl delays for external sites.

---

## 9) MVP implementation plan

### Phase 1 (2–3 weeks)
- Build browser runner for core flows (search → open result → click full text).
- Add seed targets + manual query recipes for top databases.
- Basic Streamlit report with pass/fail and screenshots.

### Phase 2 (2 weeks)
- Add reverse-engineering optimizer for query recipes.
- Add API runner and side-by-side comparator.
- Add trend charts and flaky-test classification.

### Phase 3
- CI scheduling + alerting (Slack/email).
- Historical drift detection and auto-recommendation of updated queries.

---

## 10) Practical implementation notes (Streamlit-friendly)
- Keep test execution as background jobs (Celery/RQ or subprocess queue).
- Store run artifacts in object storage (S3/Azure Blob).
- Use a small relational DB (Postgres/SQLite to start) for run metadata.
- In Streamlit, provide:
  - "Run smoke audit" button.
  - "Run full matrix" button.
  - Filterable failure explorer (database, provider, error class).

---

## 11) Suggested starting stance
Given your goal and preference, start with **browser-based auditing as the core product** and add API coverage as a complementary layer for speed and diagnostics. This balances realism with operational efficiency and gives stakeholders confidence that what patrons experience is actually working.
