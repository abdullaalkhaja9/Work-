# Counsel — an in-house legal AI for the bank's legal department

*Working name. Concept and architecture brief, September 2026.*

Counsel is a private legal AI platform, built for one legal department rather than sold to many.
It does the same jobs Harvey and Spellbook do (review and redline, drafting, playbooks, research)
but runs inside the bank's perimeter, encodes the department's own positions, and knows the bank's
regulatory context.

The pitch in one line: **every recurring piece of legal work in the department becomes a "skill"
that the platform can run, with the department's own playbook as the source of truth.**

---

## 1. Why build instead of buy

Harvey, Spellbook, Luminance and ContractPodAi are good products. A bank still has reasons to build:

| Concern | Off-the-shelf | Built in-house |
|---|---|---|
| Data residency and banking secrecy | Vendor cloud, usually US/EU | Runs in the bank's cloud region, or via an in-region model endpoint |
| Playbooks | Generic, editable to a point | The department's actual positions, versioned, with approval workflow |
| Regulatory content | Generic | Central bank rulebook, outsourcing rules, data protection law, internal policies loaded as first-class knowledge |
| Bilingual work | Usually English-first | Arabic/English side by side where the bank needs it |
| Integration | Word add-in, some DMS | Word add-in, Outlook, SharePoint/iManage, the bank's ticketing and procurement systems |
| Cost | Per-seat licence, often USD 1k+ per user per year | Pay per token, roughly USD 1–3 per full contract review (section 10) |
| Model risk management | Vendor black box | Every prompt, playbook and output is auditable and testable by the bank |

The honest counter-argument: the language model is the easy part. The hard parts are
(a) faithful tracked changes in Word, (b) playbooks that lawyers actually maintain, and
(c) an evaluation set that proves the tool is right often enough to trust. The plan below
spends most of its effort there.

---

## 2. Who uses it

- **In-house lawyers**: the primary users. Review, redline, draft, research, sign off.
- **Paralegals and legal ops**: intake, matter tracking, obligations register, template upkeep.
- **Business units** (procurement, treasury, retail, digital): self-serve intake and a triage bot that answers
  the simple questions and routes the rest.
- **Compliance and risk**: consumers of the obligations register and regulatory watch.
- **The General Counsel**: owns the playbooks and the approval of any skill before it goes live.

---

## 3. The core model: Matters, Skills, Playbooks, Knowledge

```
 Matter  ──has many──▶  Documents  ──run through──▶  Skills  ──governed by──▶  Playbooks
   │                                                    │
   └── audit trail of every AI action                   └── grounded in ──▶ Knowledge base
                                                                            (policies, rulebooks,
                                                                             precedents, past advice)
```

- **Matter**: the unit of work. A vendor contract, a facility, a dispute, a policy review. Holds documents,
  parties, deadlines, and the full audit log.
- **Skill**: a packaged capability. Instructions, the playbook(s) it uses, the tools it may call, the output
  schema, and a test set. Lawyers can read a skill in plain language; engineers can run it. See `skills/`.
- **Playbook**: the department's positions for one contract type. Preferred, fallback and walk-away positions
  per clause, with rationale and escalation rules. Written in a structured, reviewable format. See `playbooks/`.
- **Knowledge base**: internal policies, regulator rulebooks and circulars, approved clause library, precedent
  agreements, past legal memos. Retrieved with citations, never quoted from memory.

---

## 4. Feature set

### 4.1 Review and redline (the flagship)

Input: a counterparty's draft (Word or PDF). Output: a Word file with tracked changes, plus an issues list.

Pipeline:

1. Parse the document into a clause tree (headings, numbering, definitions, schedules), preserving
   original formatting so edits round-trip cleanly.
2. Classify the contract type and pick the playbook (or ask the lawyer to confirm).
3. Map each clause to the playbook's clause taxonomy (limitation of liability, indemnity, data
   protection, termination, governing law, audit rights, subcontracting, and so on).
4. For each clause, compare the counterparty's position with the playbook. Produce a finding:
   compliant / deviation / missing / unusual, with severity and the playbook's reasoning.
5. Propose redlines: the playbook's preferred language where a clause deviates, the fallback where the
   preferred is unrealistic given the deal context, and a flag for anything walk-away.
6. Write the redlines back as real Word tracked changes (insertions and deletions attributed to
   "Counsel AI"), plus margin comments explaining each change.
7. Produce an issues list and a short cover note to the business, in the department's house style.

