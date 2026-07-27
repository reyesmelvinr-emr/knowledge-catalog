# OKF Generation & Maintenance Architecture — Draft
### Scoped to GitHub MCP as the sole approved connector

**Status:** Draft for review
**Constraint driving this design:** Security review has approved GitHub MCP only. No Confluence, Jira, Slack, or generic web-fetch connectors are in scope. This document reworks the earlier three-part recommendation (custom agents / Agent Skills / pipelines) against that constraint.

---

## 1. What the constraint actually changes

Before reworking anything, it's worth separating what genuinely needs MCP from what doesn't:

| Capability | Needs MCP? | Notes |
|---|---|---|
| Reading local repo files to extract structure (models, routes, schemas) | **No** | This is filesystem access. Claude Code / a CI runner with the repo checked out reads this directly — no connector involved. |
| Reading DB schema metadata (the original repo's BigQuery source) | **No** | Direct client library (`google-cloud-bigquery`), not MCP. Same pattern applies to any first-party DB/API client your team already uses. |
| Reading GitHub Issues, Discussions, Wikis, PR descriptions, release notes | **Yes — GitHub MCP covers this** | This is exactly what's approved. |
| Reading Confluence/Jira/Slack for business context | **Yes, but not approved** | This is the capability we're losing. See §5 for the mitigation. |
| Opening a PR with proposed OKF doc changes (rather than committing directly) | **Yes — GitHub MCP covers this** | This becomes the primary write path. |

**Net effect:** the *structural pass* (reading your own code) is unaffected — it never needed MCP. The *enrichment pass* loses arbitrary web/internal-wiki crawling and is rescoped to GitHub-native content only. That's a real reduction in coverage, but it's also a smaller, easier-to-govern surface, which is the actual point of restricting to one connector.

---

## 2. Custom agents — recommendation: scale back, don't replicate the repo's pattern

The reference implementation's two bespoke ADK agents (`build_bq_agent`, `build_web_agent`, each with its own `Runner`/`InMemorySessionService`) were built when the web-ingestion pass needed a general-purpose crawler with its own link-following and host-allowlist logic (`web/fetcher.py`, `tools/web_tools.py`). With GitHub MCP as the only connector, that bespoke crawler is no longer needed — GitHub MCP already provides governed, scoped tool calls for issues/wiki/PRs.

**Recommendation:** don't build or maintain a custom agent runtime for this. Reasons:
- The custom fetcher's job (host allowlisting, depth-capping, path-prefix filtering — see `tools/context.py`'s `WebState`) is now handled by GitHub MCP's own scoping (which repos/orgs it can reach), so you'd be duplicating governance your security team already owns.
- A bespoke agent is another thing to patch, test, and get through review independently of the Skills and pipeline governance you're already standing up.

**Where a custom agent *is* still justified:** a small, non-LLM structural scanner (the `Source`/`ConceptRef` abstraction) that walks the codebase and emits candidate concepts. This is deterministic code, not a model-driven agent, and should stay a plain script/service — not something wrapped in an agent framework. Keep `bundle_tools.py`'s schema-shrinkage and citation-count guards here too; they're pure validation logic with no reason to involve a model call.

---

## 3. Agent Skills — three skills, rescoped

### 3.1 `okf-structural-pass`
- **Trigger:** invoked from Claude Code (or CI) when source files under tracked domains change.
- **Tools:** local filesystem read/write, code execution for the deterministic scripts (schema extraction, `write_concept_doc`-equivalent, shrinkage guard, index regeneration). **No MCP dependency at all.**
- **Model tier:** cheap/fast — this is extraction, not judgment.
- **Output:** proposed OKF doc changes staged locally, never committed directly (see pipeline in §4).

### 3.2 `okf-github-enrichment` (revised — bounded lookup, not a crawl)

**Important revision:** the original design for this skill had it *searching* Issues/Discussions/Wiki history across allowlisted repos, mirroring the reference implementation's `web_ingestion_instruction.md` crawl pattern. For existing or legacy applications, that doesn't hold up — years of Issue/PR history means evaluating a large, low-signal corpus against the four-gate test (canonical definition and worked examples now in `ENTERPRISE_BASELINE.md` Appendix A), at real token cost, against content that may never have been written with enough detail to be useful anyway. This skill is narrowed accordingly:

