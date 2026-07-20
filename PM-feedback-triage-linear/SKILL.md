---
name: PM-feedback-triage-linear
description: Use when a PM wants to understand, triage, or aggregate Agentforce customer feedback that FDEs/SEs have shared on Linear (AFP-* issues) — without a meeting. Reads the feedback issues and their in-app evidence documents, parses the machine-readable feedback-record blocks written by the companion FDE-customer-feedback-linear skill, answers the PM's questions with citations, builds cross-customer rollups (by feature, gap type, severity, impact), and posts follow-up questions back on the thread asynchronously. Also records the PM's decision back onto the issue as a machine-readable `triage_record` block and drives the loop to a defined closed state.
---

# PM Feedback Triage ← Linear

The PM-side companion to `FDE-customer-feedback-linear`. FDEs/SEs publish structured feedback (an in-app evidence Document + summary comment, ending in a machine-readable `feedback_record: v1` block). This skill lets a PM's coding agent **consume** that feedback: answer questions about a single item, roll up patterns across many, and close remaining gaps by commenting on the thread — so neither side needs a meeting.

## When to use

- "What is AFP-123 actually about / is it real / what's the ask?"
- "Across all Testing Center feedback, what are the top asks?" / "Which items are adoption-blocking?" / "What did customer X report this quarter?"
- "Draft my follow-up questions for AFP-123" (posted on the thread, not asked in a call)
- "Build a triage rollup doc for this month's feedback"

