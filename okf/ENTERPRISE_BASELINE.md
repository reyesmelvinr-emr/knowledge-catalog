# OKF Enterprise Profile — GitHub MCP Baseline

**Intended location:** `okf/profiles/github-mcp/ENTERPRISE_BASELINE.md`
**Relationship to core format:** This profile does not modify `okf/SPEC.md`. It defines *how* bundles get generated, enriched, and maintained in an environment where GitHub MCP is the only approved connector. Nothing here changes the bundle/frontmatter/linking rules in the core spec — a bundle produced under this profile is a fully conformant OKF bundle, readable by any consumer that doesn't care how it was produced.

---

## 1. Constraint and scope

Security review has approved **GitHub MCP only**. No Confluence, Jira, Slack, or generic web-fetch connectors are in scope. Before designing around this, it's worth being precise about what actually needs MCP:

| Capability | Needs MCP? | Notes |
|---|---|---|
| Reading local repo files to extract structure (models, routes, schemas) | **No** | Filesystem access — a CI runner with the repo checked out reads this directly. |
| Reading DB schema metadata | **No** | Direct client library (e.g. `google-cloud-bigquery`), not MCP. |
| Reading a specific, named GitHub Issue/Wiki page/PR | **Yes — GitHub MCP covers this** | Single bounded fetch, not a search. |
| Searching/crawling Issues, Discussions, Wiki history across a repo | **Not used in this profile** | See §4.2 — deliberately excluded on cost and reliability grounds, not just connector policy. |
| Reading Confluence/Jira/Slack for business context | **Yes, but not approved** | See §8 for mitigation. |
| Opening a PR with proposed OKF doc changes | **Yes — GitHub MCP covers this** | The only write path. |

**Net effect:** the structural pass (reading your own code) is unaffected — it never needed MCP. The enrichment pass loses arbitrary crawling and is rescoped to bounded, named lookups only. This is a smaller, easier-to-govern surface, which is the point.

---

## 2. Component architecture: three layers, not one

Automation under this profile is split into three distinct layers. Conflating them is the most common implementation mistake — each layer has a different cost profile and a different governance bar.

| Layer | What it does | Where it lives | Governance |
|---|---|---|---|
| **Deterministic tooling** | Extraction, validation, index regeneration — no model call, same output every run | `okf/tools/*` (plain scripts) | Unit-tested in CI, no evaluation suite needed |
| **Skills** | Judgment tasks — writing prose, deciding relevance, reconciling documents | Three named Skills (§4) | Author ≠ reviewer, evaluation suite, version-pinned |
| **Orchestration** | Detecting events (PR merged, issue labeled) and invoking the right layer with the right scope | `.github/workflows/okf-*.yml` | Reviewed as ordinary CI config; no model call, so no Skill-grade governance needed |

This is a hard requirement, not a style preference: do not build a bespoke agent runtime to replace GitHub Actions' event-listening (Actions already does this, for free, with an auditable YAML permissions model), and do not push deterministic logic into a Skill's model-driven judgment (it adds cost and non-determinism to something that should behave identically every run).

---

## 3. Deterministic tooling — `okf/tools/*`

| Script | Purpose |
|---|---|
| `structural_pass.py` | **Extraction only** — parses source (route attributes, method signatures, DTO fields, schema) into structured data. Does **not** write prose; its output is the input to the `okf-structural-pass` Skill's prose-generation step. Keeping this split explicit matters: if this script is treated as producing the final doc, the prose-generation judgment silently disappears from scope. |
| `validate_okf.py` | Frontmatter/structure conformance check against `okf/SPEC.md`'s required keys. |
| `regenerate_index.py` | Regenerates `index.md` at each directory level (mechanical, no model call). |
| `shrinkage_guard.py` | Refuses to write a doc that drops a previously-documented schema field or shrinks the `# Citations` count without an equivalent net addition. This is the deterministic gate that blocks a bad automated write — it runs before any PR is opened, not as a suggestion during review. |

---

## 4. Skills catalog

Three Skills. All three: live in the same Git repo as the OKF bundle, require separate author/reviewer, require a 3–5 case evaluation suite before version promotion, and are version-pinned in production.

