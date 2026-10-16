# Contract repository and key dates — build spec

*Replaces LEXZUR. Draft for the legal department and the M365 / Power Platform team, October 2026.*

## 1. What this is, and what it is not

The department sends dozens of documents out every day, and every one of them must end up in
SharePoint, in the folder for the right department and vendor. If an auditor looks for a document and it
is not there, the department gets a finding. Today that depends on each person remembering to upload.

The repository does four things:

1. **Filing**: anyone in the department copies a filing address on an email (or forwards it), and the
   documents are uploaded to the right `Department / Vendor / Agreement` folder in SharePoint,
   together with the email they travelled with. The vendor is identified by **reading the document**,
   not from the file name or the email chain.
2. **Completeness**: documents sent out but never filed are caught and the sender is nudged, and a
   monthly audit-readiness report shows gaps before an auditor does.
3. **Version trail**: for every agreement, the full sequence of rounds and the instructions behind
   them, so the department can always answer "why did legal draft it this way?"
4. **Key dates**: renewal, notice and other deadlines are extracted, confirmed by a lawyer, and
   reminded to the owner and a backup owner until someone records a decision.

Plus a **connector to Claude**, so lawyers and owners can query and file through Claude Enterprise
(section 9).

It is deliberately **not** a request or drafting workflow. Lawyers keep drafting and negotiating in
Word and Outlook exactly as today. The repository collects in the background; nobody fills a form to
get legal work started.