The lawyer accepts, rejects or edits each change in Word. Nothing goes out without a human.

### 4.2 Drafting

- **From template**: pick a template, fill a short form (parties, term, amount, governing law),
  get a first draft with the correct optional clauses switched on or off by the playbook.
- **From a term sheet or email**: paste the business's ask, get a draft plus a list of open points
  the drafter had to assume.
- **Clause-level**: inside Word, "give me a mutual indemnity capped at fees, bank-favourable" pulls
  from the approved clause library first and drafts only when nothing fits.
- **Ancillaries**: board resolutions, powers of attorney, engagement letters, NDAs, side letters,
  waiver letters, standard notices.

### 4.3 Playbooks

- One playbook per contract type. Start with the highest-volume types: NDA, vendor/outsourcing
  agreement, SaaS/licence, facility agreement, guarantee, employment.
- Each clause entry has: preferred position, fallback positions in order, walk-away line,
  rationale, who can approve an exception, and the approved clause text.
- Playbooks live in version control with a review step. A change to a playbook re-runs that
  playbook's test set before it goes live (section 9).
- The playbook editor is a form, not YAML, for lawyers. YAML is the storage format.

### 4.4 Clause library and precedents

- Searchable library of approved clauses, tagged by contract type, position (bank as customer,
  bank as provider, bank as lender), and jurisdiction.
- Precedent agreements: the last signed version of each major deal type, with the negotiated
  deviations recorded so the next negotiation starts informed.

### 4.5 Ask Legal (research and Q&A)

- Questions answered from the knowledge base with citations to the exact paragraph.
- Sources: internal policies, the central bank rulebook and circulars, the data protection law,
  the companies and commercial laws the department relies on, and the department's own past memos.
- The answer says what it could not find. It never fabricates a regulation.

### 4.6 Intake and triage

- Business units submit requests through a form or by emailing a shared mailbox.
- The platform classifies the request, checks whether an existing template or FAQ answers it,
  drafts a first response for a lawyer to approve, and opens a matter with the right SLA.
- Dashboard for the GC: queue, turnaround times, which business units generate what.

### 4.7 Obligations and key dates

- After signing, extract renewal dates, notice periods, price review clauses, reporting obligations,
  audit rights, exit and transition obligations, and regulatory undertakings.
- Push them to a register with owners and reminders. This is what regulators ask for when they
  audit outsourcing arrangements.

### 4.8 Regulatory watch

- Ingest new circulars and consultation papers from the regulator.
- Summarise, identify which policies, templates and live contracts are affected, and open matters
  for the changes needed.

### 4.9 Summaries, translation and explanation

- Plain-language summary of any contract for the business.
- Arabic/English translation with legal terminology consistency, and side-by-side review where the
  Arabic text prevails.

---

## 5. The skills catalogue

Skills are what "all of our skills there" turns into. Each one is a folder with instructions,
playbook references, an output schema and tests. Initial catalogue, grouped by phase:

| Phase | Skill | What it produces |
|---|---|---|
| 1 | `contract-review` | Findings list, severity, proposed redlines as tracked changes |
| 1 | `nda-review` | Same, specialised for NDAs with a fast path for standard forms |
| 1 | `contract-summary` | One-page summary for the business |
| 1 | `issues-list` | Table of open points for negotiation |
| 2 | `draft-from-template` | First draft from a template and a short form |
| 2 | `draft-from-terms` | First draft from a term sheet or email |
| 2 | `clause-draft` | One clause, from the library first |
| 2 | `ask-legal` | Cited answer from the knowledge base |
| 2 | `compare-versions` | What changed between two drafts, in plain language |
| 2 | `negotiation-response` | Reply to counterparty's markup, clause by clause |
| 3 | `intake-triage` | Classification, first response, matter creation |
| 3 | `obligations-extract` | Key dates and obligations for the register |
| 3 | `regulatory-impact` | Circular summary and affected documents |
| 3 | `outsourcing-checklist` | Regulator's outsourcing requirements checked against a contract |
| 3 | `data-protection-check` | DPA and data clauses against the data protection law and bank policy |
| 3 | `sanctions-aml-clauses` | Presence and adequacy of sanctions, AML and KYC clauses |
| 3 | `board-resolution` | Resolution and minutes from a template |
| 3 | `translate-legal` | Arabic/English translation with a term glossary |
| 3 | `litigation-chronology` | Chronology and document index from a dispute bundle |
| 3 | `memo-draft` | Advice memo in house style from research notes |