### 4.1 `okf-structural-pass`
- **Trigger:** invoked by `okf-structural-pass.yml` (§6.1) when source files under tracked domains change.
- **Input:** `structural_pass.py`'s extracted output (§3).
- **Tools:** local filesystem read/write, code execution for the deterministic scripts. **No MCP dependency.**
- **Job:** turn structured extraction into readable prose — mechanical description only, no business context, no inference about *why* something exists.
- **Model tier:** cheap/fast — extraction-to-prose, not judgment.
- **Output:** proposed OKF doc changes, staged for PR — never committed directly.

### 4.2 `okf-github-enrichment`
- **Trigger:** the triggering PR's description/commit message references a specific issue number (e.g., "Fixes #612"). **Never** a scheduled or open-ended crawl.
- **Tools:** GitHub MCP, restricted to a single named fetch ("get issue #612") — search/list operations against repo history are out of scope for this Skill.
- **Job:** apply the four-gate reference test (topic shape / not bundle-level meta / citation test / reuse test — see Appendix A for exact criteria and worked examples) to that one issue; augment the doc `okf-structural-pass` already touched, or skip.
- **Model tier:** stronger reasoning tier — the four-gate judgment call still matters — but cost is proportional to one change at a time, never to repo history size.
- **Hard limit:** one fetch per triggering reference. If the referenced issue is missing or too thin to pass the gates, flag the gap for a human rather than searching for a substitute.
- **Workflow integration:** runs as part of `okf-structural-pass.yml` (§6.1), not a separate scheduled workflow — its output folds into the same PR the structural pass opens.

### 4.3 `okf-baseline-gapfill`
- **Trigger:** on-demand, as part of the intake flow in §7 — invoked after the structural pass has produced a mechanical draft, when supporting documentation was attached to the request.
- **Tools:** reads a human-supplied document (wiki export, user guide, design doc) attached to the request. Not a crawl. If the supplied document lives in a GitHub Wiki, GitHub MCP fetches that one named page — same bounded shape as §4.2.
- **Job:** reconcile the supplied document against the structural draft. Where they agree, augment the draft with sourced business context, cited to the source document. Where the doc describes something with no matching code, or the draft has behavior the doc never mentions, emit an explicit `# Needs Confirmation` checklist — never guess.
- **Model tier:** stronger tier — reconciling two known, finite documents is a bounded but genuine judgment task. Cost scales with what was supplied, not with repo or issue-tracker size.
- **Human step, not optional:** a named Product Owner or Tech Lead resolves `# Needs Confirmation` items directly, with no model inference in that step. This is the highest-fidelity source in the pipeline — it comes from someone who actually knows the answer, not from sparse historical text.
- **Default path for legacy/existing applications** (§7.5) — preferred over relying on Issue/PR history, which for an established application is both expensive to evaluate (large, low-signal corpus) and unreliable (inconsistent ticket quality).

---

## 5. Bundle structure requirements

Every pilot or production repo adopting this profile includes, at minimum:

```
okf/
├── index.md
├── log.md
├── business-functions/
│   └── index.md
├── concepts/
│   └── index.md
└── references/
    └── index.md
```

`references/` is required, not optional, even for a first pilot — `okf-github-enrichment` and `okf-baseline-gapfill` both produce citations (to Issues or supplied documents) that belong here per the core spec's citation conventions. **Required by this enterprise profile, not by core OKF conformance** — `okf/SPEC.md` §9 explicitly says a bundle must not be rejected for a missing `index.md`, and the same permissive-consumption principle extends to any specific subdirectory; a bundle without `references/` is still a fully conformant OKF bundle. This profile requires it anyway, as an implementation default, because omitting it doesn't break anything mechanically (the indexer skips empty directories) but signals the wrong default to whoever builds the second pilot repo.

---

## 6. Pipelines (`.github/workflows/okf-*.yml`)

### 6.1 `okf-structural-pass.yml` — triggered on PR merge
1. Run `structural_pass.py` (extraction) against the diff's changed paths.
2. Invoke the `okf-structural-pass` Skill (§4.1) to produce prose.
3. **If the merged PR's description/commits reference a specific issue number:** invoke `okf-github-enrichment` (§4.2) against that one issue, folding its output into the same draft. This step is explicitly part of this workflow, not a separate scheduled job — a common implementation gap is leaving this bounded-fetch step unscoped or forgetting it entirely.
4. Run `shrinkage_guard.py` and `validate_okf.py` — fails fast, no model call, blocks the PR from opening if violated.
5. Open a PR against `okf/**` via GitHub MCP. Never commit directly to `main`.
6. Standard human review and approval — same reviewers as code, or a designated docs owner.

