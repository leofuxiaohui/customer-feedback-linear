---
name: FDE-customer-feedback-linear
description: Use when an FDE or Solution Engineer wants to share detailed Agentforce customer feedback via Linear — attach it to the customer-feedback intake issue (AFP-*) as one self-contained in-app Linear Document plus a concise summary comment on the thread, so PMs review everything without downloading anything. Turns evidence the FDE/SE has already gathered into a triage-ready write-up. Works for any feature (Testing Center, Agent Script, Voice, Builder, etc.).
---

# FDE Customer Feedback → Linear

A standalone skill dedicated to sharing **Agentforce customer feedback** via Linear. Package detailed feedback into a target customer-feedback issue as **(1)** one self-contained **in-app Document** (the deep dive) and **(2)** a concise **summary comment** on the thread that links the doc. Everything stays inside Linear — no attachments to download.

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

1. **Verify + read context.** `get_issue <id>`. Read the intake fields (**Customer Name, Product Feature Family/Group, Feedback Theme, Business Outcome/Use Case**) and tailor the framing so the evidence visibly answers the customer's stated need — quote it where useful.
2. **Draft the doc** from the *Evidence document template* below. Fill only the sections that apply. Keep raw data/logs/queries inside collapsible `<details>` blocks so the main narrative stays clean and scannable. Lead with the strongest proof.
3. **Create the doc linked to the issue:** `save_document` with `issue: <id>`, a clear `title`, and the markdown `content`.
   - **Do not pass `icon`** unless you know it is a valid Linear icon name — an invalid icon name errors the whole call. Omit it.
4. **Post the summary comment:** `save_comment` with `issueId: <id>` using the *Summary comment template* below — a TL;DR, a compact results table if applicable, the ask, and a link to the doc.
5. **Binary evidence only** (screenshots, PDFs that must be seen as images): use the attachment flow — `prepare_attachment_upload` (needs the exact byte size) → PUT the raw bytes to the returned signed URL with `curl --data-binary @file`, sending the returned headers **verbatim**, within 60s, **one file at a time** → `create_attachment_from_upload`. Prefer inline docs; attach a binary only when the picture *is* the evidence. (Linear does **not** render HTML attachments inline — never ship an HTML file as the deliverable; put the narrative in the doc.)
6. **Report** the doc URL and confirm the comment posted.

## Conventions & guardrails

- **One consolidated doc per feedback item** — not several small ones. If you split, cross-link them; better yet, fold appendices into the main doc as `<details>` sections.
- **Frame as a design gap / feature request with concrete asks**, not just "it's broken." Give the primary ask, an alternative/interim mitigation, and a documentation/workaround note.
- **Sanitize before posting.** Use mock/sample data; scrub org IDs, customer PII, tokens, and internal endpoints. State explicitly when data shown is mock.
- **Retiring a doc:** the Linear MCP has **no delete/archive-document tool**. To retire a doc, rename it to a `↪ Merged — archive me` stub whose body links the surviving doc, and ask the user to archive it in-app (hover the doc → ⋯ → **Archive**).
- **Markdown, not HTML.** Linear renders markdown tables, code fences, blockquotes, and `<details>` — use those. It will not render an HTML file inline.

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
8. Appendix — reproduction & raw data

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

## 8. Appendix — reproduction & raw data
- **Environment / IDs:** <org, agent, versions, session/run IDs>
- **Repro steps / commands:** <exact steps or CLI>
- <details><summary>Raw data / queries / logs</summary>

  ```
  <sanitized raw material>
  ```
  </details>

*Any sensitive values shown are mock/sanitized.*
```

---

## Summary comment template

Use as the `body` for `save_comment`. Keep it short — it's the thread-level hook; the doc is the depth.

```markdown
## <headline finding>

<1–2 sentences framing, tying to the customer's stated ask (quote the intake if apt).>

**TL;DR:** <the gap in one line.>

<optional compact results table — the same one from the doc's section 2>

**Product ask**
1. <primary ask>
2. <alternative / interim>

**Full write-up (inline document on this issue, no download needed)**
- 📄 [<doc title>](<doc url>) — <one line on what's inside>

*<optional: "Prepared from a live investigation; data shown is mock.">*
```

---

## Worked example

A concrete, filled instance to imitate. **Scenario:** an FDE finds that a multi-turn Testing Center suite reports a `FAILURE` that does **not** reproduce in live Agent Preview. They gathered the evidence themselves — 4 multi-turn scenarios run both ways, the Testing Center run results, and a Data Cloud runtime trace — then asked their coding agent to package it onto issue `AFP-###`. Below is what the agent produced. (Data here is a demo agent with mock records — no real customer data.)

### → The document (created with `save_document`, `issue: AFP-###`)

Title: **"Multi-turn testing gap — evidence pack (Live Preview vs Testing Center)"**

> **TL;DR.** Agentforce Testing Center executes only the **final** turn of a multi-turn test live and rebuilds context from the `conversationHistory` **text**. Any value that existed solely as a prior **action output** is lost — so a correct agent looks broken. And some cases *pass* only because credentials happened to sit in the history text, which is false confidence.

**Contents:** 1. Context · 2. Verdict · 3. Evidence (the conversations) · 4. Why they differ · 5. Testing Center results · 6. Root-cause proof · 7. Asks · 8. Appendix

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

**7. Recommendation / product ask**
1. Persist action-output state across turns by *executing* prior turns (not replaying them as text), **or**
2. Warn when a multi-turn case depends on a value not in the transcript — today it silently reports FAILURE, **or**
3. Document the limitation so `expectedActions`/`expectedOutcome` are only trusted when every needed value is in the history text.

**8. Appendix** — session IDs, the exact `sf agent preview`/`sf agent test` invocations, and the Data Cloud SQL, with raw JSON in `<details>` blocks.

### → The summary comment (posted with `save_comment`, `issueId: AFP-###`)

> ## Testing Center reports FAILURE on a case that works live
>
> Reproduced the multi-turn state gap on a live agent and captured the root cause from the Data Cloud runtime trace.
>
> **TL;DR:** Testing Center runs only the final turn live and rebuilds context from the history *text*; any value that existed only as a prior action output (here, the verified name) is lost, so a correct agent looks broken.
>
> | Scenario | Live | Testing Center | Match |
> |---|---|---|---|
> | S2 name recall | "Jane Smith" | "cannot provide the full name" (FAILURE) | ❌ |
>
> **Ask:** persist action-output state across turns, or warn when a case depends on a value not in the transcript.
>
> **Full write-up (inline doc, no download):** 📄 [Multi-turn testing gap — evidence pack](<doc url>)

### How an FDE/SE drives this with their coding agent

> "Attach this feedback to **AFP-###** as an in-app doc + summary comment. Feature: Testing Center. Here's the evidence: [paste the 4 scenarios run both ways, the Testing Center results, and the Data Cloud trace]. It's a multi-turn state-persistence gap — Testing Center only runs the final turn."

The agent then follows the Procedure above: verify workspace access → read the intake fields → draft the doc from the template → `save_document` → `save_comment` → report the links. The template is feature-agnostic — swap "scenarios" for whatever fits (findings, repro steps, examples) and the same shape applies to any Agentforce feature.