- **Trigger:** a specific code change whose commit/PR references a specific issue number (e.g., "Fixes #612") — never a scheduled or open-ended crawl.
- **Tools:** GitHub MCP, but restricted to a single, named fetch — "get issue #612" — not a search or list operation across a repo's history.
- **Behavior:** same four-gate reference test as before, but applied to exactly one bounded input at a time. No ranking, no relevance search across a corpus.
- **Model tier:** stronger reasoning tier is still appropriate here — deciding whether the one referenced issue passes the four gates is still a judgment call — but the *cost* is now proportional to one change at a time, not to repo history size.
- **Hard limit:** one fetch per triggering reference. No fallback to broader search if the referenced issue is missing or thin — flag the gap for a human instead (see §3.3).

### 3.3 `okf-baseline-gapfill` (new — for legacy/existing-application baselines)

This replaces open-ended enrichment as the primary way to build a *baseline* for an application that already exists, per the token-efficiency concern with legacy Issue/PR history.

- **Trigger:** on-demand, as part of the intake flow in §5 — invoked once the structural pass has produced a mechanical draft.
- **Tools:** reads a human-supplied document (wiki export, user guide, design doc) attached to the request — not GitHub MCP at all, not a crawl of any kind. If the supplied doc happens to live in GitHub (a repo's Wiki, say), GitHub MCP fetches that one named page — still a bounded, single fetch, same shape as §3.2.
- **Behavior:** cross-references the supplied document against the structural draft. Where they agree, augments the draft with the sourced business context. Where the supplied doc describes something with no matching code path, or the structural draft has behavior the doc doesn't mention, it emits an explicit `# Needs Confirmation` checklist rather than guessing.
- **Model tier:** reconciliation between two known, finite documents is a bounded reasoning task — stronger tier justified, but cost scales with what was supplied, not with repo or issue-tracker size.
- **Human step, not optional:** a Product Owner or Tech Lead resolves the `# Needs Confirmation` items directly — no model inference involved in that step at all. This is deliberately the highest-fidelity source in the whole pipeline, since it comes from the person who actually knows the answer rather than being inferred from sparse historical text.

All three skills should live in the same Git repo as the OKF bundle, go through the enterprise Skill governance flow described previously (separate author/reviewer, evaluation suite, pinned versions in production), and get re-reviewed any time their bundled scripts change — same bar as a source-code PR.

---

## 4. Pipeline design

```
┌─────────────────────────────────────────────────────────────┐
│  PR merged to main (source code change)                     │
│         │                                                    │
│         ▼                                                    │
│  CI: run okf-structural-pass (no MCP; local repo only)       │
│         │                                                    │
│         ▼                                                    │
│  Deterministic guard scripts run (schema/citation shrinkage) │
│    – fails fast, no model call, blocks merge if violated      │
│         │                                                    │
│         ▼                                                    │
│  If OKF docs changed: GitHub MCP opens a PR with the diff     │
│    (never commits directly to main)                          │
│         │                                                    │
│         ▼                                                    │
│  Human review + approval (same reviewers as code, or a        │
│    designated docs owner) — standard PR review, no new tool   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  PR message references an issue (e.g. "Fixes #612")           │
│         │                                                    │
│         ▼                                                    │
│  okf-github-enrichment fetches ONLY issue #612 via GitHub MCP │
│    — single named fetch, never a search or crawl              │
│         │                                                    │
│         ▼                                                    │
│  Four-gate test applied to that one issue; augments the       │
│    doc the structural pass already updated, or skips          │
│         │                                                    │
│         ▼                                                    │
│  Folded into the same PR from the structural pass above        │
│    (or its own small PR); human review required before merge  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  On merge to main (bundle changed)                            │
│         │                                                    │
│         ▼                                                    │
│  Static site generation (viewer/generator.py-equivalent)       │
│    — no model call, pure code — regenerates viz.html          │
│         │                                                    │
│         ▼                                                    │
│  Published to internal static hosting (or GitHub Pages)        │
└─────────────────────────────────────────────────────────────┘
```

Key principle carried through all three flows: **the agent/skill only ever proposes a PR; it never writes to `main` directly.** This is both a security control and the natural way to fold OKF maintenance into review processes your team already runs.

---

## 5. On-demand documentation requests (no active code changes)

Everything in §4 is reactive — triggered by a PR merge or a scheduled crawl. That doesn't cover the case where a team simply wants "document what X does today" for code that isn't changing. This needs a separate, manually-triggered entry point rather than forcing it through the diff-based pipeline.

### 5.1 Intake mechanism
Use a GitHub Issue as the request form — this keeps intake inside the one approved connector rather than adding new tooling. A lightweight issue template:

```
Business function: <name>
Description: <what it does, who owns it>
Known code components (optional): <repo/path, repo/path, ...>
Priority: <maps to the earlier business-function ranking>
```

Label it `okf-request`. A GitHub Action on `issues: labeled` triggers the workflow below — no polling, no new infrastructure beyond what §4 already uses.

### 5.2 Three-step flow: discover, confirm, generate

Because there's no diff to derive scope from, the pipeline needs an explicit discovery step before the structural pass can run — and that step deserves a human checkpoint. Misidentifying which services/tables belong to a business function is a foundational error that then gets baked into every doc generated downstream.

1. **Discovery (agent-assisted).** A small subagent (Claude Code's `context: fork` pattern works well here) searches the codebase for candidate components — keyword/path search, service and route naming, directory structure — matched against the request's description. Output is a *proposed* component list, not a final one.
2. **Confirmation (human).** The requester or a code owner confirms or edits the candidate list, as a comment on the same GitHub Issue — no new UI needed.
3. **Generation.** Once scope is confirmed, `okf-structural-pass` runs against exactly that path list (same skill as the reactive pipeline — just a fixed list instead of a git diff). For business context beyond what the code itself shows, use `okf-baseline-gapfill` (§3.3) if supporting documents exist — see §5.5, which is the recommended default for any application with meaningful history. `okf-github-enrichment` (§3.2) still applies, but only if a specific, named GitHub Issue is already known to be relevant — never as an open-ended search.

Output lands as a PR against the OKF bundle, reviewed the same as any other change. The triggering GitHub Action can comment the PR link back on the originating issue and close it on merge.

### 5.3 Why this doesn't need new agent infrastructure
This reuses both existing Skills — the only difference is how scope is determined (explicit/confirmed path list vs. diff-derived paths). The one new piece is the discovery subagent, and it's a small, bounded, read-only search task — same cheap-model, no-MCP-needed profile as the structural pass already has.

### 5.4 Ongoing maintenance for genuinely static code
If these components truly never change, the reactive pipeline in §4 will correctly never re-touch their docs — which is fine, but "no code diff" doesn't always mean "no behavior change" (config-driven behavior, external dependency shifts, etc.). A lightweight periodic recheck — say, quarterly — is worth adding: a scheduled Action opens a low-priority `okf-recheck` issue for documented-but-static components on a rotation, reusing the same intake mechanism rather than building a separate staleness-detection system.

### 5.5 Document-seeded baseline (recommended default for legacy/existing applications)

§5.2's discovery step is still useful for *locating* code components, but for the *business-context* portion of a baseline, don't rely on searching Issue/PR history — for an application with years of accumulated tickets, that's both expensive (large low-signal corpus to evaluate against the four-gate test) and unreliable (legacy tickets are often thin or inconsistent). Use `okf-baseline-gapfill` (§3.3) instead:

1. **Extend the intake template** from §5.1 with an optional attachment field:
   ```
   Business function: <name>
   Description: <what it does, who owns it>
   Known code components (optional): <repo/path, repo/path, ...>
   Supporting documentation: <attached wiki export / user guide / design doc>
   Priority: <maps to the earlier business-function ranking>
   ```
2. **Structural pass runs first, unchanged** — a mechanical draft from the code alone, no dependency on whatever was attached.
3. **Gap-fill pass reads only the attached document(s).** No search, no crawl — this is a bounded reconciliation between two known, finite inputs (the structural draft and the supplied doc). Where they agree, the business context gets folded into the draft with a citation to the source document. Where the supplied doc describes something with no matching code, or the draft has behavior the doc never mentions, it's listed under an explicit `# Needs Confirmation` heading rather than guessed at.
4. **A Product Owner or Tech Lead resolves the confirmation items directly**, as a comment or a direct edit to the PR — no model inference in this step. This is intentionally the highest-fidelity input in the whole pipeline, since it comes from someone who actually knows the answer.
5. **Merge as the baseline.** From this point forward, only the bounded, single-issue lookup in §3.2 applies to future changes to these components — never a broad crawl. See the Stage 3 (code-change) walkthrough earlier in this conversation for what that looks like once the baseline exists.

This should be the **default path** for any application where meaningful history already exists; treat §5.2's crawl-based discovery as a fallback only for components genuinely too new or too undocumented to have anything worth attaching.

---

## 6. Mitigating the lost coverage (Confluence/Jira/Slack)

Restricting to GitHub MCP means business context that lives outside GitHub (tickets, chat threads, wiki pages in other tools) is invisible to the enrichment pass. Two low-cost mitigations, ranked by effort:

1. **Encourage — don't force — mirroring high-value context into GitHub.** ADRs, RFCs, and design docs that already tend to live as markdown are easy to move into the repo (or a dedicated internal docs repo) where GitHub MCP can reach them. This is a process nudge, not new infrastructure.
2. **Manual seeding, via `okf-baseline-gapfill` (§3.3/§5.5) rather than ad hoc pasting.** For the handful of top-priority business functions, attach whatever Jira/Confluence export or user-guide content exists as a supporting document on the `okf-request` issue, and let the gap-fill pass reconcile it against the code — with a Product Owner/Tech Lead resolving anything it can't confirm. The ongoing four-gate/augmentation rules then apply to keep it from going stale, but the initial ingestion doesn't require a connector at all.

Do **not** work around this by having the enrichment skill call raw `web_fetch`-style tooling against Confluence/Jira APIs directly — that's reintroducing the exact connector risk the security review rejected, just without the MCP governance layer around it.

---

## 7. Governance checklist before rollout

- [ ] GitHub MCP token scoped to read-only + PR-creation only (no direct write/merge permission)
- [ ] Explicit repo allowlist enforced in the enrichment skill's bundled script, not left to prompt instructions alone
- [ ] Skill author ≠ Skill reviewer for `okf-structural-pass`, `okf-github-enrichment`, and `okf-baseline-gapfill`
- [ ] Evaluation suite (3–5 cases: should-mint, should-augment, should-skip) run before each skill version promotion
- [ ] Guard scripts (`bundle_tools.py`-equivalent) covered by unit tests, run in CI independent of any model call
- [ ] `okf-github-enrichment` restricted to single named-issue fetches, never search/list calls against a repo's history — enforced in the bundled script, not left to model judgment
- [ ] `okf-baseline-gapfill`'s `# Needs Confirmation` items block PR merge until a named human (PO/Tech Lead) has resolved them — no silent auto-resolution
- [ ] All writes land as PRs; no direct commits from any agent or skill
- [ ] Rollout follows the earlier business-function priority list — start with 1–2 domains, expand after the first cycle proves out

---

## 8. Impact of universal GitHub Copilot access on the enrichment process

Copilot and the pipeline above operate at different moments — Copilot at write-time (in-editor, PR authoring, code review), our Skills at merge-time or on-demand. They aren't competing for the same job, but Copilot being available to every developer changes the *inputs* the pipeline receives and opens one real architectural improvement. It does not change what the three OKF Skills do.

### 8.1 Shared `AGENTS.md` for cross-tool conventions
Add a repo-root `AGENTS.md` carrying the OKF-maintenance conventions relevant to any assistant working in the repo — e.g., "if you're editing a file listed in an OKF doc's `code_paths`, the PR description should note the business-context change, not just the mechanical diff." Copilot's coding agent, chat, and code review all read `AGENTS.md` natively, and Claude-based tooling honors the same convention — so this is one file, not a Claude-specific instruction duplicated into a separate Copilot-specific format. See §9 (below) for why this belongs in `AGENTS.md` rather than as a fourth Skill.

### 8.2 Better raw material for the bounded enrichment fetch — future PRs only
If the org enables Copilot's PR-description/summary features, PR text quality improves structurally at the point of authoring, before `okf-github-enrichment`'s single-issue fetch (§3.2) ever runs. This is a genuine, low-cost mitigation of the "assumes Issues/PRs were written with sufficient detail" concern — but only going forward. It does nothing for existing history behind an already-built component, which is exactly why §5.5's document-seeded baseline stays the right approach for backfilling anything already in production. Universal Copilot access reduces how often future components will need that path as badly; it doesn't remove the path.

### 8.3 Copilot code review as an advisory nudge, not a replacement for the guard scripts
A custom instruction (repository-wide or path-specific, scoped to `code_paths`-tracked files) can have Copilot's code review flag a PR that touches tracked code without mentioning business impact. Worth adding — it's a free, already-visible nudge to the developer. It stays advisory only: GitHub's own documentation is explicit that Copilot code review's instruction-following is non-deterministic, so it cannot be the thing that blocks a merge. The deterministic guard scripts in `bundle_tools.py`-equivalent remain the actual gate.

### 8.4 What does not change
Do **not** reimplement `okf-structural-pass`, `okf-github-enrichment`, or `okf-baseline-gapfill` inside Copilot's own separate agent/skill ecosystem. That would mean maintaining two parallel implementations of identical logic in two vendor-specific formats, with governance (evaluation suites, version pinning, review) split across both — added maintenance burden with no offsetting benefit, since the existing Skills already cover this work.

### 8.5 Capacity note
Universal Copilot access typically raises PR volume and velocity org-wide. That's fine under the bounded, single-fetch enrichment model — cost per PR stays constant — but the audit logging in the §7 governance checklist should assume a higher baseline volume of triggering events, not a different cost shape.

---

## 9. Why `AGENTS.md` here, and not a fourth Skill

This isn't a general "`AGENTS.md` beats `SKILL.md`" position — the two formats do different jobs, and the three OKF Skills are correctly built as Skills for exactly the reasons `AGENTS.md` would be wrong for them. The reminder in §8.1 is the reverse case. Specifically:

- **Reach.** The §8.1 content needs to be honored by *every* assistant a developer might be using — Copilot's coding agent, chat, and code review, plus Claude-based tooling. `AGENTS.md` is the interoperable convention multiple vendors' agents read natively. `SKILL.md` is Anthropic's Agent Skills format — Copilot has no equivalent discovery mechanism for it. A cross-tool reminder in `SKILL.md` would simply be invisible to half the developers using it.
- **Shape of the content.** §8.1's content is a standing repo convention — "remember X whenever you touch Y" — not a bounded, invokable capability with its own tools and multi-step judgment (the four-gate test, schema extraction, gap-fill reconciliation). That's what the three actual Skills do, and it's why they're Skills: bounded tasks, their own tool access, versioned, security-reviewed. A one-line contextual reminder doesn't have any of that shape.
- **Loading model.** Skills use progressive disclosure — frontmatter loaded first, full body loaded only when Claude judges it relevant to the current task — which is the right behavior for a narrow capability you don't want cluttering every context window. A repo-wide convention is the opposite: it should be loaded whenever *anything* in the repo is being touched, which is exactly how `AGENTS.md` behaves (always-on repo context), not something that benefits from conditional discovery.
- **Governance weight.** Skills carry deliberate overhead — separate author/reviewer, evaluation suite, version pinning, security review — appropriate for something that executes code and exercises judgment. Putting a one-line PR-description reminder through that same process would be governance disproportionate to what it does.

Net: the choice tracks the same principle used throughout this document — match the tool to the actual job, rather than defaulting to whichever format is more familiar. The generation logic stays in Skills; the cross-tool repo convention goes in `AGENTS.md`.

---

## 10. Open questions for the team

- Who owns re-approving `okf-github-enrichment` or `okf-baseline-gapfill` when GitHub MCP's own tool surface changes (new scopes/capabilities)?
- Is a dedicated internal docs repo (mitigation #1 in §6) worth standing up now, or should we wait to see how much coverage gap the GitHub-only pass actually leaves?
- Who confirms the discovery step's candidate component list in §5.2 — the requester, a designated code owner, or whoever triggered the `okf-request` issue? Worth naming explicitly so requests don't stall waiting on an unclear owner.
- What cadence for the `okf-recheck` rotation in §5.4 — quarterly per component, or tied to some risk/criticality tier from the business-function priority list?
- For §5.5, what document formats does `okf-baseline-gapfill` need to accept out of the gate (PDF, Confluence export, plain markdown) — worth scoping this narrowly for v1 rather than building a general ingestion parser upfront?
- Who's accountable for actually resolving `# Needs Confirmation` items in a reasonable timeframe, so baseline requests don't stall indefinitely waiting on a PO/Tech Lead?
