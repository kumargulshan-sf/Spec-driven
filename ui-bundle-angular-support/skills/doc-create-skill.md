# Doc Creation Skill — Technical Decision Documents

**Purpose:** Guidelines for creating internal technical decision documents that get read and acted on. Apply these principles every time you write a stakeholder-facing doc.

---

## Core Principles

### 1. BLUF (Bottom Line Up Front)
State the recommendation in the FIRST paragraph. A reader who stops after 3 sentences should know:
- What you're recommending
- Why (one-line reason)
- Key trade-off you're accepting

### 2. Inverted Pyramid
Most important info first, supporting details after, background last. Never "build up to a conclusion." The conclusion IS the opening.

### 3. Optimal Length
- Target: 600-900 words body text (2-3 pages)
- Readers only read 18% of added verbiage
- Every section must earn its place by answering a question a skeptical reader would ask
- If a section doesn't change the reader's decision, cut it

### 4. Section Count: 5-8
- Fewer is better
- Each section should fit on one screen without scrolling
- Merge related sections rather than creating many short ones

### 5. Recommendation Before Evidence
Don't make the reader hold a comparison table in memory to figure out what you're recommending. State the decision, THEN show why.

### 6. "Rejected Alternatives" Builds Trust
Explicitly showing what you evaluated and rejected is the highest-trust signal in a proposal. It preempts devil's advocate objections. Keep it brief (1-2 sentences per rejected option).

### 7. Active Voice, First Person Plural
- "We will ship X" — signals a real decision
- "It was determined that X" — signals hedging, makes readers re-derive the conclusion
- Own the recommendation

---

## Structure Template

```
Title: [Verb phrase + subject]
  e.g. "Ship Angular CLI as Paved Path for UI Bundles"
  NOT: "Angular Framework Comparison"

---

## TL;DR (3-5 sentences)
What we're shipping. Why. Key trade-off. What we need from the reader (approval/feedback/awareness).

## Section 1: Context (What exists, what we need)
Neutral facts. Forces at play. Constraints. No opinions yet.
Keep to 1-2 paragraphs or a table.

## Section 2: The Recommendation (What we're doing)
Active voice: "We will..."
How it works (brief). What it achieves.
This is the main section — the "what."

## Section 3: Why This Choice (Evidence)
Data, market signals, architectural reasoning.
Tables > prose for comparisons.
Only include data that changes the decision.

## Section 4: Consequences / Trade-offs
Split into:
- What we gain (bullets)
- What we accept (bullets — don't hide negatives)
- Why acceptances are OK (brief mitigation per item)

## Section 5: User Journey / How It Works
The end-user experience. Keep minimal — 4-6 steps max.
Shows you've thought through execution, not just theory.

## Section 6: Rejected Alternatives
1-2 sentences per option: what it was, why not.
This is where trust is built. Be honest about the rejected option's strengths.

## [Optional] Deep-Dive / Appendix
Link to separate doc for implementation details, POC results, raw data.
Don't inline 5 pages of technical detail into a 2-page decision doc.
```

---

## Anti-Patterns to Avoid

| Don't | Why | Do Instead |
|---|---|---|
| Build up to conclusion at the end | Reader gives up before reaching it | Conclusion first (BLUF) |
| Wall of prose paragraphs | F-pattern scanning skips 80% of it | Tables, bullets, bold lead sentences |
| Show ALL the data you collected | Overwhelms, signals insecurity | Only data that changes the decision |
| Hedge the recommendation ("we could...") | Reader doesn't know what to approve | "We will ship X" |
| Hide trade-offs | Reader finds them anyway, loses trust | Explicit negative consequences section |
| Give LOC counts / time estimates in decision doc | They change, date the doc, distract | Keep effort qualitative ("acceptable", "significant") |
| Pros/cons without a verdict | Decision fatigue — reader must decide | State your pick, then show why |
| Include implementation code | Wrong audience for a decision doc | Link to implementation doc |

---

## Formatting Rules

- **Bold the first sentence of each section** — F-pattern scanning catches it
- **Tables for structured comparisons** — offload cognitive work to visual scanning
- **Bullets for lists of 3+** — easier to scan than inline commas
- **One key insight per section** — if you can't summarize the section in one sentence, split it
- **Headings contain the conclusion** — "Why Angular 17+" not "Version Analysis"
- **No jargon without context** — if a term needs explanation, explain it inline or cut it

---

## Reader Psychology

| Reader Type | What They Need | Where They Stop |
|---|---|---|
| **Busy lead (2 min)** | TL;DR + recommendation | After section 2 |
| **Skeptical architect (10 min)** | Evidence + trade-offs + rejected alternatives | Reads full doc |
| **Future onboarder (later)** | Full context + deep-dive links | Reads everything including appendix |

Design for all three. The TL;DR serves the first. The body serves the second. Links serve the third.

---

## Checklist Before Sharing

- [ ] Can a reader stop after paragraph 1 and know the recommendation?
- [ ] Is the recommendation in active voice ("We will...")?
- [ ] Are trade-offs explicit, not hidden?
- [ ] Is there a "rejected alternatives" section?
- [ ] Is every section < 1 screen scroll?
- [ ] Total body text < 900 words (excluding tables)?
- [ ] No hedging language ("perhaps", "it seems", "we might consider")?
- [ ] Headings contain conclusions, not neutral topics?
- [ ] Data included only if it changes the decision?
- [ ] Implementation details linked, not inlined?

---

## Sources

- NNGroup: Inverted Pyramid, F-pattern, Progressive Disclosure, Chunking
- Minto Pyramid Principle (SCQA: Situation → Complication → Question → Answer)
- Michael Nygard: Architecture Decision Records (2011)
- AWS Prescriptive Guidance: ADR process + examples
- Rust RFC Template: 9-section structure
- Google SWE Book: Design doc requirements
- PEP 1: Python Enhancement Proposals (Rejected Ideas section)
- RFC 7282: Rough Consensus ("can anyone not live with this?")
- Ben Kuhn: Concrete examples, strong framing, clear titles
