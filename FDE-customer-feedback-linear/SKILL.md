---
name: FDE-customer-feedback-linear
description: Use when an FDE or Solution Engineer wants to share detailed Agentforce customer feedback via Linear — attach it to the customer-feedback intake issue (AFP-*) as one self-contained in-app Linear Document plus a concise summary comment on the thread, so PMs review everything without downloading anything and without needing a meeting. The doc includes an anticipated-PM-questions section and a machine-readable feedback-record block that the companion PM-feedback-triage-linear skill (a PM's coding agent) can parse for cross-customer insights. Works for any feature (Testing Center, Agent Script, Voice, Builder, etc.). Also handles round-2: answering PM follow-ups on the thread until the loop closes with a recorded PM disposition.
---

# FDE Customer Feedback → Linear

A standalone skill dedicated to sharing **Agentforce customer feedback** via Linear. Package detailed feedback into a target customer-feedback issue as **(1)** one self-contained **in-app Document** (the deep dive) and **(2)** a concise **summary comment** on the thread that links the doc. Everything stays inside Linear — no attachments to download.

**Design goal: no meeting needed.** The doc must let a PM (or a PM's coding agent, via the companion `PM-feedback-triage-linear` skill) fully understand and triage the feedback asynchronously. Two features serve that goal:
- a **"Questions a PM will ask"** section that pre-answers the follow-ups a PM would otherwise schedule a call for, and explicitly flags what the FDE/SE could *not* answer;
- a **machine-readable `feedback-record` block** (fixed vocabulary, defined below) at the end of every doc, so PM agents can parse and aggregate feedback across many issues reliably.

This skill handles **packaging and posting**. It does *not* gather the evidence (that is feature-specific) — the FDE/SE brings the evidence; you structure and publish it.

## When to use

- The FDE/SE has evidence for a product gap / feature request / bug and a target Linear issue to enrich (usually an `AFP-###` in the Agentforce Feedback intake).
- Goal: make the feedback triage-ready — clear narrative, expected-vs-actual, root cause if known, business impact, and a concrete ask — reviewable in-app.

Do **not** use this to run tests or collect traces, or to file the initial intake issue.

## Inputs to collect from the user (ask for anything missing before drafting)

1. **Target issue** — `AFP-###` or a Linear issue URL.
2. **Feature area** — e.g. Testing Center, Agent Script, Voice.
3. **Evidence bundle** — what was tried, **observed vs expected**, supporting data (transcripts, screenshots, query results, links), root cause if known, business/adoption impact, and the **ask**.

If the evidence is thin, ask targeted questions first. A good doc needs at minimum: a headline finding, a reproducible "what happens", expected-vs-actual, and a concrete recommendation.

## Prerequisites — Linear workspace (check this first)

Feedback issues (`AFP-*`) live in the **`eventsmobileapp`** workspace, team **"Agentforce Feedback"**. The Linear MCP connects to **one workspace at a time**.

1. Verify access with `get_issue <id>`.
2. If it returns *"Could not find referenced Issue"*, the MCP is pointed at the wrong workspace (commonly the `agentforce` dev workspace). Ask the user to run `/mcp` → **linear** → reauthenticate and pick **eventsmobileapp** (log out of the other Linear account in-browser first, or it silently re-grants the same one). Then retry.

## Procedure

1. **Verify + read context.** `get_issue <id>`. Read the intake fields (**Customer Name, Product Feature Family/Group, Feedback Theme, Business Outcome/Use Case**) and tailor the framing so the evidence visibly answers the customer's stated need — quote it where useful. If your evidence **corrects or sharpens the intake's problem statement** (e.g. the intake says "not supported" but the mechanism is "supported but state-lossy"), plan to say so explicitly — the roadmap must aim at the true mechanism, not the first approximation. Set `feature_family` to mirror the intake's **Product Feature Family** (or Feature Group when that's the precise area); if your evidence refines which area it really is, say so in prose rather than silently diverging from the intake taxonomy. Then **prepend the thread-header block to the issue description** (see *The thread header* below) via `save_issue` with `description:`, above a `---` divider — keep the intake verbatim below the divider.
2. **Draft the doc** from the *Evidence document template* below. Fill only the sections that apply. Keep raw data/logs/queries inside collapsible `<details>` blocks so the main narrative stays clean and scannable. Lead with the strongest proof.
3. **Pre-answer the PM.** Generate the *"Questions a PM will ask"* section: put yourself in the triaging PM's seat and answer the standard follow-ups (see the checklist in the template — scope, regression-vs-always, frequency, workaround, quantified impact, what-changes-if-fixed). Where the FDE/SE can't answer from the evidence, ask them once; anything still unanswered goes in **open questions** — flagged, not omitted. This section is what replaces the meeting. **FDE open questions (`Q#`) are about the limits of *your own* evidence** — what your investigation did not cover (e.g. "reproduced on chat; not yet re-run on the customer's voice channel") — **not** product-wide questions like prevalence or roadmap, which are the PM's to answer. If you're tempted to ask "how many customers are affected," that's a PM triage action, not an FDE question.
4. **Emit the `feedback-record` block** (schema below) as the last section of the doc. Derive values from the evidence and intake fields; **use only the fixed vocabulary** for the enum fields — if none fits, use the closest and note the nuance in `notes`. Confirm `severity` and `customer_impact` with the FDE/SE if not obvious — these drive PM prioritization rollups. (`open_questions` here are `Q#` evidence-limit items only — see step 3.)
5. **Create the doc linked to the issue:** `save_document` with `issue: <id>`, a clear `title`, and the markdown `content`.
   - **Do not pass `icon`** unless you know it is a valid Linear icon name — an invalid icon name errors the whole call. Omit it.
   - The create call returns the doc URL. **Update the doc once more** (`save_document` with `id: <doc id>`) to fill `evidence_doc:` in the record block with that URL, so the record is self-locating.
6. **Post the summary comment:** `save_comment` with `issueId: <id>` using the *Summary comment template* below — a TL;DR, a compact results table if applicable, the ask, open questions (if any), and a link to the doc. This is Round 1 of the loop, so it follows *Comment structure*: header envelope up top, numbered sections, and a trailing 📒 register snapshot as the last section.
7. **Binary evidence only** (screenshots, PDFs that must be seen as images): use the attachment flow — `prepare_attachment_upload` (needs the exact byte size) → PUT the raw bytes to the returned signed URL with `curl --data-binary @file`, sending the returned headers **verbatim**, within 60s, **one file at a time** → `create_attachment_from_upload`. Prefer inline docs; attach a binary only when the picture *is* the evidence. (Linear does **not** render HTML attachments inline — never ship an HTML file as the deliverable; put the narrative in the doc.)
8. **Report** the doc URL and confirm the comment posted. State who the ball is with (normally the PM, for triage) and that the loop stays open until the PM posts a disposition (`triage_record`) — see the companion `PM-feedback-triage-linear` skill.

## Conventions & guardrails

- **One consolidated doc per feedback item** — not several small ones. If you split, cross-link them; better yet, fold appendices into the main doc as `<details>` sections.
- **Frame as a design gap / feature request with concrete asks**, not just "it's broken." Give the primary ask, an alternative/interim mitigation, and a documentation/workaround note.
- **Sanitize before posting.** Use mock/sample data; scrub org IDs, customer PII, tokens, and internal endpoints. State explicitly when data shown is mock.
- **Retiring a doc:** the Linear MCP has **no delete/archive-document tool**. To retire a doc, rename it to a `↪ Merged — archive me` stub whose body links the surviving doc, and ask the user to archive it in-app (hover the doc → ⋯ → **Archive**).
- **Markdown, not HTML.** Linear renders markdown tables, code fences, blockquotes, and bulleted lists in both docs and comments; it will not render an HTML file inline. **`<details>`/`<summary>` collapsibles render only in Documents, *not* in comments** — use them to fold raw data in the evidence doc, but never in a thread comment. Comment reasoning uses blockquote cards with bulleted `why` blocks instead — see *Comment cards*.
- **Every hand-off names an owner.** When you post open questions for the PM, say explicitly the ball is with the PM. When answering PM follow-ups, flag any question that needs *customer* input as `(customer-dependent)` — it can't be answered from the desk, and the thread should show that the delay is a customer touch, not FDE silence.
- **Never invent specifics.** Don't fabricate customer detail, channel mechanics, or quantified impact to fill the template. You have one customer's evidence — stay inside it. Mark any illustrative/simulated value explicitly, and prefer "not quantified" over a fake-precise number.
- **Lead each loop comment with the header envelope** (round · direction · purpose · ball · open) and **end it with the 📒 register snapshot** — see *Comment structure* and *Keep the register readable*.
- **A token must not claim a step that hasn't happened.** `🔎 to size` = accepted, sizing pending — not estimated; only use `📋 accepted`/`✅ resolved`/`🔎 to size` when that state is actually true. Prefer under-claiming.
- **The loop isn't done at the PM's disposition** — accepted asks become tracked work issues (by register ID) and the outcome returns to the customer via CS. See the PM skill's *From decision to delivery*. As FDE, when asked, confirm delivery details for your customer and relay the shipped outcome.

---

## The register (ask/answer ledger)

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

---

## Round 2 — responding to PM follow-ups

1. When the PM posts follow-up questions (threaded under your summary comment), answer **on the same thread** — `save_comment` with `parentId: <the PM comment's id>` so the round-trip stays one nested conversation. Lead with the header envelope and **end the reply with the full register snapshot** (all items, not just the changed rows), per *Keep the register readable*.
2. Answer what you can from existing evidence. Questions flagged `(customer-dependent)` require a customer touch — say so, give an ETA, don't guess on the customer's behalf.
3. **New evidence goes in the doc, not the thread**: append an appendix section to the existing evidence doc (`save_document` with `id:`) and link it from your reply. The thread carries conclusions; the doc carries proof.
4. Update the doc's `feedback_record` **only if facts changed** (e.g. `repro_status`, `severity`, new `asks`). Answering a question doesn't require a record edit; changing a fact does. Remove resolved entries from `open_questions`.
5. **Converge, don't count.** Aim to shrink the register's open set with this reply, not just answer questions — see the PM skill's *When is the loop closed?* section. Most items settle in one or two exchanges, but that's an observation, not a pass/fail bar; if a round doesn't trend the open items to zero, or adds more than it closes, that's the cue to recommend a sync.

---

## The `feedback-record` schema (v1)

Every evidence doc ends with exactly one fenced ```yaml block starting with `feedback_record: v1`. This is the machine-readable contract the companion `PM-feedback-triage-linear` skill parses — keep field names and enum values exact.

```yaml
feedback_record: v1
issue: AFP-###                # the Linear issue identifier
date: YYYY-MM-DD
customer: <name from intake, or internal-demo>
feature_family: <Testing Center | Agent Script | Agent Builder | Voice | Data Cloud | Other — free text allowed, prefer the intake's Product Feature Family>
gap_type: bug | limitation | feature-request | docs-gap          # exactly one
severity: blocker | workaround-exists | inconvenience            # exactly one
customer_impact: adoption-blocking | deal-risk | productivity-loss | confidence-loss   # one or more, comma-separated
repro_status: reproduced-live | reported-only | intermittent | not-reproducible        # exactly one
root_cause_known: true | false
root_cause: <one line; empty if unknown>
asks:
  - <primary ask, one line>
  - <alternative / interim, one line>
workaround: <one line, or "none">
evidence_doc: <this doc's Linear URL — fill after save_document returns it, via a second save_document update, or leave as pending>
open_questions:
  - <anything the FDE/SE could not answer — omit the list only if truly empty>
notes: <optional nuance, e.g. why an enum was a near-fit>
```

**Enum meanings** (pick honestly — PM rollups depend on these being comparable across FDEs):
- `feature_family` — Set to mirror the intake's **Product Feature Family** (or Feature Group when that's the precise area), so PM rollups key consistently. If your evidence refines which area it really is, say so in prose and note it in the record's `notes` — don't silently diverge from the intake taxonomy.
- `gap_type` — `bug` (behaves contrary to design), `limitation` (works as designed but the design falls short), `feature-request` (net-new capability), `docs-gap` (capability exists, discoverability/guidance doesn't).
- `severity` — `blocker` (customer cannot proceed), `workaround-exists` (proceeding, but painfully), `inconvenience` (annoyance, low friction).
- `customer_impact` — `adoption-blocking`, `deal-risk`, `productivity-loss`, `confidence-loss` (erodes trust in the product, e.g. false test passes).
- `repro_status` — `reproduced-live` (you reproduced it yourself), `reported-only` (customer report, not yet reproduced), `intermittent`, `not-reproducible`.

---

## Evidence document template

Use as the `content` for `save_document`. Delete sections that don't apply; rename "scenarios/cases" to whatever fits (findings, repro, examples).

```markdown
> **TL;DR.** <1–3 sentences: the gap, why it matters, and how it maps to the customer's stated need.>

**Contents**
1. Context
2. Summary / verdict
3. What happens (evidence)
4. Expected vs actual
5. Root cause (if known)
6. Why it matters
7. Recommendation / product ask
8. Questions a PM will ask
9. Appendix — reproduction & raw data
10. Machine-readable record

---

## 1. Context
- **Feature:** <feature area>
- **Environment:** <org / agent / app + version(s)>
- **Customer & use case:** <pull from the issue intake fields; tie to the outcome they want>

## 2. Summary / verdict
<The headline finding in a sentence or two. If there are multiple cases/scenarios, a compact table:>

| Case | Expected | Actual | Result |
|---|---|---|---|
| <name> | <…> | <…> | ✅ / ❌ |

## 3. What happens (evidence)
<Concrete, reproducible detail: transcripts (use blockquotes), steps, screenshots as links, query results. Keep it factual.>

## 4. Expected vs actual
| | Expected | Actual |
|---|---|---|
| <dimension> | <…> | <…> |

## 5. Root cause (if known)
<The mechanism. Include the single most decisive piece of proof (trace, log, data point). Put bulky raw evidence in a collapsible block:>

<details>
<summary>Raw evidence (trace / log / query output)</summary>

```
<paste raw data here>
```
</details>

## 6. Why it matters
<Business / adoption impact. Connect to the customer outcome and to scale/production risk.>

## 7. Recommendation / product ask
1. **<Primary ask>** — <what should change>.
2. **<Alternative / interim>** — <a lighter mitigation, e.g. a warning or doc>.
3. **Until then** — <workaround FDEs/SEs or customers can use today>.

| Item | Status | Next |
|---|---|---|
| **A1** <primary ask, ≤6 words> | ⏳ open · R1 | PM |
| **A2** <alternative / interim, ≤6 words> | ⏳ open · R1 | PM |

## 8. Questions a PM will ask
<Pre-answer the follow-ups that would otherwise require a meeting. Cover at least:>

| Question | Answer |
|---|---|
| Which customers / how many are affected? | <…> |
| Regression or always been this way? | <…> |
| How often does it bite (every time / edge case)? | <…> |
| Is there a workaround, and is the customer using it? | <…> |
| Can we quantify the impact (time lost, deals, scale)? | <…> |
| What changes for the customer if the ask ships? | <…> |
| <feature-specific question you'd expect> | <…> |

**Open questions (couldn't answer from evidence):**
- <flagged explicitly — the PM can reply on the thread instead of booking a call>

## 9. Appendix — reproduction & raw data
- **Environment / IDs:** <org, agent, versions, session/run IDs>
- **Repro steps / commands:** <exact steps or CLI>
- <details><summary>Raw data / queries / logs</summary>

  ```
  <sanitized raw material>
  ```
  </details>

*Any sensitive values shown are mock/sanitized.*

## 10. Machine-readable record

<the fenced yaml `feedback_record: v1` block, per the schema above>
```

---

## Summary comment template

Use as the `body` for `save_comment`. Keep it short — it's the thread-level hook; the doc is the depth.

```markdown
### 🟢 Round 1 · FDE → PM
**Purpose:** share evidence + asks · **Ball after:** 🔵 PM · **Open:** 3 (A1, A2, Q1)
*🤖 via `FDE-customer-feedback-linear` · <attribution>*

## 1 · What I found
<1–2 sentences framing, tying to the customer's stated ask (quote the intake if apt). Include the TL;DR and, if apt, "Refines the intake" line.>

**TL;DR:** <the gap in one line.>

**Refines the intake** *(omit if the intake framing holds)*: <one line — what the evidence shows the problem actually is, vs how the intake described it.>

## 2 · Repro & results
<optional compact results table — the same one from the doc's section 2>

## 3 · Root cause
<one line, if known — pointer to the doc's §5 for proof.>

## 4 · Asks & Questions

🤖 *Every item below is the **agent's proposal** — the FDE can **override** any of them; an item shows an override note only once that happens.* Each is laid out **Ask → Why**, with the reasoning bulleted.

> **A1 · Ask:** <primary ask, one line>
> → <one-line framing: weight / role>
>
> *why —*
> - **From evidence:** <chain from evidence to the ask>
> - **Why A1 and A2/A3:** <relation to the other asks>
> - **Grounded in:** <proof — direct evidence vs inference>
> - **Confidence:** <…>
> - **Would soften if:** <condition>

> **A2 · Ask:** <alternative / interim, one line>
> → <framing>
>
> *why —*
> - **From evidence:** <…>
> - **Layer:** <the near-term mitigation role>
> - **Grounded in:** <…>
> - **Confidence:** <…>

> **Q1 · Question** *(evidence limit)***:** <what your investigation didn't cover>
> → <one line: flag if it matters before roadmap>
>
> *why —*
> - **Grounded vs inferred:** <what's proven vs assumed about the boundary>
> - **Would convert to a task if:** <the trigger>

## 5 · 📒 Register — after Round 1 (FDE)
**ball: 🔵 PM** · open: 3 (A1, A2, Q1)

| Item | Status | Next |
|---|---|---|
| **A1** <primary ask, ≤6 words> | ⏳ open · R1 | PM |
| **A2** <alternative / interim, ≤6 words> | ⏳ open · R1 | PM |
| **Q1** <evidence-limit label, ≤6 words> | ⏳ open · R1 | PM |

*Reading it: **bold ID** = changed this round · **Next** = a name (whose move) / ✓ done (resolved) / → AFP-## (tracked in delivery) · IDs join the `feedback_record` in the doc.*

**Status legend**
- ⏳ **open** — awaiting a decision; the ball is with whoever's named in **Next**.
- 📋 **accepted** — committed as a near-term deliverable.
- 🔎 **to size** — accepted in principle and will be built, but not yet scheduled or estimated.
- 🔴 **blocked** — can't proceed without external / customer input.
- ✅ **resolved** — answered or closed; no further action.
- ❌ **declined** — not being pursued; reason in the item's *why*.

**Full write-up (inline document on this issue, no download needed)**
- 📄 [<doc title>](<doc url>) — <one line on what's inside; includes pre-answered PM questions + a machine-readable feedback record>

*<optional: "Prepared from a live investigation; data shown is mock.">*
```

---

## Worked example

A concrete, filled instance to imitate. **Scenario:** an FDE finds that a multi-turn Testing Center suite reports a `FAILURE` that does **not** reproduce in live Agent Preview. They gathered the evidence themselves — 4 multi-turn scenarios run both ways, the Testing Center run results, and a Data Cloud runtime trace — then asked their coding agent to package it onto issue `AFP-###`. Below is what the agent produced. (Data here is a demo agent with mock records — no real customer data.)

### → The document (created with `save_document`, `issue: AFP-###`)

Title: **"Multi-turn testing gap — evidence pack (Live Preview vs Testing Center)"**

> **TL;DR.** Agentforce Testing Center executes only the **final** turn of a multi-turn test live and rebuilds context from the `conversationHistory` **text**. Any value that existed solely as a prior **action output** is lost — so a correct agent looks broken. And some cases *pass* only because credentials happened to sit in the history text, which is false confidence.

**Contents:** 1. Context · 2. Verdict · 3. Evidence (the conversations) · 4. Why they differ · 5. Testing Center results · 6. Root-cause proof · 7. Asks · 8. PM questions · 9. Appendix · 10. Machine-readable record

**1. Context**
- Feature: Testing Center (multi-turn `sf agent test`)
- Environment: agent `PowerGreens1_xvl2to` v3 (active), org `my-org`
- Customer & use case: *<from the intake fields — the need to validate stateful multi-turn flows before scaling>*

**2. Verdict** — 3/4 scenarios agree between live and Testing Center; **1 diverges (S2, name recall)**; and 2 of the "agreements" hold only because credentials sat in the transcript.

| Scenario | Live preview | Testing Center | Match |
|---|---|---|---|
| S1 order status after verify | tracking returned | re-ran verify+status → tracking | ✅ agree |
| **S2 name recall** | "Jane Smith" | "I cannot provide the full name" (FAILURE) | ❌ **diverge** |
| S3 topic switch | status w/o re-verify | re-ran verify+status → tracking | ✅ agree (diff path) |
| S4 guardrail | refund declined | refund declined | ✅ agree |

**3. Evidence — the divergent case (S2)**
> *Live:* … "What is the full name on file?" → **"The full name on file for your account is Jane Smith."**
> *Testing Center (final turn live, prior turns as static history):* same question → **"I cannot provide the full name on file…"** · `actionsSequence: []`

**4. Why they differ** — live executes every turn and persists variable state; Testing Center executes only the final turn and rebuilds state from transcript text, so a prior action's output that isn't in the text is lost. S1/S3 only "pass" because the order # + email were in the history, letting it re-run `verify_customer`.

**5. Testing Center results (per case)**

| Case | Scenario | actionsSequence | output_validation |
|---|---|---|---|
| 1 | S1 | `['verify_customer','get_order_status']` | PASS |
| 2 | S2 | `[]` | **FAILURE** |
| 3 | S3 | `['verify_customer','get_order_status']` | PASS |
| 4 | S4 | `[]` | PASS |

**6. Root-cause proof (Data Cloud runtime trace, `ssot__AiAgentInteractionStep__dlm`)**

Live session — a `VARIABLE_UPDATE_STEP` fired on the verify turn:

```
// PRE
{ "order_number":"", "is_verified":"", "customer_name":"" }
// POST
{ "order_number":"A-2231", "is_verified":"true", "customer_name":"Jane Smith" }
```

→ **1** step recorded `customer_name` (live) vs **0** in the Testing Center session. Same agent; the only difference is whether the verify turn actually ran. This is a testing-method limitation, not an agent defect.

**7. Recommendation / product ask** *(full asks are in §4/§6 above and in the machine record below; register carries the labels)*

| Item | Status | Next |
|---|---|---|
| **A1** Execute prior turns to persist state | ⏳ open · R1 | PM |
| **A2** Warn when a value isn't in the transcript | ⏳ open · R1 | PM |
| **A3** Document the state-replay limitation | ⏳ open · R1 | PM |

**8. Questions a PM will ask**

| Question | Answer |
|---|---|
| Which customers are affected? | Any customer building stateful multi-turn agents who relies on Testing Center for pre-deployment validation; reported by one enterprise customer, reproduced on a demo agent. |
| Regression or always been this way? | By design — Testing Center's multi-turn model has always replayed history as text. Not a regression. |
| How often does it bite? | Every multi-turn case whose expected answer depends on a prior action's *output*; masked whenever the needed value happens to be in the transcript text. |
| Workaround? | Validate stateful cases with live `sf agent preview` scripts. Customer is doing this; it doesn't scale and isn't CI-friendly. |
| Quantified impact? | For the customer's use case, every stateful flow (verify-then-act) is untestable in CI — that's most of their production conversations. |
| What changes if the ask ships? | Testing Center passes become trustworthy for stateful flows → customer can gate deployment on the suite instead of manual preview sessions. |

**Open questions (couldn't answer from evidence):**

| Item | Status | Next |
|---|---|---|
| **Q1** Not yet re-run on voice channel | ⏳ open · R1 | PM |

**9. Appendix** — session IDs, the exact `sf agent preview`/`sf agent test` invocations, and the Data Cloud SQL, with raw JSON in `<details>` blocks.

**10. Machine-readable record**

```yaml
feedback_record: v1
issue: AFP-###
date: 2026-07-20
customer: internal-demo
feature_family: Testing Center
gap_type: limitation
severity: workaround-exists
customer_impact: confidence-loss, productivity-loss
repro_status: reproduced-live
root_cause_known: true
root_cause: Testing Center executes only the final turn; prior turns are static text, so action-output state is never created
asks:
  - Persist action-output state across turns by executing prior turns
  - Warn when a case depends on a value not present in the transcript
workaround: Validate stateful multi-turn cases with live sf agent preview scripts
evidence_doc: <doc url>
open_questions:
  - "Q1: reproduced on chat; channel-independent by mechanism, not yet re-run on the customer's voice channel"
notes: S1/S3 "passes" are false confidence — they re-derive state from credentials left in the history text. feature_family matches the intake's Product Feature Family as-is; no refinement needed.
```

### → The summary comment (posted with `save_comment`, `issueId: AFP-###`)

> ### 🟢 Round 1 · FDE → PM
> **Purpose:** empirical proof of the gap + 3 asks (A1–A3) · **Ball after:** 🔵 PM · **Open:** 4 (A1–A3, Q1)
> *🤖 via `FDE-customer-feedback-linear` · prepared from a live investigation*
>
> ## 1 · What I found
> Reproduced the multi-turn state gap on a live agent and captured the root cause from the Data Cloud runtime trace.
>
> **TL;DR:** Testing Center runs only the final turn live and rebuilds context from the history *text*; any value that existed only as a prior action output (here, the verified name) is lost, so a correct agent looks broken.
>
> **Refines the intake:** the intake reads as an agent defect ("agent forgets context mid-conversation"); the evidence shows it's a testing-tool limitation — Testing Center replays prior turns as text rather than executing them.
>
> ## 2 · Repro & results
> | Scenario | Live | Testing Center | Match |
> |---|---|---|---|
> | S2 name recall | "Jane Smith" | "cannot provide the full name" (FAILURE) | ❌ |
>
> ## 3 · Root cause
> Testing Center executes only the final turn; prior turns are replayed as static history text, so a prior action's output is never re-created as state. See the doc's §6 for the Data Cloud trace proof.
>
> ## 4 · Asks & Questions
>
> 🤖 *Every item below is the **agent's proposal** — the FDE can **override** any of them; an item shows an override note only once that happens.* Each is laid out **Ask → Why**, with the reasoning bulleted.
>
> > **A1 · Ask:** Persist action-output state across turns by *executing* prior turns (not replaying them as text).
> > → The complete fix; heavy (an execution-model change).
> >
> > *why —*
> > - **From evidence:** the trace shows prior-turn action outputs (the verified name) are dropped, replayed as history text only; the one fix that restores them is executing prior turns.
> > - **Why A1 and A2/A3:** A1 is complete but heavy; A2 (warn) + A3 (doc) ship without touching the engine — layered fallbacks, not duplicates.
> > - **Grounded in:** the Data Cloud trace (the 1-vs-0 `VARIABLE_UPDATE_STEP`) — direct evidence, not inference.
> > - **Confidence:** high on the mechanism, open on feasibility (a platform-team call).
> > - **Would soften if:** executing prior turns proves infeasible → A1 drops to "warn + document" (A2 + A3 only).
> >
> > **A2 · Ask:** Warn when a multi-turn case depends on a value not present in the transcript.
> > → Near-term; today it silently FAILs and reads as an agent bug.
> >
> > *why —*
> > - **From evidence:** the S2-class case fails silently and looks like an agent defect; a warning turns a silent wrong-answer into a visible one.
> > - **Layer:** the near-term mitigation — no engine change, so it lands while A1 is sized.
> > - **Grounded in:** the same trace; detectable at authoring time.
> > - **Confidence:** high — purely additive.
> >
> > **A3 · Ask:** Document that stateful cases only hold when every needed value is in the history text; otherwise validate with live preview.
> > → Immediate; ships alongside A2.
> >
> > *why —*
> > - **From evidence:** the "passes" (S1/S3) are misleading — they pass only because credentials sat in the transcript.
> > - **Layer:** zero-cost guidance, the doc half of the A2 warning.
> > - **Grounded in:** the false-confidence finding (§3).
> >
> > **Q1 · Question** *(evidence limit)***:** Reproduced on a chat agent; channel-independent by mechanism, not yet re-run on the customer's voice channel.
> > → Flag if voice verification matters before roadmap.
> >
> > *why —*
> > - **Grounded vs inferred:** the replay model is identical for chat and voice by construction — an **inference from the mechanism**, not a voice observation.
> > - **Would convert to a task if:** the PM wants voice confirmation before roadmap.
>
> 📄 Full write-up: [Multi-turn testing gap — evidence pack](<doc url>)
>
> ## 5 · 📒 Register — after Round 1 (FDE)
> **ball: 🔵 PM** · open: 4 (A1–A3, Q1)
>
> | Item | Status | Next |
> |---|---|---|
> | **A1** Execute prior turns to persist state | ⏳ open · R1 | PM |
> | **A2** Warn on transcript-absent dependency | ⏳ open · R1 | PM |
> | **A3** Document the replay limitation | ⏳ open · R1 | PM |
> | **Q1** Voice channel not yet re-run | ⏳ open · R1 | PM |
>
> *Reading it: **bold ID** = changed this round · **Next** = a name (whose move) / ✓ done (resolved) / → AFP-## (tracked in delivery) · IDs join the `feedback_record` in the doc.*
>
> **Status legend**
> - ⏳ **open** — awaiting a decision; the ball is with whoever's named in **Next**.
> - 📋 **accepted** — committed as a near-term deliverable.
> - 🔎 **to size** — accepted in principle and will be built, but not yet scheduled or estimated.
> - 🔴 **blocked** — can't proceed without external / customer input.
> - ✅ **resolved** — answered or closed; no further action.
> - ❌ **declined** — not being pursued; reason in the item's *why*.

### How an FDE/SE drives this with their coding agent

> "Attach this feedback to **AFP-###** as an in-app doc + summary comment. Feature: Testing Center. Here's the evidence: [paste the 4 scenarios run both ways, the Testing Center results, and the Data Cloud trace]. It's a multi-turn state-persistence gap — Testing Center only runs the final turn."

The agent then follows the Procedure above: verify workspace access → read the intake fields → draft the doc from the template → `save_document` → `save_comment` → report the links. The template is feature-agnostic — swap "scenarios" for whatever fits (findings, repro steps, examples) and the same shape applies to any Agentforce feature.
