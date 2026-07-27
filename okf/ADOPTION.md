# OKF Adoption — 30-Day Pilot Rollout

**Intended location:** `okf/ADOPTION.md`
**Prerequisite reading:** `okf/profiles/github-mcp/ENTERPRISE_BASELINE.md` — this document is the "how to roll it out," that one is the "what it is and why."

**Goal:** stand up a governed, reliable context layer for one pilot application, prove the value concretely to engineering teams, and produce a repeatable pattern for expanding to the priority business functions identified earlier — without over-building before the first cycle has proven anything out.

---

## Week 1 — Repo structure, templates, guard scripts, and sign-off

**Build:**
- `okf/` starter bundle in the pilot repo: `index.md`, `log.md`, `business-functions/index.md`, `concepts/index.md`, `references/index.md` (see `ENTERPRISE_BASELINE.md` §5 — `references/` is required from day one, not added later).
- `AGENTS.md` at repo root with the OKF-maintenance conventions (`ENTERPRISE_BASELINE.md` §9).
- `okf-request` Issue template (`ENTERPRISE_BASELINE.md` §7.1), including the optional supporting-documentation attachment field.
- `okf/tools/*`: `structural_pass.py`, `validate_okf.py`, `regenerate_index.py`, `shrinkage_guard.py`.
- `okf/profiles/github-mcp/policies/governance-checklist.md` and `roles-and-slas.md` — the latter must name a specific accountable owner (not a team name) for resolving `# Needs Confirmation` items, with a stated turnaround SLA.

**Exit criteria for Week 1 — do not proceed to Week 2 until all of these are true:**
- [ ] Guard scripts pass unit tests in CI, independent of any model call.
- [ ] **Security/governance sign-off obtained** on the GitHub MCP token's scope (read-only + PR-creation only) and the repo allowlist. This is a named approval step, not an assumption baked into "we wrote guard scripts."
- [ ] `roles-and-slas.md` has a named owner, not a placeholder.
- [ ] The three Skills (`okf-structural-pass`, `okf-github-enrichment`, `okf-baseline-gapfill`) each have an assigned author and a *different* assigned reviewer.

Automation does not open its first PR against a real repo until this checklist is signed off.

---

## Week 2 — Structural pass live, opening PRs

**Build:**
- `okf-structural-pass.yml`, covering the full sequence in `ENTERPRISE_BASELINE.md` §6.1 — including step 3, the bounded `okf-github-enrichment` fetch when a merged PR references a specific issue number. This is explicitly in scope for Week 2, not deferred — it rides on the same workflow and the same PR as the structural pass, so building one without the other leaves a documented capability unimplemented.
- Run the `okf-structural-pass` Skill's evaluation suite (3–5 cases) and get it version-pinned before it's allowed to run against real merges.
- Confirm the deterministic guards (`shrinkage_guard.py`) actually block a bad write in a test run before relying on them in production.

**Exit criteria:**
- [ ] At least 3 real merged PRs in the pilot repo have produced a correctly-scoped OKF update PR.
- [ ] At least one of those exercised the bounded `okf-github-enrichment` fetch (a PR that referenced a specific issue).
- [ ] No guard-script false negatives observed (a bad write that should have been blocked, wasn't).

---

## Week 3 — On-demand flow and one baseline capability

**Build:**
- `okf-on-demand.yml` (`ENTERPRISE_BASELINE.md` §6.2), covering the full discover → confirm → generate flow.
- Seed **one** business-capability baseline using the document-seeded default path (§7.5 / §4.3): one bounded supporting document (a wiki export or user guide), the confirmed code paths, and an honest `# Needs Confirmation` list where the document and the code disagree or one has no counterpart in the other.
- Route the resulting `# Needs Confirmation` items to the named owner from `roles-and-slas.md` and track resolution time.

**Exit criteria:**
- [ ] One complete baseline doc merged, with at least one `# Needs Confirmation` item actually resolved by a human (not left open) before this week closes.
- [ ] The on-demand flow produces a PR, not a direct commit, same as the reactive flow.

---

## Week 4 — Demo and pilot metrics

**Demo script — "before vs. after" for one feature**, per `ENTERPRISE_BASELINE.md`'s intent of proving a reliable context layer:

- **Before:** show the feature with no OKF doc — tribal knowledge, scattered across old tickets, comments, and whoever remembers.
- **After:** show `okf/business-functions/<feature>.md` — real code paths, citations to the specific issue or supporting document that explains *why*, and the change log from `log.md`.
- **Live test:** have an AI assistant answer three questions using only the governed OKF bundle and repo artifacts, with no other context supplied:
  - "What does this feature do?"
  - "What code is involved?"
  - "What changed, and why?"

Answering all three accurately, with correct citations, is the actual proof of the "reliable context layer" claim — a compelling doc alone doesn't prove it, verified answers from it do.

**Publish pilot metrics** (baseline collected across weeks 2–4):

| Metric | What it catches |
|---|---|
| % merged code PRs that triggered a correctly-scoped OKF update PR | Whether `code_paths` mapping and the reactive trigger are actually working |
| Median time from merge to OKF update PR published | Pipeline latency |
| # `# Needs Confirmation` items unresolved > 7 days | Whether the human accountability loop (`roles-and-slas.md`) is actually holding |
| **Citation/business-rules accuracy spot-check** (sample 3–5 docs, manually verify cited facts still match current code behavior) | The specific drift failure mode where a code change updates the mechanical `# Schema`/`# Endpoints` sections but leaves a stale `# Business Rules` section untouched — the reactive pipeline alone won't catch this; only a periodic check (tied to the §7.4 `okf-recheck` rotation) will |
| Developer survey: "AI answers are grounded in project context" (baseline vs. 30 days) | Qualitative trust signal — pair with the citation spot-check above, since a survey alone won't surface silent staleness |

---

## After the pilot

Do not expand to additional repos until:
1. All Week 4 metrics have a baseline reading.
2. At least one `okf-recheck` cycle has run and surfaced (or confirmed the absence of) drift.
3. The governance checklist in `ENTERPRISE_BASELINE.md` §11 is fully checked, not partially.

Expansion should follow the existing business-function priority ranking — 1–2 additional domains per cycle, not a bundle-wide rollout, so each cycle's metrics stay attributable to a specific, reviewable change rather than being averaged across too much scope to act on.