Built on what the bank already runs: SharePoint Online, Power Automate, Microsoft Entra ID, Teams, and
Claude (via the bank's approved route, see section 10).

---

## 2. Components

```
 Outlook ──CC/BCC or "File to repository" button──▶ legal-repository@ mailbox
                         │
                         ▼
             [Flow 1: Filing]
               1. read the document → who is the vendor?
               2. which department's agreement is it?   ── unsure ──▶ ask the sender (Teams card)
               3. which agreement, which round?                         │ no answer
                         │                                              ▼
                         ▼                                       "To file" queue (paralegal)
   Agreements / Department / Vendor / Agreement folder  ──▶ filing receipt to the sender
                         │
           new version ──┼──▶ [Flow 2: Negotiation log]  ── what changed, linked instruction
                         │
           executed ─────┴──▶ [Flow 3: Extraction] ── key dates + clause refs
                                        │
                                        ▼
                         Lawyer confirms dates (Teams card)
                                        │
                                        ▼
                     Key Dates list ──▶ [Flow 4: Reminders, daily]
                                        │
                                        ▼
                     Owner / backup / escalation ──▶ decision recorded

 Legal sent items ──▶ [Flow 6: Unfiled check, daily] ──▶ nudge the sender
 Everything ────────▶ [Flow 5: Housekeeping + audit-readiness report]

 Claude Enterprise ◀──▶ [Repository connector (MCP)] ◀──▶ SharePoint lists + library
```

| # | Component | Type |
|---|---|---|
| 1 | Vendors | SharePoint list (who each vendor is, every name it goes by) |
| 2 | Contract Register | SharePoint list (one row per agreement) |
| 3 | Key Dates | SharePoint list |
| 4 | Negotiation Log | SharePoint list |
| 5 | To File queue | SharePoint list |
| 6 | Agreements library | The existing library, `Department / Vendor / Agreement` folders |
| 7 | Standard Documents library | Existing library (templates, approved clauses) |
| 8 | Flows 1–6 | Power Automate |
| 9 | Repository connector | Remote MCP server in the bank's Azure tenant |

---

## 3. Data model

### 3.1 Vendors (one row per counterparty)

This list is what lets the filing flow recognise a vendor from the document itself, whatever the file is
called and whoever sent it.

| Column | Notes |
|---|---|
| VendorID | `VEN-0042` |
| DisplayName | The name used for folders, e.g. "Batelco" |
| LegalNames | Every legal name the entity has used, as it appears in contracts, in English and Arabic |
| Aliases | Trade names, brand names, group and former names, common abbreviations |
| RegistrationNumbers | Commercial registration or company numbers. The strongest identifier when a document shows one. |
| EmailDomains | Used only as a supporting signal, never on its own |
| Departments | The departments with agreements with this vendor (maintained by the flows) |
| RelatedVendors | Group companies that are separate counterparties (so a subsidiary is not filed under the parent) |

### 3.2 Contract Register (one row per agreement)

| Column | Type | Notes |
|---|---|---|
| ContractID | Text, unique | `CTR-2026-0142`. Generated on creation. |
| Title | Text | "SMS banking bundle 2026" |
| Vendor | Lookup → Vendors | |
| Department | Choice | IT, Retail, Operations, Treasury, … The department the agreement belongs to. |
| FolderPath | Text | `Agreements/Retail/Batelco/CTR-2025-0210 – SMS banking bundle` |
| ParentContract | Lookup → Register | For SOWs, amendments and side letters under a master agreement |
| References | Text | Vendor's and bank's reference numbers, PO numbers, the agreement date as cited in amendments. Used for matching. |
| ContractType | Choice | Lease, IT/SaaS, Outsourcing, NDA, Services, Facility, Other |
| BankEntity | Choice | Which bank entity is party |
| Owner | Person | Business owner who decides on renewal |
| BackupOwner | Person | Mandatory. Covers leave and departures. |
| LegalContact | Person | Lawyer responsible |
| Status | Choice | In negotiation · Signed (dates unconfirmed) · Active · Notice given · Expired · Terminated · Superseded |
| EffectiveDate | Date | |
| ExpiryDate | Date | Blank if evergreen |
| AutoRenews | Yes/No | |
| RenewalTerm | Number + unit | e.g. 12 months |
| NoticePeriod | Number + unit | e.g. 1 month, 90 days |
| NoticeDeadline | Date | Calculated, see 6.1. Never typed by hand. |
| NoticeClauseRef | Text | Clause number and the notice formalities (method, address) |
| MaterialOutsourcing | Yes/No | For the regulator's outsourcing register |
| Sensitivity | Choice | Standard · Restricted (restricted folders have narrower access) |
| DatesConfirmedBy / On | Person / Date | Set by the lawyer confirmation step |
| RenewalDecision | Choice | Renew · Renegotiate · Terminate · Pending |
| DecisionBy / On / Note | Person / Date / Text | |
| LexzurID | Text | Migration reference |

### 3.3 Key Dates (many per agreement)

Reminders run off this list, not the register, because one agreement has many dates.

| Column | Notes |
|---|---|
| ContractID | Lookup |
| DateType | Expiry · Notice deadline · Price/rent review · Option exercise · Insurance certificate due · Regulatory notification · Exit plan test · Other |
| DueDate | |
| SourceClause | Clause number + quoted text, from extraction |
| Recurring | None · Annual · Custom |
| ReminderProfile | Default, Lease, Outsourcing (see 6.2) |
| Confirmed | Yes/No |
| Status | Open · Actioned · Dismissed (with reason) |

### 3.4 Negotiation Log (one row per filed version)

ContractID, Seq, Date, Direction (OUT / IN / INT), Stage, DocumentLink, EmailLink, ChangeSummary,
LinkedInstruction (suggested link to the email that appears to explain a change),
InstructionConfirmedBy.

### 3.5 To File queue

EmailLink, Attachments, Sender, best guess (vendor, department, agreement, stage), confidence,
the evidence used, what is unresolved, AssignedTo, Resolution.

### 3.6 Filed Documents index

One row per filed file: SHA-256 hash of the file, ContractID, path, filed by, filed on, source email.
Used to skip duplicates and by the unfiled check (Flow 6).

---

## 4. Folder and file conventions

The flows follow the existing structure of the Agreements library: department first, then vendor, then
one folder per agreement. The same vendor appears under every department that has an agreement with it.

```
Agreements/
  IT/
    Batelco/
      CTR-2024-0031 – Managed network services MSA/
      CTR-2026-0142 – SD-WAN SOW/
  Retail/
    Batelco/
      CTR-2025-0210 – SMS banking bundle/
        01 2026-03-02 OUT Bank draft – SMS Bundle v1.docx
        01 2026-03-02 OUT Bank draft – email.eml
        02 2026-03-10 IN  Counterparty comments – Batelco comments.docx
        02 2026-03-10 IN  Counterparty comments – email.eml
        03 2026-03-11 INT Business instruction – email.eml
        04 2026-03-14 OUT Bank round 2 – SMS Bundle v2 (redline).docx
        04 2026-03-14 OUT Bank round 2 – email.eml
        05 2026-03-20 OUT Agreed clean – SMS Bundle final.docx
        06 2026-03-28 EXE Executed – SMS Bundle signed.pdf       [record]
```

- If the bank's current structure differs (e.g. vendor first, then department), the flows read
  `FolderPath` from the register, so they follow whatever structure is agreed. The structure is a
  setup decision, not a code change.
- File name = `Seq Date Direction Stage – original file name`. The original name is kept so it can be
  matched to what the counterparty has.
- **Each round is a separate file.** SharePoint version history is not used as the negotiation trail:
  drafts come back under different names, and retention policies can trim old versions.
- **The email is filed with the document**, as `.eml`. The email is usually the evidence of *why*
  ("business asked for a 24-month cap"), and mailbox retention may delete it later.
- Stages: Bank draft · Counterparty comments · Bank round N · Business instruction · Agreed clean ·
  Execution · Ancillary (side letters, amendments, notices given).
- The executed copy gets a retention label that declares it a record: it cannot be edited or deleted.
- SOWs, amendments and side letters under a master agreement get their own folder when they have their
  own dates, linked through `ParentContract`; otherwise they are filed as Ancillary in the master's folder.
- Restricted agreements live in a separate library with narrower permissions.

---

## 5. Flows

### Flow 1 — Filing (trigger: new email in `legal-repository@`)

**How people send documents to it**

- **CC or BCC** `legal-repository@` on the email that sends the document out. This is the normal route.
- **"File to repository" button** in Outlook (a Quick Step that forwards the selected email to the
  mailbox) for anything already sent, and for documents received.
- Optional tags in the subject or the first line of a forward override the flow's own identification:
  `[IT]`, `[Retail]`, `[CTR-2026-0142]`, `[instruction]` (a business instruction email with no attachment).

**Step 1 — Collect**

1. Export the email as `.eml`. Collect the attachments; drop inline images and signature logos.
2. Hash each attachment. If the hash is already in the Filed Documents index, do not file it again; the
   receipt says where it already is.
3. Each attachment is identified separately (one email can carry documents for two agreements).

**Step 2 — Identify the vendor from the document**

The file name and the email chain are often unhelpful ("Agreement_final_v3.docx", a forward with no
subject), so the vendor is identified from the document itself.

1. Claude reads the title, preamble and parties clause, the definitions of the parties, signature
   blocks, letterhead, and any registration numbers. It returns the counterparty's name and registration
   number as written, with the quoted text it relied on.
2. That is matched against the Vendors list, strongest first:
   - registration number (exact match);
   - legal name or alias (normalised: case, punctuation, "B.S.C.", "W.L.L.", "Ltd", Arabic and English forms);
   - external email domains on the email (supporting only, never decisive on its own).
3. Group companies are kept apart: if the document names a subsidiary that is its own row in Vendors,
   it is filed under that subsidiary, not the parent.
4. If the document names a counterparty that is not in Vendors, it goes to the To File queue as a
   possible new vendor. New vendors are added by a person.
5. If the document and the email chain point to different vendors, the document wins for the
   identification, but the item is sent for confirmation (step 5), because it may be the wrong attachment.

**Step 3 — Identify the department**

The same vendor can have agreements with several departments (Batelco with IT and with Retail), and the
document itself rarely says which department it is for. Signals, strongest first:

1. A tag from the sender (`[Retail]` or a ContractID).
2. **The document refers to an existing agreement**: "Amendment No. 2 to the Master Services Agreement
   dated 3 May 2023", a contract, PO or reference number. That agreement's department is used.
3. **Same email conversation** as an email already filed.
4. **Only one department** has agreements with this vendor.
5. **Who is on the email**: internal people in To/CC (the business requester) mapped to their
   department from Entra ID.
6. **Subject matter**: what the document is about compared with each department's existing agreements
   with that vendor (e.g. data-centre connectivity for IT against an SMS banking bundle for Retail).

Rules:

- One of signals 1–3 gives the answer → file.
- Only signals 4–6, and they agree with high confidence → file.
- Signals conflict, or confidence is low → **ask the sender** (step 5). Never guess silently.

**Step 4 — Identify the agreement and the round**

1. Within `Department / Vendor`, match the agreement: references in the document, same conversation,
   title and subject matter compared with the open agreements in that folder.
2. A new negotiation (no matching agreement) → a new register row and folder, created only after the
   sender confirms.
3. Stage and direction:
   - Direction is mechanical: sender in the bank and any recipient external → OUT; sender external → IN;
     all internal → INT.
   - Stage from the document and email (draft, comments, redline, clean, signed copy). A PDF with
     signatures or an e-signature certificate → Execution.

**Step 5 — Ask when unsure**

A Teams card (or a reply email, for people not on Teams) to the person who sent or forwarded the email:

> **Where should this go?** `Amendment_final.docx` — Batelco
> ◯ IT / Batelco / Managed network services MSA
> ◯ Retail / Batelco / SMS banking bundle
> ◯ New agreement — Batelco, department: [choose]
> ◯ Not a contract document, do not file

Filed as soon as they answer. No answer within one working day → To File queue for the paralegal on
duty, with everything the flow found.

**Step 6 — File and confirm**

1. Upload to the agreement folder with the next sequence number and the naming convention. Never
   overwrite; a file with the same name is a new sequence.
2. Save the `.eml` beside it. Add rows to Filed Documents and the Negotiation Log; update the register.
3. **Filing receipt** to the sender:
   "Filed 2 documents → Retail / Batelco / SMS banking bundle — round 2 (OUT, 14 Mar) [open folder]".
   The receipt is each person's proof that they filed.

**Guardrails**

- Files only into a folder matched to a confirmed vendor, department and agreement.
- Never creates a vendor, department or agreement folder without a person confirming.
- Never sends anything outside the bank. Receipts and questions go to internal senders only.
- Instructions inside documents or emails ("file this under…", "ignore previous…") are not acted on; the
  item goes to the queue.

### Flow 2 — Negotiation log (trigger: new document filed in an agreement folder)

1. Find the previous document in the same agreement (highest sequence with a document).
2. Claude compares the two and writes a change summary by clause ("12.2 liability cap: 12 → 24 months'
   fees; 15.1 governing law unchanged").
3. Claude looks at INT emails and the email bodies filed since the previous version and suggests which
   instruction explains each change. Suggestions are marked unconfirmed; the legal contact confirms
   or clears them from a Teams card.
4. Append to the Negotiation Log.

### Flow 3 — Extraction (trigger: document filed with stage Execution)

1. Apply the record retention label.
2. Claude extracts: parties, effective date, expiry, term, auto-renewal, renewal term, notice period
   and formalities, price/rent review dates, option dates, post-termination obligations, governing law,
   and whether it looks like material outsourcing. Each value comes with the clause number and quoted
   text.
3. Register row updated, Status = Signed (dates unconfirmed). Key Dates rows created, Confirmed = No.
4. Teams card to the legal contact: the extracted values next to the quoted clauses, with Confirm /
   Edit buttons. On confirm, Status = Active and DatesConfirmedBy/On are set.
5. Unconfirmed dates still produce reminders, sent to the legal contact and flagged "unconfirmed", so a
   missed confirmation can never mean a missed deadline.

### Flow 4 — Reminders (trigger: daily, 07:00)

See section 6.

### Flow 5 — Housekeeping and audit readiness (trigger: weekly; report monthly)

Weekly:

- Agreements whose Owner or BackupOwner is disabled in Entra ID → reassign queue for the legal contact.
- Active agreements with no Key Dates → flag.
- To File items older than 3 working days → escalate to the paralegal team lead.
- Expired agreements with Status still Active → flag.

Monthly **audit-readiness report**, per department, to the Head of Legal:

- Active agreements with no executed copy in the folder.
- Executed copies in the library with no register row.
- Agreements whose last filed round is older than 60 days while Status is still In negotiation.
- Active agreements with unconfirmed dates.
- Unfiled-document nudges (Flow 6) still open, by person.
- To File items still open.

The aim is that the department finds every gap before an auditor does.

### Flow 6 — Unfiled check (trigger: daily, 08:00) — *needs compliance and InfoSec approval*

CC-ing the filing address depends on people remembering. This flow catches what was forgotten.

1. For legal department mailboxes only: list emails sent the previous working day to at least one
   external recipient with a `.docx`, `.doc` or `.pdf` attachment.
2. Hash each attachment and check the Filed Documents index.
3. Anything not filed → one Teams card per person per day: "3 documents you sent yesterday are not
   filed", each with **File it** (runs Flow 1 on that email) and **Not a contract document**.
4. Unanswered nudges appear in the monthly audit-readiness report.

Scope is deliberately narrow: legal department mailboxes, sent items, emails with document attachments,
nothing else. Mailbox access is restricted to those mailboxes by an Exchange application access policy.
Staff are told it exists.

---

## 6. Reminder rules

### 6.1 The notice deadline

For anything that renews automatically, the date that matters is the **last day notice can be given**,
not the expiry date.

```
NoticeDeadline = ExpiryDate − NoticePeriod
```

- Months are subtracted as calendar months; days as calendar days unless the contract says business days.
- If the result falls on a weekend or a bank holiday, move it **earlier** to the previous working day.
- If the contract requires notice to be *received* by a date, and the method is courier or registered
  post, the reminder text says so and suggests sending at least 5 working days before.
- After a renewal, the next ExpiryDate and NoticeDeadline are rolled forward automatically and the
  owner is told.

### 6.2 Schedules

| Profile | Reminders before the deadline | Used for |
|---|---|---|
| Default | 90, 60, 30, 14, 7, 1 days | Most agreements |
| Lease | 180, 120, 90, 60, 30, 14, 7 days | ATM sites, branches (landlord negotiations take time) |
| Outsourcing | 180, 90, 60, 30, 14, 7 days | Material outsourcing (regulator notification and exit planning) |

### 6.3 Who gets what

- Every reminder goes to the **Owner**. From 30 days onward it also goes to the **BackupOwner**.
- Reminders are a Teams card (with an email copy) showing: agreement, the date and what it means, the
  notice clause text and how notice must be given, a link to the folder, and buttons: **Renew**,
  **Renegotiate**, **Terminate**, **Not my contract**.
- No decision by 30 days → also the owner's manager and the legal contact.
  No decision by 7 days → Head of Legal.
- A decision stops the reminders for that date and is written to the register. Terminate or
  Renegotiate opens a task for the legal contact.
- "Not my contract" goes to the legal contact to reassign.

---

## 7. Setup and migration from LEXZUR

1. **Build the Vendors list and register from what exists**: walk the existing Agreements library
   (`Department / Vendor` folders) and the LEXZUR export. For each vendor folder, Claude reads the
   executed copies to collect legal names and registration numbers as they appear in the contracts.
   A paralegal reviews the resulting Vendors list before go-live; it drives all filing.
2. Request a full export from LEXZUR support: contract records with all fields, reminders, owners, and
   all attached documents. (Check LEXZUR's own licence renewal date and notice period while at it.)
3. Load the LEXZUR records into the register with `LexzurID`, matched to the existing folders, and copy
   any documents that exist only in LEXZUR into the right folders.
4. Run Flow 3 extraction over every executed copy and **compare the extracted dates with LEXZUR's**.
5. Each mismatch (different expiry, missing notice period, unrecorded auto-renewal, no owner) goes to a
   lawyer to resolve. Expect to find agreements already at risk.
6. Run the first audit-readiness report. It is the baseline: what is missing today.
7. Run both systems in parallel for one to two months. Compare reminders. Then switch LEXZUR off and keep
   the export as an archive.

---

## 8. How Claude is used in the flows

- Every call uses structured outputs, so flows receive validated JSON (vendor, department, agreement,
  stage, dates with clause references, and the evidence for each), never prose to parse.
- Every identification and extracted value carries the quoted text it came from, so a person can check
  by reading, not hunting.
- Documents and emails from counterparties are untrusted input. Text in them that looks like an
  instruction is ignored and reported, never acted on. This is tested before go-live.
- Claude never sends anything externally, never deletes, never creates vendors or folders on its own,
  and never confirms its own dates.

---

## 9. Connector to Claude (remote MCP server)

### Why a dedicated connector

Claude Enterprise's Microsoft 365 connector can search SharePoint files and Outlook mail, which helps.
It does not know the register, the vendors, the rounds, the notice-deadline logic, or the confirmation
rules. A dedicated connector gives Claude contract-aware tools, so a lawyer can ask in Claude:

- "Which agreements need a notice decision before the end of Q1, and who owns them?"
- "What agreements do we have with Batelco, across all departments?"
- "Show me how the liability cap in the Site 14 ATM lease changed, round by round, and who asked for it."
- "File the attached markup against the Retail Batelco SMS agreement as counterparty comments."
- "Review this NDA against our standard NDA" (Claude pulls the standard from the Standard Documents library).

The two connectors are used side by side.

### Tools

Read (available to all repository users, filtered by their own permissions):

| Tool | Returns |
|---|---|
| `search_contracts(query, vendor?, department?, type?, status?, owner?)` | Matching register rows |
| `get_vendor(name_or_id)` | Vendor row and its agreements in every department |
| `get_contract(contract_id)` | Register row, key dates, owner and backup, folder link |
| `list_versions(contract_id)` | Every filed item in order: seq, date, direction, stage, links |
| `get_document(item_id)` | Document text with paragraph numbers, plus a link to the file |
| `get_email(item_id)` | Filed email: headers and body |
| `compare_versions(item_a, item_b)` | Deterministic paragraph diff between two filed documents |
| `get_negotiation_log(contract_id)` | Change summaries and confirmed/suggested instructions |
| `upcoming_deadlines(days, owner?, department?, type?)` | Open key dates in the window, notice deadlines first |
| `audit_gaps(department?)` | The audit-readiness checks from Flow 5, live |
| `get_standard_document(type)` | Current approved template from the Standard Documents library |

Write (legal and paralegal roles only; Claude asks the user to approve each call):

| Tool | Effect |
|---|---|
| `file_document(contract_id, stage, file, email_ref?)` | Files a document into the agreement folder with the next sequence number |
| `create_contract(fields)` | Creates a register row and folder in status In negotiation |
| `confirm_key_dates(contract_id, dates)` | Lawyer confirmation (only the legal contact or legal leads) |
| `record_decision(key_date_id, decision, note)` | Records Renew / Renegotiate / Terminate |
| `reassign(contract_id, owner?, backup_owner?)` | Changes owners |

No delete tool. No tool sends email.

### How it is built

- **Server**: a small remote MCP server (Python or TypeScript) on Azure App Service or Azure Functions,
  behind Azure API Management. It talks to SharePoint through Microsoft Graph.
- **Authentication**: OAuth with Microsoft Entra ID. The user signs in once when they add the connector in
  Claude. The server calls Graph **on behalf of that user** (delegated permissions), so Claude can only
  see what that person can already see in SharePoint, and restricted folders stay restricted.
- **Registration**: an organisation owner adds it in Claude Enterprise as a custom connector (the
  server's URL plus the OAuth client details).
- **Network**: Claude's servers call the connector, so the endpoint has to be reachable from Anthropic's
  published IP ranges. Allow only those ranges at API Management. This is an InfoSec decision.
- **Audit**: every tool call is logged with the user, tool, arguments, and items returned.
- **Untrusted content**: documents and emails returned by the tools come from counterparties. The server
  returns them as data, and the connector's tool descriptions say so.

---

## 10. Security and approvals checklist

- [ ] Graph application permissions for the flows limited to the repository site (`Sites.Selected`) and
      the filing mailbox only (Exchange application access policy restricting `Mail.Read` to that mailbox).
- [ ] Flow 6 only: read access to legal department mailboxes' sent items, restricted by an application
      access policy to those mailboxes; approved by compliance and data protection; staff informed.
- [ ] Delegated Graph permissions for the connector, consented by an admin.
- [ ] Route to Claude approved (Anthropic API with the bank's data processing terms, or Claude in the
      bank's cloud through Microsoft Foundry or another approved provider), and data residency checked.
- [ ] Connector endpoint exposure approved (IP allowlist, API Management, logging).
- [ ] Retention labels: record label for executed copies; retention period for drafts and emails agreed
      with records management.
- [ ] Privilege and access: restricted library for sensitive matters; business-to-legal emails treated
      as privileged.
- [ ] The identification and extraction steps registered under the model risk policy, with test results.
- [ ] Power Automate premium licences for the flow owners (the HTTP action to call Claude is premium).

---

## 11. Phasing

| Phase | Scope | Result |
|---|---|---|
| A | Vendors list and register built from the existing library and LEXZUR; Flow 1 filing with receipts; Flow 3 extraction; Flow 4 reminders; first audit-readiness report | Every document sent out can be filed with one CC; reminders run off confirmed dates; LEXZUR can be switched off at the end of A |
| B | Flow 6 unfiled check; Flow 2 negotiation log; Flow 5 monthly report; read-only connector | Forgotten documents are caught; the "why" trail exists; Claude can answer questions about the repository |
| C | Write tools in the connector; Power BI view for the GC | Filing and decisions from inside Claude |

---

## 12. Open questions for the department

1. Is the existing Agreements library organised `Department / Vendor`, `Vendor / Department`, or
   something else? Which departments are in scope?
2. Who owns the build: the bank's Power Platform team, or an external partner?
3. Is Flow 6 (checking legal department sent items for unfiled documents) acceptable to compliance and
   data protection, or should filing stay CC-only?
4. Which bank-holiday calendar applies to deadlines, and are there entities in other jurisdictions?
5. Reminder profiles: are the schedules in 6.2 right for leases and outsourcing?
6. Who is the paralegal on duty for the To File queue and new-vendor confirmations?
7. Retention period for drafts and emails once an agreement has expired.