Do **not** use this to author FDE feedback (that's `FDE-customer-feedback-linear`) or to change issue states/priorities without the PM explicitly asking.

## Prerequisites — Linear workspace

Feedback issues (`AFP-*`) live in the **`eventsmobileapp`** workspace, team **"Agentforce Feedback"**. The Linear MCP connects to one workspace at a time — verify with `get_issue AFP-###` or `list_issues` with `team: "Agentforce Feedback"`. If access fails, have the PM reconnect via `/mcp` → linear → pick **eventsmobileapp**.

## How feedback is structured (the contract)

Each feedback item is one `AFP-*` issue carrying:
1. **Intake description** — customer name, feature family/group, feedback theme, business outcome (the original form fields).
2. **Summary comment** — TL;DR, ask, open questions, link to the doc.
3. **Evidence Document** (linked to the issue) — the deep dive. Its last section is a fenced ```yaml block starting `feedback_record: v1`:

```yaml
feedback_record: v1
issue: AFP-###
date: YYYY-MM-DD
customer: <name | internal-demo>
feature_family: <e.g. Testing Center>
gap_type: bug | limitation | feature-request | docs-gap
severity: blocker | workaround-exists | inconvenience
customer_impact: adoption-blocking | deal-risk | productivity-loss | confidence-loss   # may be a comma-separated list
repro_status: reproduced-live | reported-only | intermittent | not-reproducible
root_cause_known: true | false
root_cause: <one line>
asks: [<primary>, <alternative>]
workaround: <one line | none>
evidence_doc: <url>
open_questions: [<…>]
notes: <optional>
```

The doc also contains a **"Questions a PM will ask"** section with pre-answered follow-ups and explicitly flagged open questions — read it before drafting your own questions; most will already be answered.

**Fallback:** older or hand-written docs may lack a record block. Parse the prose instead and infer the same fields; mark inferred values as `(inferred)` in any rollup so they aren't mistaken for FDE-asserted facts.

## The triage_record schema (v1) — what the PM side emits

The FDE→PM direction is machine-readable (`feedback_record`); the PM→FDE direction must be too. Every triage disposition comment ends with exactly one fenced ```yaml block:

```yaml
triage_record: v1
issue: AFP-###
date: YYYY-MM-DD
disposition: planned | shipped | wontfix | duplicate | needs-evidence   # exactly one — terminal states of the loop
accepted_asks:
  - <which of the record's asks were accepted, verbatim; empty list if none>
declined_asks:
  - <asks not taken, with a short "why" in parens; omit if none>
duplicate_of: <AFP-### — only when disposition: duplicate>
target: <epic/project/release the work lands in, or "unscheduled">
owner: <who carries the next action — a person, "FDE", or "PM">
next_action: <one line — the single next concrete step>
loop_status: closed | open-with-FDE | open-with-PM | open-with-customer   # who the ball is with
decided_date: YYYY-MM-DD
notes: <optional nuance>
```

**Enum meanings:**
- `disposition` — `planned` (accepted, scheduled on roadmap/epic), `shipped` (fix released), `wontfix` (declined, reason required in the comment), `duplicate` (fold into `duplicate_of`), `needs-evidence` (parked; ball with FDE until evidence arrives).
- `loop_status` — `closed` (all open_questions resolved AND disposition committed to issue fields AND stated on thread), else `open-with-<role>` naming who owes the next move.

## Procedure

### A. Understand one feedback item
1. `get_issue AFP-###` — read the intake description and note the customer/feature fields.
2. `list_comments` on the issue — find the summary comment and any thread discussion since.
3. The issue's `documents` list names its evidence docs — `get_document` each; read TL;DR → verdict → root cause → asks → PM-questions section → the record block.
4. Answer the PM's question **with citations**: issue ID, doc title + URL, and section. Distinguish FDE-asserted facts from your inferences.
5. If the PM has follow-ups the doc doesn't answer: check the doc's *open questions* first (the FDE may have flagged exactly that), then draft the questions and — on the PM's confirmation — post them as a `save_comment` on the issue thread.

### B. Aggregate across many items
1. Enumerate candidates: `list_issues` with `team: "Agentforce Feedback"` (filter by `createdAt`/`updatedAt` for a time window, or `query` for a feature keyword). Page through — don't stop at the first page.
2. For each issue: `get_issue` → `get_document` its evidence doc(s) → extract the `feedback_record` block (fallback: infer from prose, mark `(inferred)`).
3. Build the rollup in whatever cut the PM asked for — by `feature_family`, `gap_type`, `severity`, `customer_impact`, `customer`, or recurring `asks`. Count items, list the issue IDs behind every number, and surface: **blockers first**, then adoption-blocking/deal-risk impacts, then clusters of identical asks (same ask from N customers = strongest signal).
4. Deliver as chat, or — if the PM wants it shareable — as a Linear Document (`save_document` on the PM's chosen project/team/issue) using the rollup template below.
5. Collect every record's `open_questions` into a "needs FDE input" list; offer to post each back to its issue thread as a comment.

### C. Close the loop asynchronously
- Post PM follow-ups as comments on the specific issue (`save_comment` with `issueId`) — the FDE answers on the thread.
- If the PM disposition is "accepted / roadmap / duplicate-of-X / needs-more-evidence", offer to post that as a comment too, so the FDE isn't left waiting.

### D. Record the decision on the issue
1. Draft the disposition (using the enums above) and show it to the PM. **Only on the PM's explicit confirmation** (this preserves the "don't silently mutate" guardrail — see Guardrails below) apply it to the issue itself via `save_issue`: set `state` (e.g. Backlog → Planned/Canceled per disposition), `priority`, and add a label for the feature family if one exists. The issue's own fields are the source of truth; the comment is the rationale.
2. Post the disposition comment **threaded under the FDE's summary comment** (`save_comment` with `parentId: <FDE comment id>`), ending with the fenced `triage_record: v1` block.
3. If `disposition: planned`, link the target epic/project in the comment; if `wontfix`, state the reason in prose above the record; if `duplicate`, also post a one-liner on the duplicate target issue pointing back.
4. Report what was changed on the issue and what the record says.

## When is the loop closed?

- **Terminal states:** `planned`, `shipped`, `wontfix`, `duplicate` are **closed**; `needs-evidence` is **parked** (open, ball with FDE).
- **The three-part closure test** (all must hold): (1) every `open_questions` entry from the `feedback_record` is answered on the thread; (2) the disposition is committed to the issue's own fields (state/priority/label), not just prose; (3) a closing comment states it so CS/FDE/customer can see the outcome.
- **Convergence rule:** target ≤ 2 FDE↔PM round-trips. Each round must reduce the count of open questions; if a round *adds* more than it closes, or two rounds pass without convergence, say so and recommend the PM book a sync — the skills eliminate routine meetings, not genuine deadlocks.

## Rollup document template

```markdown
> **Feedback rollup** — <scope: time window / feature / team> · generated <date> · <N> items reviewed

## Headlines
- <the 2–4 things a PM must know, each citing issue IDs>

## By severity
| Severity | Count | Issues |
|---|---|---|
| blocker | n | AFP-…, AFP-… |
| workaround-exists | n | … |
| inconvenience | n | … |

## By feature / gap type
| Feature | bug | limitation | feature-request | docs-gap |
|---|---|---|---|---|
| <family> | n | n | n | n |

## Recurring asks (strongest signal)
1. **<ask>** — requested in AFP-…, AFP-… (<customers>)
2. …

## Needs FDE input (open questions found in records)
- AFP-### — <question>

## Item index
| Issue | Customer | Feature | Gap | Severity | Impact | Repro | Ask (primary) |
|---|---|---|---|---|---|---|---|
| AFP-### | … | … | … | … | … | … | … |

*(values marked `(inferred)` were parsed from prose, not FDE-asserted records)*
```

## Guardrails

- **Cite everything.** Every claim in an answer or rollup traces to an issue/doc. No uncited aggregates.
- **Don't silently mutate.** Never change issue state, priority, assignee, or labels unless the PM explicitly asks.
- **Respect the record.** Where a record and your prose-reading disagree, the record wins; note the discrepancy.
- **Confidentiality.** Rollups may name customers — treat outputs as internal; don't paste them to external channels.
- **Every hand-off names an owner.** Follow-up questions posted to the thread must say who owes the answer (`loop_status: open-with-FDE` / `open-with-customer`); a question that requires customer input must be flagged `(customer-dependent)` so the FDE knows to schedule a customer touch rather than answer from the desk.

## Worked example

Two filled instances to imitate — a single-item triage, and the cross-feature rollup that shows this skill spans **all** Agentforce feedback (Testing Center below is just the example, not the scope).

### 1. Single-item triage (Procedure A + D)

**Scenario:** the PM asks their agent *"What's AFP-### about, and draft my disposition."* The agent reads the intake → summary comment → evidence doc → parses the `feedback_record`. (Same demo item as the FDE skill's worked example — a multi-turn Testing Center gap, customer `internal-demo` — so the two examples visibly connect as one loop.)

**Triage verdict**

| Dimension | Verdict |
|---|---|
| Classification | `limitation` — agree with the FDE, not a bug |
| Severity | `workaround-exists` — but the workaround (manual `sf agent preview` runs) isn't CI-viable |
| Impact | `confidence-loss, productivity-loss` |
| Repro | `reproduced-live`, root cause proven via the Data Cloud trace — evidence quality high, no further validation needed |

**Disposition:** Accepted. The near-term ask — warn when a case depends on a value not present in the transcript — ships fastest since it's detectable at authoring time. The execution-model ask — persist action-output state by executing prior turns — is the real fix but heavier; routing it to the multi-turn testing epic for sizing. The documentation ask (state the replay limitation so PASS/FAIL is trusted correctly) is immediate and ships alongside the warning.

```yaml
triage_record: v1
issue: AFP-###
date: 2026-07-20
disposition: planned
accepted_asks:
  - Warn when a multi-turn case depends on a value not present in the transcript (near-term, authoring-time check)
  - Document the state-replay limitation so PASS/FAIL is trusted correctly (immediate)
target: multi-turn testing epic
owner: FDE
next_action: FDE confirms the warning is feasible at authoring time and answers whether other customers have silently-passing stateful suites
loop_status: open-with-FDE
decided_date: 2026-07-20
notes: Execution-model ask (persist action-output state) accepted in principle; sized separately in the epic, not blocking the near-term ship
```

On the PM's confirmation, the agent applied the disposition to `AFP-###` itself per Procedure D: `state` → Planned, `priority` raised, and label `testing-center` added.

### 2. Cross-feature rollup (Procedure B)

**Scenario:** *"Roll up this month's Agentforce feedback."* The agent enumerates issues across every feature family, not just Testing Center:

| Issue | Customer | Feature | Gap | Severity | Impact | Repro | Ask (primary) |
|---|---|---|---|---|---|---|---|
| AFP-101 | Acme Energy | Testing Center | limitation | workaround-exists | confidence-loss | reproduced-live | Execute prior turns in multi-turn tests |
| AFP-102 | Northwind Bank | Agent Script | feature-request | inconvenience | productivity-loss | reported-only | Loop constructs in Agent Script |
| AFP-103 | Contoso Health | Voice | bug | blocker | adoption-blocking | reproduced-live | Barge-in cuts off variable capture |
| AFP-104 | Acme Energy | Agent Builder | docs-gap | inconvenience | confidence-loss | reported-only | Document topic-routing precedence |

**Headlines:**
- One `blocker` leads the list (AFP-103, Voice barge-in — adoption-blocking, reproduced live).
- `Acme Energy` appears twice (AFP-101, AFP-104) — a repeat reporter across two unrelated feature families.
- Every value above was parsed straight from `feedback_record` blocks — zero prose interpretation.

Same table, any feature family — the record contract is what makes the rollup possible.