### 6.2 `okf-on-demand.yml` — triggered on issue labeled `okf-request`
1. Parse the issue body (business function name, description, known components, optional attached documentation).
2. If components aren't fully specified, run the discovery subagent (§7.2) and post candidates as a comment; wait for human confirmation before proceeding.
3. Run `okf-structural-pass` against the confirmed path list.
4. If supporting documentation was attached: run `okf-baseline-gapfill` (§4.3) — this is the default path (§7.5). If a specific issue was named instead: run `okf-github-enrichment` (§4.2) against that one issue only.
5. Open a PR; comment the link back on the originating issue; close the issue on merge.

### 6.3 On merge to `main` (bundle changed)
Static site generation regenerates the bundle viewer — no model call, pure code. Publish to internal static hosting or GitHub Pages.

---

## 7. On-demand documentation requests

Everything in §6.1 is reactive — triggered by a PR merge. That doesn't cover documenting a business function whose code isn't currently changing.

### 7.1 Intake
A GitHub Issue, labeled `okf-request`:
```
Business function: <name>
Description: <what it does, who owns it>
Known code components (optional): <repo/path, repo/path, ...>
Supporting documentation (optional): <attached wiki export / user guide / design doc>
Priority: <maps to business-function priority ranking>
```

### 7.2 Discovery, confirm, generate
1. **Discovery (agent-assisted):** if components aren't fully named, a small subagent searches the codebase for candidates (keyword/path search, naming conventions) matched to the description. Output is *proposed*, not final.
2. **Confirmation (human):** the requester or a code owner confirms/edits the list as an issue comment.
3. **Generation:** `okf-structural-pass` runs against the confirmed path list.

### 7.3 Why this needs no new agent infrastructure
Both Skills used here are the same ones used reactively — the only difference is how scope is determined (confirmed path list vs. diff-derived paths). The discovery step is a small, bounded, read-only search task with the same cheap-model, no-MCP profile as the structural pass.

### 7.4 Ongoing maintenance for genuinely static code
If a component truly never changes, §6.1 correctly never re-touches its doc — but "no code diff" doesn't always mean "no behavior change" (config-driven behavior, external dependency shifts). A scheduled Action opens a low-priority `okf-recheck` issue for documented-but-static components on a rotation, reusing the same intake mechanism rather than a separate staleness system.

### 7.5 Document-seeded baseline — default for legacy/existing applications
1. Structural pass runs first, producing a mechanical draft from code alone.
2. `okf-baseline-gapfill` reads *only* the attached document(s) — no search, no crawl. It reconciles the draft against the supplied document, augmenting where they agree and flagging `# Needs Confirmation` where they don't.
3. A named Product Owner/Tech Lead resolves confirmation items directly, in the PR.
4. Merge as baseline. From here forward, only §4.2's bounded single-issue lookup applies to future changes — never a broad crawl.

Treat §7.2's crawl-free discovery as locating *code*, and §7.5 as the default source of *business context* for anything with meaningful existing history. Fall back to open confirmation-only drafts (structural pass alone, heavy on `# Needs Confirmation`) only for components too new or too undocumented to have anything worth attaching.

---

## 8. Coverage gap: Confluence/Jira/Slack

Restricting to GitHub MCP makes business context outside GitHub invisible to automated enrichment. Two low-cost mitigations:

1. **Encourage mirroring high-value context into GitHub.** ADRs, RFCs, and design docs already tend to live as markdown and are easy to move into the repo (or a dedicated internal docs repo) where GitHub MCP can reach them. A process nudge, not new infrastructure.
2. **Route through `okf-baseline-gapfill` (§4.3/§7.5), not ad hoc pasting.** Attach whatever Jira/Confluence export exists as a supporting document on the `okf-request` issue and let the gap-fill pass reconcile it against the code, with human resolution of anything it can't confirm.

**Do not** work around this by having a Skill call raw API tooling against Confluence/Jira directly — that reintroduces the exact connector risk the security review rejected, without MCP's governance layer around it.

