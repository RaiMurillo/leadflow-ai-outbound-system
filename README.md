# LeadFlow AI — Outbound Prospecting System

A hands-on GTM engineering practice project: building an end-to-end outbound
lead-generation system in **Clay** — from a 47.9M-company universe down to a
verified, personalized, ready-to-send outreach list.

This was built as practice for a **GTM Engineer / Outbound Engineer** role,
simulating the exact workflow described in the job post: infrastructure,
list building, personalization, copywriting, and iterative campaign thinking.

> **Note:** LeadFlow AI is a fictional client used for practice. No real
> companies were contacted; all emails generated were for demonstration only.

---

## The Brief

**Fictional client:** LeadFlow AI — sells AI automation software to B2B SaaS
companies (automating CRM updates, lead routing, and follow-ups for sales
teams).

**The ask:** Find SaaS companies that could benefit from AI automation,
identify the right decision-makers, research them, personalize outreach, and
produce a qualified outbound list.

---

## The Pipeline

```
TARGET MARKET (ICP defined)
      ↓
Find Companies (Clay, filtered search)
      ↓
Filter / Qualify Companies (industry, size, revenue, B2B)
      ↓
AI Qualification (custom prompt — GOOD FIT / NOT A FIT)
      ↓
ICP Score (formula — weighted scoring, 0–100)
      ↓
Find Decision-Makers (Surfe — job title + seniority targeting)
      ↓
Verify / Find Emails (waterfall enrichment)
      ↓
Research Company (AI)
      ↓
Extract Personalization Signal (AI)
      ↓
Write Personalized Opening Line (AI)
      ↓
Generate Full Cold Email (AI)
      ↓
Export → Outbound Tool (Brevo)
```

---

## Step 1 — ICP Definition

Before touching any tool, the target profile was defined as a hypothesis to
test, not an assumption:

| Criteria | Target |
|---|---|
| Industry | B2B SaaS |
| Location | United States |
| Company size | 50–500 employees |
| Revenue | $5M–$100M |
| Business type | B2B |

**Primary personas:** VP Sales, Head of Revenue Operations, Head of Sales
Operations, COO, CRO, CEO/Founder

**Buying signals:** Hiring sales/RevOps roles, rapid headcount growth, active
sales team, multiple CRM/sales tools in use

**Exclusions:** Sub-10-employee companies, B2C businesses, consulting
agencies, dev communities/forums, AI infrastructure/hosting providers

**Hypothesis tested:**
> Growing B2B SaaS companies with 50–500 employees and expanding sales teams
> are likely to have repetitive CRM and operational processes that can be
> automated.

---

## Step 2 — Company Search & Filtering

Built in Clay's **Find Companies** source using layered filters:

- Industry: Software Development
- Description contains: "SaaS" (+ related keywords)
- Country: United States
- Company size: 51–200 + 201–500 employees
- Business type: B2B

**Funnel so far:**

| Stage | Companies |
|---|---|
| Unfiltered universe | 47,911,094 |
| After industry + description + size + country | 1,120 |
| After adding Business Type = B2B | 943 |

**Key lesson:** Firmographic filters alone are never fully clean. Manual
preview caught false positives even after filtering — companies like Stack
Overflow (too large), Together AI (infrastructure, not SaaS product), and
TekRevol (agency, not product company) still passed the filters. This is
exactly why an AI qualification layer was added next, rather than trying to
perfect the filter logic indefinitely.

Result limited to **30 companies** for the practice batch, imported to a
new table.

---

## Step 3 — AI Qualification Layer

A custom "Use AI" column was built to catch what static filters couldn't:
whether a company was a genuine **product** company (vs. agency/infra) and
whether it plausibly had an active sales function that could benefit from
automation.

**Prompt used:**

```
You are a B2B sales analyst working for LeadFlow AI, a company that sells
AI automation software to B2B SaaS companies (automating CRM updates, lead
routing, and follow-ups for sales teams).

I am evaluating {{Name}} (website: {{Domain}}) as a POTENTIAL CUSTOMER, not
as a competitor or reference point.

Answer the following:
1. Is {{Name}} itself a B2B SaaS product company (not a consulting agency,
   dev community, forum, or AI infrastructure/hosting provider)?
2. Does {{Name}} likely have an active sales team that manages leads, CRM
   data, and follow-ups?
3. Based on this, would {{Name}} plausibly benefit from buying AI
   automation software for their sales operations?
4. Final verdict: GOOD FIT or NOT A FIT — with one sentence explaining why.

Do NOT describe who {{Name}} sells to or their own customers' ICP. Only
assess whether {{Name}} is a good prospect FOR US to sell to.
```

**Output fields:** `Is SaaS Product Company` (bool), `Has Active Sales Team`
(bool), `Fit Verdict` (text), `Fit Reason` (text)

**Example result (TekRevol — correctly rejected):**
> *"TekRevol is a software development/services agency rather than a B2B
> SaaS product company and therefore falls outside LeadFlow AI's stated
> target market."*

**Example result (Abacus.AI — correctly accepted):**
> *"Abacus.AI's official careers page identifies an Associate Account
> Executive role responsible for outbound prospecting and pipeline
> generation... its sales operation likely manages prospects, pipeline,
> CRM activity, and follow-ups."*