See `skills/contract-review/SKILL.md` for a worked example of the format.

---

## 6. Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│ Clients                                                              │
│  Web app (matters, playbooks, dashboards)   Word add-in   Outlook    │
└───────────────┬─────────────────────────────────┬────────────────────┘
                │ SSO (Entra ID / SAML), RBAC, matter-level access
┌───────────────▼─────────────────────────────────▼────────────────────┐
│ API + orchestration (Python, FastAPI)                                │
│  Skill runner · Playbook service · Matter service · Audit log        │
├──────────────────────────────────────────────────────────────────────┤
│ Document engine                                                      │
│  DOCX/PDF parsing → clause tree · OOXML tracked-change writer        │
├──────────────────────────────────────────────────────────────────────┤
│ Retrieval                                                            │
│  Postgres + pgvector · hybrid (BM25 + embeddings) · citation index   │
├──────────────────────────────────────────────────────────────────────┤
│ Model gateway                                                        │
│  Claude via in-region endpoint · prompt cache · structured outputs   │
│  · citations · per-skill model and effort · cost metering            │
└──────────────────────────────────────────────────────────────────────┘
   Storage: Postgres (metadata), object store (documents, encrypted, KMS),
   immutable audit log. Integrations: SharePoint/iManage, ticketing, HR, procurement.
```

### Model layer

- **Model**: Claude Opus 5 for review, drafting and research. Claude Sonnet 5 for bulk extraction and
  classification. Claude Haiku 4.5 for routing and intake classification where cost matters more than nuance.
- **Hosting** is the first decision to make with the bank's information security team:
  - Anthropic's API directly, with a data processing agreement and, if the bank qualifies, zero data retention.
  - Or Claude on Amazon Bedrock, Google Vertex AI or Microsoft Foundry in the bank's own cloud region.
    PDF input, structured outputs, citations and prompt caching are all available there. Message
    batches and the Files API are not on Bedrock or Vertex, so bulk jobs run as normal requests.
- **Features that matter for legal work**:
  - Structured outputs so findings, redlines and obligations come back as validated JSON, not prose to parse.
  - Citations on every document block so each finding points at the clause it came from.
  - Prompt caching for the playbook and knowledge context, which is the same across hundreds of reviews.
  - Adaptive thinking with effort set per skill (high for review and drafting, low for routing).
  - 1M-token context, so a full facility agreement with schedules fits in one request.

### Document engine

This is where most competitors are weakest and where the most engineering time goes.

- Parse DOCX with a library that preserves the OOXML (python-docx for structure; direct XML for
  tracked changes). PDFs go through OCR when scanned.
- Keep a stable ID per paragraph so a finding can be written back to the exact run.
- Write tracked changes as real `w:ins` / `w:del` elements with author and timestamp, so lawyers use the
  Word review pane they already know. The Word add-in alternative is to insert with change tracking on;
  support both, prefer server-side for batch.

### Word add-in

- Office.js task pane. Shows findings for the open document, one click to apply a redline, one click
  to insert a library clause, "explain this clause", "what does our playbook say".
- Same backend as the web app, so every action is audited.

---

## 7. Security, privilege and governance

- **No training on bank data.** Contractually with the model provider and technically by design.
- **Data residency**: choose the hosting option that satisfies the regulator's outsourcing and
  data localisation rules. Treat the model provider as a material outsourcing if that is how the
  regulator classifies it, and do the due diligence up front.
- **Access control**: SSO, role-based access, matter-level permissions, ethical walls for
  conflicts, and a separate high-sensitivity tier (M&A, disputes, board matters) with a smaller
  access list and no retrieval into the general knowledge base.
- **Privilege**: mark matters and outputs as privileged where applicable, keep AI outputs inside
  the privileged workspace, and log who saw what.
- **Audit**: immutable log of every prompt, retrieved source, model output and human decision.
  This is what model risk management and internal audit will ask for.
- **Model risk management**: treat each skill as a model under the bank's model risk policy.
  Documented purpose, test results, limitations, owner, review cadence.
- **Human in the loop, always**: outputs are drafts. The UI says so. Nothing is sent externally
  by the system.
- **Prompt injection**: contracts from counterparties are untrusted input. Instructions in the
  document are never treated as instructions to the system. Test for this in the eval set.

---

## 8. Playbook format

Stored as YAML, edited through a form. Example in `playbooks/nda.example.yaml`. The shape:

```yaml
clause: limitation_of_liability
preferred:  "Counterparty liability uncapped for breach of confidentiality, data protection, IP; bank liability capped at fees paid in prior 12 months."
fallbacks:
  - "Mutual cap at 12 months' fees, with carve-outs for confidentiality, data protection, IP, gross negligence, wilful misconduct."
  - "Mutual cap at 24 months' fees with the same carve-outs."