---

## 9. GitHub Copilot integration

Copilot operates at write-time (editor, PR authoring, code review); this profile's Skills operate at merge-time or on-demand. They aren't competing, but universal Copilot access changes the inputs this pipeline receives.

- **Shared `AGENTS.md`.** Repo-root conventions ("if editing a file listed in an OKF doc's `code_paths`, note the business-context change in the PR description") should live in `AGENTS.md`, not a Claude-specific file — Copilot's coding agent/chat/review and Claude-based tooling both read it natively. See §10 for why this is `AGENTS.md` and not a fourth Skill.
- **Better raw material, future PRs only.** If Copilot's PR-description features are enabled, PR text quality improves at authoring time, before `okf-github-enrichment`'s fetch ever runs. This does not help legacy history — §7.5 remains the backfill path.
- **Copilot code review as an advisory nudge only.** A custom instruction can flag PRs touching `code_paths`-tracked files without a business-context note. This stays advisory — Copilot code review's instruction-following is documented as non-deterministic, so `shrinkage_guard.py` remains the actual merge gate, not this nudge.
- **Do not reimplement the three Skills inside Copilot's separate agent ecosystem.** That duplicates governance and maintenance for identical behavior with no offsetting benefit.

---

## 10. Why `AGENTS.md`, not a fourth Skill, for cross-tool conventions

Not a general format preference — the two formats fit different jobs:

- **Reach:** `AGENTS.md` is read by multiple vendors' agents (Copilot, Claude-based tooling). `SKILL.md` is Anthropic-specific; a cross-tool reminder there is invisible to half the developers using it.
- **Shape:** a standing repo convention ("remember X whenever you touch Y") isn't a bounded, invokable capability with its own tools and multi-step judgment — that's what the three Skills in §4 are for.
- **Loading model:** Skills use progressive disclosure (loaded when judged relevant); a repo-wide convention should be always-on, which is how `AGENTS.md` behaves natively.
- **Governance weight:** Skill-grade governance (separate reviewer, eval suite, version pinning) is disproportionate for a one-line reminder.

---

## 11. Governance checklist before rollout

- [ ] **Security/governance sign-off obtained** on GitHub MCP token scope and repo allowlist **before any Skill is permitted to open its first PR** — this is an explicit gate, not implied by "guard scripts exist."
- [ ] GitHub MCP token scoped to read-only + PR-creation only (no direct write/merge permission)
- [ ] Explicit repo allowlist enforced in each Skill's bundled script, not left to prompt instructions alone
- [ ] Skill author ≠ Skill reviewer for all three Skills in §4
- [ ] Evaluation suite (3–5 cases: should-mint, should-augment, should-skip) run before each Skill version promotion
- [ ] Guard scripts (§3) covered by unit tests, run in CI independent of any model call
- [ ] `okf-github-enrichment` restricted to single named-issue fetches, enforced in the bundled script — never search/list calls against repo history
- [ ] `okf-baseline-gapfill`'s `# Needs Confirmation` items block PR merge until a named human has resolved them — no silent auto-resolution
- [ ] All writes land as PRs; no direct commits from any Skill or workflow
- [ ] `roles-and-slas.md` names an accountable owner and turnaround SLA for `# Needs Confirmation` resolution
- [ ] Rollout follows business-function priority — start with 1–2 domains, expand after the first cycle proves out

---

## 12. Open items for the team

- Who owns re-approving `okf-github-enrichment` or `okf-baseline-gapfill` when GitHub MCP's tool surface changes?
- Is a dedicated internal docs repo (§8, mitigation 1) worth standing up now, or wait to see the actual coverage gap?
- Who confirms the §7.2 discovery step's candidate list — the requester, a designated code owner, or whoever filed the `okf-request`?
- What cadence for the §7.4 `okf-recheck` rotation — flat quarterly, or tied to a risk/criticality tier?
- What document formats must `okf-baseline-gapfill` accept in v1 — worth scoping narrowly (e.g., markdown/PDF only) rather than building a general parser upfront?

---

## Appendix A: The four-gate reference test (canonical definition)

