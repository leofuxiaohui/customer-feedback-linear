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
tracking:                       # register ID -> delivery work issue; added at planned/shipped
  A2: <issue-id or url>
epic: <roadmap parent name/url, or "unscheduled">
owner: <who carries the next action — a person, "FDE", or "PM">
next_action: <one line — the single next concrete step>
loop_status: closed | open-with-FDE | open-with-PM | open-with-customer   # who the ball is with
decided_date: YYYY-MM-DD
notes: <optional nuance>
```

**Enum meanings:**
- `disposition` — `planned` (accepted, scheduled on roadmap/epic), `shipped` (fix released), `wontfix` (declined, reason required in the comment), `duplicate` (fold into `duplicate_of`), `needs-evidence` (parked; ball with FDE until evidence arrives).
- `loop_status` — `closed` (all open_questions resolved AND disposition committed to issue fields AND stated on thread), else `open-with-<role>` naming who owes the next move.

`tracking` is filled once the decision becomes tracked work (Procedure E); before that, omit it.

## Procedure

### The register (ask/answer ledger)

Every **item** in the loop — an ask, a question, a follow-up, a decision — has a stable **ID** and a **status token**, and lives in a compact **register table**, never as a loose bullet. Prose around the table carries the reasoning. Every round comment ends with the register (see *Keep the register readable*).

| Item | Status | Next |
|---|---|---|
| **A1** Execute prior turns to persist state | ⏳ open · R1 | PM |

- **Item** — the ID as a **bold prefix**, then a ≤6-word label (`**A1** Execute prior turns…`). Bold the ID **only in the round that item changed** — an at-a-glance delta marker; leave it unbolded in later rounds where it's unchanged.
- **Status** — the token, then the round it last changed (`⏳ open · R1`).
- **Next** — what happens next for this item, never blank: a **name** (`PM` / `FDE` / `FDE (customer)`) when someone owes the move · `✓ done` when resolved · `→ AFP-##` once it's tracked in delivery · `→ roadmap` when accepted but delivery isn't spun up yet.