**Lesson learned:** The first version of this prompt (Clay's default
"Research company ICP" template) asked the wrong question — it researched
*who the target company sells to*, not *whether the target company is a
good prospect for us*. Rewriting the prompt to explicitly frame the
evaluation direction fixed this. Also noted: even with an explicit
instruction to return only two verdict values, the model occasionally
drifted to a third phrase ("Good prospect") — a reminder that strict output
constraints need to be enforced harder in production (e.g., "respond with
EXACTLY one of these two words").

---

## Step 4 — ICP Fit Score (Formula)

A weighted scoring formula combined the AI qualification with hard
firmographic data — deliberately layering signals rather than trusting the
AI verdict alone (an AI can call a company "GOOD FIT" on business-model
grounds while it still fails on size/revenue, as happened with Stack
Overflow).

| Condition | Points |
|---|---|
| Fit Verdict = GOOD FIT / Good prospect | 30 |
| Is SaaS Product Company = true | 25 |
| Has Active Sales Team = true | 20 |
| Size = 51–200 or 201–500 | 25 |
| **Max** | **100** |

**Debugging note:** Initial formula compared `Size` against `"51-200
employees"`, but the actual field value was just `"51-200"` (no word
"employees"). Text-equality conditions fail silently on exact-string
mismatches — always verify the real field format before writing comparison
logic, not just what you assume it looks like.

**Result:** Filtered to companies scoring **100** → **23 companies**
advanced to the next stage.

---

## Step 5 — Find Decision-Makers

Used Clay's **Find People at Company** (Surfe) enrichment against the 23
qualified companies:

- **Job titles:** VP Sales, Head of Revenue Operations, Head of Sales
  Operations, COO, CRO, CEO, Founder
- **Seniority:** C-Level, Director, VP
- **Country:** United States
- **Limit:** top match per company

**Result:** Found named contacts (name, title, LinkedIn URL) for roughly
15–18 of the 23 companies. The remainder returned "No people found" — a
realistic data-coverage gap, not a process failure.

---

## Step 6 — Email Verification (Waterfall)

Ran a work-email-finder enrichment against found contacts.

**Result: 10 of 23 companies (~43%) yielded a verified, correctly formatted
work email** on a single provider.

**Lesson for iteration:** A production system would chain multiple
providers (e.g., Surfe → Apollo → Findymail → Prospeo) so each catches what
the previous missed, raising the hit rate. This single-provider run was
sufficient to prove the concept for a practice build.

---

## Step 7 — Research → Personalization → Cold Email

Three chained AI columns, each filtered to only the **10 email-verified
contacts** (to control enrichment cost):

1. **Company Research** — 3–4 sentence AI summary of what the company
   sells, business model, and likely operational pain points
2. **Personalization Signal** — one specific, evidence-based fact extracted
   from the research (explicitly instructed to avoid generic statements)
3. **Personalized Opening** — that signal converted into one natural, ≤20
   word sentence
4. **Cold Email** — full subject + body combining the opening line, a named
   pain point, the offer, and a low-pressure CTA (≤70 words)

**Example output (Sidetrade):**

> **Personalization signal:** *"Sidetrade's Augmented Cash deployment
> across one in three staff at 600 Loxam locations worldwide raises
> questions about scaling automation."*

**Example output (Reltio) — flagged "high confidence" by the model itself:**

> **Personalization signal:** *"After a merger left Clario with nine ERPs,
> Reltio unified its data and put Clario on a $4.5M YOY savings
> trajectory."*

**Example full email (Stack Overflow):**

> **Subject:** Streamlining sales workflows
>
> I noticed Stack Overflow for Teams supports more than 20,000
> organizations, helping them distribute knowledge and increase
> efficiency. With multiple products and enterprise sales, keeping CRM
> ownership, lead routing, and follow-ups consistent across self-serve and
> enterprise opportunities can be a challenge. We help B2B SaaS teams
> automate those repetitive workflows with AI. Open to a 15-minute
> conversation next week?

Every email tested pulled a real, specific, verifiable detail rather than
generic filler ("I saw you're growing") — the difference between
personalization that actually reads as researched vs. templated.

---

## Full Funnel Summary

```
47,911,094  Total companies in Clay's database
     ↓  (industry + size + location + description filters)
     943  After firmographic filtering
     ↓  (imported top 30 for practice, ran AI qualification)
      23  Scored 100/100 on ICP Fit Score
     ↓  (found named decision-makers)
   ~16  Companies with a contact identified
     ↓  (verified work email)
      10  Ready-to-send, fully qualified prospects
```

**~0.02% of the raw universe** converted into a fully qualified,
contact-verified, personalized prospect — the entire value of a layered
GTM engineering pipeline in one number.

---

## Tools Used

- **Clay** — company search, enrichment, AI columns, formulas, waterfalls
- **Surfe** (via Clay) — people search / decision-maker finding
- **Email-finder waterfall** (via Clay) — work email verification
- **Brevo** (planned) — outbound sending platform for the final export

---

## Next Iterations (if continued)

- [ ] Add a multi-provider email waterfall to raise the 43% hit rate
- [ ] Tighten the Fit Verdict prompt to force exactly two output values
- [ ] Convert ICP Fit Score to a true numeric field (was built as Text,
      limiting filter operators to string comparisons only)
- [ ] Build a second "review tier" (score 60–79) instead of only using the
      100-score cutoff
- [ ] Set up the Clay → Brevo handoff via webhook instead of manual export,
      for a fully automated pipeline
- [ ] Run an A/B test between two opening-line styles once live sending is
      connected, and iterate based on reply/positive-reply rate

---

## What This Demonstrates

This project covers the core competencies described in outbound/GTM
engineering roles: ICP definition and hypothesis-driven targeting,
multi-source list building and qualification, AI-assisted research and
personalization at scale, and a documented, debuggable system — including
the mistakes made and fixed along the way, which matter as much as the
final result.