Referenced repeatedly in §4.2 and §4.3 but defined once, here, to remove reviewer-to-reviewer variance. This is the test both `okf-github-enrichment` and `okf-baseline-gapfill` apply to a candidate piece of content (a fetched Issue, or a section of a supplied document) before it's allowed to land in an OKF doc. **All four gates must pass.** If any fails, the correct action is **skip** — never a partial or best-guess inclusion.

1. **Topic shape.** The content defines something concretely referenceable by name — a business rule, a policy, a metric definition, an enum/status value, a field/parameter meaning, a pricing/billing note, or a units/timezone/identifier convention. General discussion, opinion, or narrative does not qualify, even if it's on-topic.
2. **Not bundle-level meta.** The content is not an overview, introduction, getting-started guide, tutorial, walkthrough, release notes, changelog, roadmap, FAQ, or planning document. These describe the *process* of building or discussing something, not a fact about how the system behaves.
3. **Citation test.** You can write a single, natural sentence of the form *"See the [X] for context"* where X is the specific fact, naming a concrete noun — not a vague pointer like "see the discussion" or "see the thread." If the sentence can't be written without hedging, the gate fails.
4. **Reuse test.** Either (a) at least two existing OKF concepts would benefit from citing this content, or (b) exactly one existing concept needs it as load-bearing background it can't function without. A passing citation test alone isn't sufficient — the content also has to matter to something already documented, not just be true.

**Deciding mint vs. augment (once all four gates pass):** if the content clears the reuse test via (a) — two or more concepts benefit — mint a standalone doc under `references/` and link from each. If it clears via (b) only — single-concept load-bearing background — augment the existing concept doc directly rather than minting a one-citation reference file.

**Tie-breaker for ambiguous currency or reliability:** if the content technically passes all four gates but its *current accuracy* is unclear — e.g., it describes an exception, a workaround, or a decision without stating whether it's still in effect — do not assert it as settled fact. Route it into a `# Needs Confirmation` item instead (the same mechanism `okf-baseline-gapfill` uses), even though this content came from `okf-github-enrichment`. This is the deliberate default: when in doubt, flag for a human rather than let an LLM's confidence stand in for currency it can't verify from a single fetched document.

### Worked examples, using the `EnrollmentManagement` business function from earlier in this project

**Pass — augment.** Issue #482: *"Per Registrar policy, students must enroll at least 30 days before term start to allow processing time. Escalated after the Fall 2025 late-enrollment backlog."*
- Topic shape: ✅ — a specific business rule (a 30-day enrollment window).
- Not meta: ✅ — a rule, not an overview or roadmap item.
- Citation test: ✅ — *"See the 30-day enrollment window policy for why this check exists"* reads naturally.
- Reuse test: ✅ via (b) — this is load-bearing background for exactly one concept (`EnrollmentManagement`'s `ValidateEnrollmentWindow` method), not reused elsewhere.
- **Action:** augment `business-functions/enrollment-management.md` directly with a `# Business Rules` section citing Issue #482. Do not mint a standalone `references/` doc for a single-concept fact.

**Fail — skip.** Issue titled *"Q3 planning notes for Enrollment team"* — a mix of roadmap items, a hiring update, and general meeting notes.
- Topic shape: ❌ — nothing here is a definable, citable fact; it's narrative.
- Not meta: ❌ — this is a planning document by its own description.
- **Action:** skip. Do not attempt a partial extraction of the one roadmap line that happens to mention enrollment — if the overall document fails gates 1–2, don't mine it for fragments.

**Borderline — flag, don't assert.** Issue: *"Temporarily bypassing the 30-day window for VIP applicants this term — see thread for context."*
- Topic shape: ✅ (technically) — describes a specific rule variance.
- Not meta: ✅ — not an overview.
- Citation test: ✅ (technically) — a sentence can be written.
- Reuse test: ✅ via (b) — relevant background for the same concept.
- **But:** nothing in the issue states whether this exception is still active, was reverted, or was a one-time event. All four gates pass mechanically, but currency is unverifiable from this one document.
- **Action:** apply the tie-breaker. Do not augment the doc asserting a VIP bypass exists as current behavior. Instead add a `# Needs Confirmation` item: *"Issue #_ describes a temporary VIP enrollment-window bypass — confirm whether this is still active policy or was time-boxed to a specific term."* Let a human resolve it, same as any `okf-baseline-gapfill` gap.