**Status tokens:** ⏳ open · 📋 accepted · 🔎 to size *(accepted in principle, **sizing still pending** — see the guardrail: never let a token claim a step that hasn't happened)* · 🔴 blocked *(needs customer / external input)* · ✅ resolved · ❌ declined *(reason in prose)*

**ID prefixes:** `A#` = FDE ask · `Q#` = FDE evidence-limit question (about the boundary of the FDE's own investigation) · `F#` = PM follow-up to the FDE · `P#` = PM-owned triage action (e.g. a telemetry pull)

IDs are the join key to the machine records: an `A1` in the register is the same `A1` referenced in `feedback_record`/`triage_record`.

#### Keep the register readable — the rolling snapshot

**Every round comment ends with the full register** — all items so far, in the 3-column table above — as its last-numbered section (`## N · 📒 Register — after Round N (<side>)`). Because a Linear thread reads top-down, **the newest comment's register is always the live state** — there's no separate ledger comment to maintain, and each round's snapshot is a frozen audit record. Don't post delta-only registers, and don't keep a standalone "current state" comment at the end of the thread.

Above the table, a one-line **hand-off header**: `**ball: 🔵 PM** · open: 2 (F1, F2)` — *ball* is the side that owes the next move (🟢 FDE, 🔵 PM); list PM-only self-owned items (e.g. a telemetry pull) separately as `+1 PM-side`. Below the table, two things: a one-line **Reading it** guide, then a **Status legend** with one token per line and its meaning, so a cold reader needs nothing else:

*Reading it: **bold ID** = changed this round · **Next** = a name (whose move) / ✓ done (resolved) / → AFP-## (tracked in delivery) / → roadmap (accepted, delivery spun up at close) · IDs join `feedback_record` / `triage_record`.*

**Status legend**
- ⏳ **open** — awaiting a decision; the ball is with whoever's named in **Next**.
- 📋 **accepted** — committed as a near-term deliverable (scoped small enough to schedule now).
- 🔎 **to size** — accepted in principle and *will* be built, but not yet scheduled or estimated; sits in the epic backlog until it gets a size.
- 🔴 **blocked** — can't proceed without external / customer input (named on the item).
- ✅ **resolved** — answered or closed from evidence or mechanism; no further action.
- ❌ **declined** — not being pursued; the reason is in the item's *why*.

**Three columns, no more.** Markdown has no column-width syntax, and Linear splits table width **evenly by column count** — so every extra column starves the Item text (a 5-column register shreds the description into 3 lines). Keep the register to `Item · Status · Next`; never add `Owner`/`Since`/`ID` back as their own columns. Keep each Item label ≤~6 words; if a cell needs a clause to be understood, that clause belongs in the prose, not the table.

**Rationale lives in prose, not the table.** The register is the index; the **item cards** in the numbered sections are the content — each ask/answer/question is a **blockquote card** laid out **Ask → (decision) → Why**, per *Comment cards* below.

### Comment cards — Ask → Why

Inside a round's item sections, each item is a **blockquote card**, never a bare list row. It reads top-down and ends in a **bulleted `why`** so a human can audit the call and keep or override it.

**FDE ask** — asker's side, no decision yet:

> **A1 · Ask:** <the ask, one line>
> → <one-line framing: weight / role among the asks>
>
> *why —*
> - **From evidence:** <the chain from the evidence to this ask>
> - **Why A1 and A2/A3:** <relation to the other asks — layered fallbacks, not duplicates>
> - **Grounded in:** <the proof — say **direct evidence** vs **inference** explicitly>
> - **Confidence:** <high/low, and on what>
> - **Would soften if:** <the condition that would change it>

**PM answer** — decider's side, restate the ask then decide:

> **A1 · Ask:** <restate the FDE's ask, one line>
> **Proposed decision:** <token> <label> — <one-line reason>
>
> *why —*
> - **What "<token>" commits to:** <esp. for 🔎 to size — what it does and doesn't promise>
> - **Over the alternatives:** <accept-now ✗ · decline ✗ · chosen ✓ — why this over those>
> - **Grounded in:** <proof vs inference>
> - **Confidence:** <…>
> - **Would flip to <other decision> if:** <the trigger>

**Question** — a Q# (FDE evidence-limit), F# (PM follow-up), or P# (PM action) has no decision:

> **F1 · ⏳ open · Next: FDE** — <the question>
> → <one line: what the answer decides>
>
> *why it matters —*
> - <what hinges on the answer>
> - **Grounded vs inferred:** <mark what's proven vs assumed>
> - <who can answer, and why it's theirs>

**Card rules:**
- **Grounded vs inferred is mandatory** wherever a claim could be either — a human overrides an *inference* readily but hesitates on a *proven* fact, so the reasoning must say which. (This is the anti-fabrication guardrail applied to reasoning.)
- **Bulleted lists render inside blockquote comment cards** (`> - item`) — use them for the `why`. **`<details>`/`<summary>` do NOT render in Linear comments** — never put a collapsible in a comment (it shows as raw tags; collapsibles render only in Documents).
- **State the proposal/override note once per section, not per item** — see below.

### The proposal & override note (once per section)

Every item in an asks/answers/new-items section is the **agent's proposal**; the human keeps it by posting and can override later. State this **once**, in the section's intro line, prefixed with 🤖:

- **FDE asks:** `🤖 *Every item below is the **agent's proposal** — the FDE can **override** any of them; an item shows an override note only once that happens.* Each is laid out **Ask → Why**, with the reasoning bulleted.`
- **PM answers:** `🤖 *Every decision below is the **agent's proposal** — the PM can **override** any of them; an item shows an override note only once that happens.* Each is laid out **Ask → Proposed decision → Why**, with the reasoning bulleted.`
- **PM new items:** `🤖 *Items the triage opened — follow-up questions to the FDE (**F#**) and a PM-owned action (**P#**); agent-proposed, the PM can drop or reword any.* Each is laid out **Question → Why**, with the reasoning bulleted.`

Never repeat a per-item "proposed · keep · override?" marker on every card — it's noise. A marker appears on an individual item **only when a human actually overrides it** (that item then carries `✏️ overridden by <name>: <new call>`, and the change appears as the next register snapshot — never a silent edit).

### Comment structure (every loop comment)

Each comment in the loop is a self-contained, scannable unit:

1. **Header envelope** — three lines up top:
   > `### <ball emoji> Round N · <FROM> → <TO>`
   > `**Purpose:** <one line> · **Ball after:** <emoji> <role> · **Open:** <count>`
   > `*🤖 via `<skill-name>` · <attribution>*`

   The ball emoji is the author's colour — **🟢 FDE**, **🔵 PM**. The `🤖` line is a small italic caption (who/what authored the comment), never the headline.
2. **Numbered sections** — `## 1 · … ## 2 · …`, so the comment reads as an outline and sections are stable references (`§3`). The **📒 Register is always the last-numbered section**. Standard running order:
   - **FDE Round 1:** 1 · What I found · 2 · Repro & results · 3 · Root cause · 4 · Asks & Questions · 5 · Register
   - **FDE later rounds:** 1 · Answers · 2 · Record impact · 3 · Register
   - **PM triage round:** 1 · Triage verdict · 2 · Answers · 3 · New items opened this round · 4 · Disposition · 5 · Register
   - **PM close:** 1 · Decision · 2 · Delivery (Procedure E) · 3 · Closure test · 4 · For CS · 5 · Register

The whole thread also gets a header block so a cold reader orients before any round — see *The thread header*.

### The thread header (issue description)

So a human landing on the issue orients in seconds, the feedback issue's **description** opens with a loop-status block, kept above a `---` divider that preserves the CS intake untouched:

```markdown
## 🧵 Feedback loop — <short title>

**Customer:** <name> · **Feature:** <family>
**Status:** <triage loop open | ✅ closed> · disposition <…> · issue in **<state>**
**Rounds:** <n>

**How to read:** each comment is one round (FDE ⇄ PM); every round is numbered and ends with a **📒 Register** snapshot — the newest register is the live state. Machine records: `feedback_record` (in the linked doc) · `triage_record` (in the close comment).

---

<the original CS intake — never edited>
```

The **FDE** adds this block when first publishing feedback (via `save_issue` with `description:`, prepending it above the intake); the **PM** refreshes `Status`/`Rounds`/deliverables at disposition and close. **Only ever touch the block above the divider** — the intake below it belongs to CS. This is the one time the loop edits the description; everything else is comments.

### A. Understand one feedback item
1. `get_issue AFP-###` — read the intake description and note the customer/feature fields.
2. `list_comments` on the issue — find the summary comment and any thread discussion since.
3. The issue's `documents` list names its evidence docs — `get_document` each; read TL;DR → verdict → root cause → asks → PM-questions section → the record block.
4. Answer the PM's question **with citations**: issue ID, doc title + URL, and section. Distinguish FDE-asserted facts from your inferences.
5. If the PM has follow-ups the doc doesn't answer: check the doc's *open questions* first (the FDE may have flagged exactly that), then draft the questions as `F#` items (Next = FDE, or FDE (customer) if customer-dependent) — not loose bullets. They appear as rows in the **trailing register snapshot** of the round comment (posted per *Comment structure*), plus as **blockquote cards** (Question → Why) in the New-items section, per *Comment cards*, and, on the PM's confirmation, post them as a `save_comment` on the issue thread.
6. **Prevalence is a PM question.** When triaging, raise blast-radius / prevalence questions yourself as `P#` register items (e.g. "pull telemetry on how many suites are affected") — do not expect the FDE to supply prevalence; the FDE only has their one case.

### B. Aggregate across many items
1. Enumerate candidates: `list_issues` with `team: "Agentforce Feedback"` (filter by `createdAt`/`updatedAt` for a time window, or `query` for a feature keyword). Page through — don't stop at the first page.
2. For each issue: `get_issue` → `get_document` its evidence doc(s) → extract the `feedback_record` block (fallback: infer from prose, mark `(inferred)`).
3. Build the rollup in whatever cut the PM asked for — by `feature_family`, `gap_type`, `severity`, `customer_impact`, `customer`, or recurring `asks`. Count items, list the issue IDs behind every number, and surface: **blockers first**, then adoption-blocking/deal-risk impacts, then clusters of identical asks (same ask from N customers = strongest signal).
4. Deliver as chat, or — if the PM wants it shareable — as a Linear Document (`save_document` on the PM's chosen project/team/issue) using the rollup template below.
5. Collect every record's `open_questions` into a "needs FDE input" list; offer to post each back to its issue thread as a comment.

### C. Close the loop asynchronously
- Post PM follow-ups as `F#` items (Status ⏳ open) in a comment on the specific issue (`save_comment` with `issueId`) — they appear as rows in the comment's trailing register snapshot, with the reasoning as **blockquote cards** (Question → Why), per *Comment cards*. The FDE answers on the thread by updating the row's status in their reply's own snapshot.
- If the PM disposition is "accepted / roadmap / duplicate-of-X / needs-more-evidence", offer to post that as a comment too, so the FDE isn't left waiting.

### D. Record the decision on the issue
1. Draft the disposition (using the enums above) and show it to the PM. **Only on the PM's explicit confirmation** (this preserves the "don't silently mutate" guardrail — see Guardrails below) apply it to the issue itself via `save_issue`: set `state` (e.g. Backlog → Planned/Canceled per disposition), `priority`, and add a label for the feature family if one exists. The issue's own fields are the source of truth; the comment is the rationale.

   | `disposition` | Linear issue state |
   |---|---|
   | `planned` | Todo (or In Progress once work starts) |
   | `shipped` | Done |
   | `wontfix` | Canceled |
   | `duplicate` | Duplicate |
   | `needs-evidence` | Triage |

   Also set **`assignee`** to the current owner of the next action, so the board shows who's up — reassign as ownership moves across rounds.
2. Post the disposition comment **threaded under the FDE's summary comment** (`save_comment` with `parentId: <FDE comment id>`), ending with the fenced `triage_record: v1` block.
3. If `disposition: planned`, link the target epic/project in the comment; if `wontfix`, state the reason in prose above the record; if `duplicate`, also post a one-liner on the duplicate target issue pointing back.
4. Report what was changed on the issue and what the record says.

### E. From decision to delivery (close the real loop)

A `planned`/`shipped` disposition is a decision, not a delivery. The loop only truly closes when the decision becomes **tracked work** and the outcome reaches the customer. **Only on the PM's explicit confirmation** — same guardrail as Procedure D — before creating any issues:

1. **Spin up tracked work** — one work issue per accepted deliverable, titled with its register ID (`[AFP-###·A2] …`), created in the delivery team's epic/project. Feedback issues live in `eventsmobileapp`; eng work lives in the dev workspace and **sub-issues cannot cross workspaces** — so create the delivery issues in the eng team and **cross-link by URL** (a Linear relation), or, when tracking inside the feedback team, create them as sub-issues of the feedback issue. State which you did.
2. **Record the mapping** in the `triage_record` `tracking:` block (register ID → work issue) and set `epic:`.
3. **Roll up** — when every tracked child is Done, move the feedback issue to Done.
4. **Close back to the customer** — post a short comment addressed to CS summarizing what shipped (by register ID), so CS can relay it to the customer. This is the real close; a disposition alone leaves the customer uninformed.

## When is the loop closed?

- **Two different closures.** The **FDE↔PM triage loop** closes when open items resolve and a disposition is committed to the issue; **the feedback is fully closed** only when the tracked work ships and the customer is informed (Procedure E). Terminal `disposition` states remain as defined below; a `planned` disposition implies delivery tracking follows via Procedure E.
- **Terminal states:** `planned`, `shipped`, `wontfix`, `duplicate` are **closed**; `needs-evidence` is **parked** (open, ball with FDE).
- **The three-part closure test** (all must hold): (1) every `open_questions` entry from the `feedback_record` is answered on the thread; (2) the disposition is committed to the issue's own fields (state/priority/label), not just prose; (3) a closing comment states it so CS/FDE/customer can see the outcome.
- **Converge, don't count.** Each exchange must *shrink the open set* in the register. If open items aren't trending to zero — or a round adds more than it closes — that's the signal to escalate to a sync. The skills remove *routine* meetings, not genuine deadlocks. (Most items settle in one or two exchanges; treat that as observation, never a pass/fail bar.)

## Rollup document template

Per-item registers posted in issue comments follow the rolling-snapshot conventions (labels, not sentences) — see *Keep the register readable* above. The rollup below is a different artifact (a cross-issue index for a PM audience), and its columns may stay tabular.

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
- **Prevalence is yours.** How-many-customers / how-widespread questions are PM triage actions (`P#`), never FDE asks — the FDE sees one customer.
- **Never invent specifics.** Don't manufacture customer detail, quantified impact, or telemetry to make a triage look complete. If a number isn't known, say so and make obtaining it a `P#` action. Any illustrative/simulated value must be labelled — and preferred qualitative over a fake-precise figure.
- **A token must not claim a step that hasn't happened.** `🔎 to size` means accepted-in-principle with sizing pending, not estimated; don't upgrade an item's token past its real state.

## Worked example

Two filled instances to imitate — a single-item triage, and the cross-feature rollup that shows this skill spans **all** Agentforce feedback (Testing Center below is just the example, not the scope).

### 1. Single-item triage (Procedure A + D)

**Scenario:** the PM asks their agent *"What's AFP-### about, and draft my disposition."* The agent reads the intake → summary comment → evidence doc → parses the `feedback_record`. (Same demo item as the FDE skill's worked example — a multi-turn Testing Center gap, customer `internal-demo` — so the two examples visibly connect as one loop.)

> ### 🔵 Round 1 · PM → FDE
> **Purpose:** triage + answer A1–A3/Q1, open F1–F2 + P1 · **Ball after:** 🟢 FDE · **Open:** 2 (+1 PM-side)
> *🤖 via `PM-feedback-triage-linear` · triage of AFP-###*
>
> ## 1 · Triage verdict
> `limitation` (agree, not a bug) · severity `workaround-exists` (manual `sf agent preview` isn't CI-viable) · impact `confidence-loss, productivity-loss` · `reproduced-live`, root cause proven (1-vs-0 `VARIABLE_UPDATE_STEP`) — evidence quality high, no further validation needed our side.
>
> ## 2 · Answers — A1–A3, Q1
>
> 🤖 *Every decision below is the **agent's proposal** — the PM can **override** any of them; an item shows an override note only once that happens.* Each is laid out **Ask → Proposed decision → Why**, with the reasoning bulleted.
>
> > **A1 · Ask:** Execute prior turns to persist state.
> > **Proposed decision:** 🔎 to size — the real fix, but an execution-model change; deferred to the epic, doesn't gate the near-term ship.
> >
> > *why —*
> > - **What "to size" commits to:** accepted as valid and will be built — but not yet scheduled or estimated; it enters the multi-turn epic backlog and gets a size before any sprint commit.
> > - **Over the alternatives:** accept-now ✗ (engine change, too heavy to commit unsized) · decline ✗ (it's the real fix) · route-to-epic ✓ (accept the intent, defer scheduling).
> > - **Grounded in:** the FDE's Data Cloud trace — mechanism proven.
> > - **Confidence:** high the fix is right, low on effort (unsized).
> > - **Would flip to accept-now if:** P1 telemetry shows a large share of suites hit this → escalate A1 out of the epic.
> >
> > **A2 · Ask:** Warn when a case depends on a value not present in the transcript.
> > **Proposed decision:** 📋 accepted — near-term ship; detectable at authoring time, no execution-model change. First deliverable.
> >
> > *why —*
> > - **Over the alternatives:** the cheapest thing that cuts the pain immediately — additive, no engine risk.
> > - **Grounded in:** statically detectable at authoring time.
> > - **Confidence:** high, small scope.
> > - **Role:** the near-term cover that lets A1 wait for sizing.
> >
> > **A3 · Ask:** Document the replay limitation.
> > **Proposed decision:** 📋 accepted — immediate; ships alongside A2.
> >
> > *why —*
> > - **Over the alternatives:** zero-cost, ships with A2 — no reason to defer.
> > - **Grounded in:** the false-confidence finding (the FDE's §3).
> > - **Confidence:** high.
> >
> > **Q1 · Question:** Voice channel not yet re-run — is that a blocker?
> > **Proposed decision:** ✅ resolved — not blocking; the mechanism is channel-independent, so a voice re-run wouldn't change the plan.
> >
> > *why —*
> > - **Grounded vs inferred:** rests on an **inference from the mechanism** (final-turn-only replay would fail voice the same way), not a voice observation — flagged so the FDE can push back.
> > - **Would reopen if:** anyone wants empirical voice confirmation before roadmap → becomes a 🔴 blocked customer touch.
>
> ## 3 · New items opened this round
>
> 🤖 *Items the triage opened — follow-up questions to the FDE (**F#**) and a PM-owned action (**P#**); agent-proposed, the PM can drop or reword any.* Each is laid out **Question → Why**, with the reasoning bulleted.
>
> > **F1 · ⏳ open · Next: FDE** — Case mix: verify-then-act vs action-output-dependent?
> > → Which kind of stateful case dominates the customer's suite? The answer sets A1's urgency.
> >
> > *why it matters —*
> > - **Two kinds, from the FDE's evidence:** *re-derivable from transcript* (like S1/S3 — value sits in history text, reconstructed; these **survive**) vs *action-output-dependent* (like S2 — value existed only as a prior action's output, never in the transcript; these **break**).
> > - **What the answer changes:** mostly S1/S3 → the A2 warning covers most of the pain and A1 can wait; mostly S2 → the warning only *flags* the gap, so A1 (or a seeding stopgap) becomes urgent.
> > - **Why the FDE, not me:** they've reviewed the customer's actual flows; I'm triaging from one evidence pack.
> >
> > **F2 · 🔴 blocked · Next: FDE (customer)** — Appetite for a "seed session variables" beta?
> > → Would the customer adopt a stopgap that seeds `customer_name` etc. as test inputs while A1 is designed?
> >
> > *why it matters —*
> > - **What it unlocks:** seeding covers the S2 class for regression **without** re-executing prior turns — a bridge until A1 ships.
> > - **Why ask before building:** a stopgap only earns its cost if the customer will actually adopt it.
> > - **Why blocked:** needs a customer touch — can't be answered from the desk.
> >
> > **P1 · ⏳ open · Next: PM** — Telemetry: blast-radius of affected suites.
> > → How many suites use `conversationHistory` with assertions on prior-turn action outputs?
> >
> > *why it matters —*
> > - **What it decides:** prevalence sets overall priority — and it's the trigger that could pull A1 out of the epic (see A1's flip-condition).
> > - **Why it's mine, not an FDE ask:** prevalence is fleet-wide; the FDE has one customer's evidence.
> > - **Non-blocking:** the disposition doesn't wait on it.
>
> ## 4 · Disposition (preliminary)
> **Accepted — advancing to roadmap review.** The strongest argument is the **false-confidence finding**: S1/S3 pass only because credentials sat in the transcript, so a green stateful suite can prove nothing. Final disposition follows once F1/F2 close.
>
> ## 5 · 📒 Register — after Round 1 (PM)
> **ball: 🟢 FDE** · open: 2 (F1, F2) + 1 PM-side (P1, non-blocking)
>
> | Item | Status | Next |
> |---|---|---|
> | **A1** Execute prior turns to persist state | 🔎 to size · R1 | → roadmap |
> | **A2** Warn on transcript-absent dependency | 📋 accepted · R1 | → roadmap |
> | **A3** Document the replay limitation | 📋 accepted · R1 | → roadmap |
> | **Q1** Voice channel not yet re-run | ✅ resolved · R1 | ✓ done |
> | **F1** Case mix: S2-class share? | ⏳ open · R1 | FDE |
> | **F2** Seeding-beta appetite? | 🔴 blocked · R1 | FDE (customer) |
> | **P1** Telemetry blast-radius | ⏳ open · R1 | PM |
>
> *Reading it: **bold ID** = changed this round · **Next** = a name (whose move) / ✓ done (resolved) / → AFP-## (tracked in delivery) / → roadmap (accepted, delivery spun up at close) · IDs join `feedback_record` / `triage_record`.*
>
> **Status legend**
> - ⏳ **open** — awaiting a decision; the ball is with whoever's named in **Next**.
> - 📋 **accepted** — committed as a near-term deliverable.
> - 🔎 **to size** — accepted in principle and will be built, but not yet scheduled or estimated.
> - 🔴 **blocked** — can't proceed without external / customer input.
> - ✅ **resolved** — answered or closed; no further action.
> - ❌ **declined** — not being pursued; reason in the item's *why*.

```yaml
triage_record: v1
issue: AFP-###
date: 2026-07-20
disposition: planned
accepted_asks:
  - Warn when a multi-turn case depends on a value not present in the transcript (A2 — near-term, authoring-time check)
  - Document the state-replay limitation so PASS/FAIL is trusted correctly (A3 — immediate)
target: multi-turn testing epic
owner: FDE
next_action: FDE answers F1 (case mix) and F2 (seeding-beta appetite — customer-dependent); PM runs the P1 telemetry pull
loop_status: open-with-FDE
decided_date: 2026-07-20
notes: Execution-model ask (A1) accepted in principle; routed to epic, sizing pending — not blocking the near-term ship. A material share of multi-turn suites likely depend on prior-turn action outputs; exact count pending the P1 telemetry pull — no hard number yet.
```

On the PM's confirmation, the agent applied the disposition to `AFP-###` itself per Procedure D: `state` → Todo (per the disposition→state mapping, `planned` → Todo), `priority` raised, label `testing-center` added, and `assignee` set to the FDE — the current owner of A2's next action.

Then, per Procedure E: once A2 and A3 shipped, the agent created delivery issues in the eng team (sub-issues can't cross workspaces from `eventsmobileapp`) and cross-linked them by URL, then updated the `triage_record`:

```yaml
tracking:
  A2: AFL-412
  A3: AFL-413
epic: multi-turn testing epic
```

...and posted a close-back comment addressed to CS on `AFP-###` noting A2 and A3 shipped.

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