walk_away: "Any cap on counterparty's data protection or confidentiality liability."
approver_for_exception: "Head of Legal, Commercial"
rationale: "Regulator's outsourcing rules require the bank to be able to recover for data incidents."
approved_text: clauses/lol_bank_customer_v3.md
```

---

## 9. Evaluation: how we know it works

Ship nothing without a measured baseline.

- **Golden set**: 50 to 100 real (anonymised) contracts per playbook, with a lawyer's own issues
  list and redlines. Built during Phase 0 from the last two years of matters.
- **Metrics per skill**: issue-spotting precision and recall against the lawyer's list; redline
  acceptance rate in production; time-to-first-draft; injection resistance; citation accuracy for
  Ask Legal.
- **Regression**: every change to a prompt, playbook or model re-runs the golden set. A drop blocks
  the release.
- **Production feedback**: each accept, reject or edit in Word is a labelled example. Review these
  monthly with the lawyers; they become new test cases.

---

## 10. Cost of running it

Rough per-task token economics, Opus 5 list prices, before cloud-provider margins:

| Task | Input tokens | Output tokens | Approx. cost |
|---|---|---|---|
| NDA review (10 pages) | 15k (mostly cached playbook) | 3k | USD 0.10 |
| Vendor agreement review (60 pages) | 60k | 10k | USD 0.60 |
| Facility agreement review (200 pages, schedules) | 200k | 20k | USD 1.50 |
| Draft from template | 20k | 8k | USD 0.30 |
| Ask Legal question | 30k retrieved | 1k | USD 0.20 |

A department doing 300 reviews and 1,000 questions a month would spend in the low hundreds of
dollars on inference. Engineering and hosting dominate the cost, not tokens.

---

## 11. Roadmap

**Phase 0, discovery (4–6 weeks)**
- Pick two contract types with volume (NDA and vendor agreement are the usual answer).
- Write the two playbooks with the lawyers. This is the real work.
- Build the golden set from past matters.
- Decide hosting with information security. Get the outsourcing and data classification approvals moving.

**Phase 1, review and redline MVP (8–12 weeks)**
- Web app with matters and upload, the document engine with tracked-change output, `contract-review`
  and `nda-review` skills, issues list, Word add-in read-only view of findings.
- Pilot with three to five lawyers. Measure against the golden set and against their own time.

**Phase 2, drafting and knowledge (8–12 weeks)**
- Template drafting, clause library, `ask-legal` with the policy and rulebook corpus, version compare,
  negotiation response, Word add-in with apply and insert.
- Extend playbooks to four to six contract types.

**Phase 3, operations (ongoing)**
- Intake and triage, obligations register, regulatory watch, outsourcing and data protection
  checklists, translation. Integrate with the DMS and ticketing.

**Team for Phases 0–2**
- One lawyer as product owner, half time. Two more lawyers contributing to playbooks, a few hours a week.
- Two backend engineers (one owns the document engine), one front-end engineer (web + Office add-in),
  one engineer on the model layer, prompts and evaluation.
- Information security and architecture review from the bank's existing teams.

---

## 12. Risks, stated plainly

- **Playbooks rot** if lawyers don't own them. Make the product owner a lawyer and make editing easy.
- **Tracked-change fidelity** on heavily formatted or scanned documents is hard. Budget for it and
  fall back to a clean-copy comparison when the source is bad.
- **Over-trust**. Lawyers will accept redlines without reading them once the tool is good. Keep the
  acceptance UI deliberate and audit acceptance rates by user.
- **Regulatory approval** for hosting can take longer than building. Start it in Phase 0.
- **Scope creep**. Every business unit will want its own chatbot. Hold the line at legal work until
  Phase 2 is measured.

---

## Repository layout (proposed)

```
docs/            architecture decisions, security design, eval methodology
playbooks/       one YAML per contract type, plus approved clause texts
skills/          one folder per skill: SKILL.md, schema.json, tests/
apps/web         Next.js web app
apps/word-addin  Office.js add-in
services/api     FastAPI backend, skill runner, model gateway
services/docs    document engine (parse, clause tree, tracked-change writer)
evals/           golden sets and the regression runner
```
