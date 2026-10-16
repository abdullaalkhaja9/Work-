# Contract repository and key dates — build spec

*Replaces LEXZUR. Draft for the legal department and the M365 / Power Platform team, October 2026.*

## 1. What this is, and what it is not

The repository does three things:

1. **Capture**: every draft, markup, clean and executed copy is filed automatically, together with the
   email it travelled with, including documents legal *sends*.
2. **Version trail**: for every contract, the full sequence of rounds and the instructions behind them,
   so the department can always answer "why did legal draft it this way?"
3. **Key dates**: renewal, notice and other deadlines are extracted, confirmed by a lawyer, and
   reminded to the owner and a backup owner until someone records a decision.

Plus a **connector to Claude**, so lawyers and owners can query and file through Claude Enterprise
(section 9).

It is deliberately **not** a request or drafting workflow. Lawyers keep drafting and negotiating in
Word and Outlook exactly as today. The repository collects in the background; nobody fills a form to
get legal work started.

Built on what the bank already runs: SharePoint Online, Power Automate, Microsoft Entra ID, Teams, and
Claude (via the bank's approved route, see section 11).

---

## 2. Components

```
 Outlook ──CC/BCC──▶ legal-repository@ mailbox
                         │
                         ▼
             [Flow 1: Filing]  ── Claude: match contract, stage, direction
                         │                     │ low confidence
                         ▼                     ▼
          Contract folder (SharePoint)     "To file" queue (paralegal)
                         │
           new version ──┼──▶ [Flow 2: Negotiation log]  ── Claude: what changed, linked instruction
                         │
           executed ─────┴──▶ [Flow 3: Extraction] ── Claude: key dates + clause refs
                                        │
                                        ▼
                         Lawyer confirms dates (Teams card)
                                        │
                                        ▼
                     Key Dates list ──▶ [Flow 4: Reminders, daily]
                                        │
                                        ▼
                     Owner / backup / escalation ──▶ decision recorded

 Claude Enterprise ◀──▶ [Repository connector (MCP)] ◀──▶ SharePoint lists + library
```

| # | Component | Type |
|---|---|---|
| 1 | Contract Register | SharePoint list |
| 2 | Key Dates | SharePoint list |
| 3 | Negotiation Log | SharePoint list |
| 4 | To File queue | SharePoint list |
| 5 | Contracts library | SharePoint document library, one folder per contract |
| 6 | Standard Documents library | Existing library (templates, approved clauses) |
| 7 | Flow 1–5 | Power Automate |
| 8 | Repository connector | Remote MCP server in the bank's Azure tenant |

---

## 3. Data model

### 3.1 Contract Register (one row per contract)

| Column | Type | Notes |
|---|---|---|
| ContractID | Text, unique | `CTR-2026-0142`. Generated on creation. |
| Title | Text | "ATM lease – Site 14 – [Landlord]" |
| Counterparty | Text | Legal entity name as in the contract |
| ContractType | Choice | Lease, IT/SaaS, Outsourcing, NDA, Services, Facility, Other |
| BankEntity | Choice | Which bank entity is party |
| BusinessUnit | Choice | |
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
| FolderLink | Hyperlink | Contract folder |
| LexzurID | Text | Migration reference |

### 3.2 Key Dates (many per contract)

Reminders run off this list, not the register, because one contract has many dates.

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

### 3.3 Negotiation Log (one row per filed version)

ContractID, Seq, Date, Direction (OUT / IN / INT), Stage, DocumentLink, EmailLink, ChangeSummary
(Claude), LinkedInstruction (Claude's suggestion: link to the email that appears to explain a change),
InstructionConfirmedBy.

### 3.4 To File queue

EmailLink, Attachments, Claude's best guess (contract, stage, confidence, reasoning), AssignedTo,
Resolution.

---

## 4. Folder and file conventions

One folder per contract in the Contracts library, named `ContractID – Title`.

```
CTR-2026-0142 – ATM lease – Site 14 – [Landlord]/
  01 2026-03-02 OUT Bank draft – ATM Lease v1.docx
  01 2026-03-02 OUT Bank draft – email.eml
  02 2026-03-10 IN  Counterparty comments – Lease_LL comments.docx
  02 2026-03-10 IN  Counterparty comments – email.eml
  03 2026-03-11 INT Business instruction – email.eml
  04 2026-03-14 OUT Bank round 2 – ATM Lease v2 (redline).docx
  04 2026-03-14 OUT Bank round 2 – email.eml
  05 2026-03-20 OUT Agreed clean – ATM Lease final.docx
  06 2026-03-28 EXE Executed – ATM Lease signed.pdf       [record]
```

- File name = `Seq Date Direction Stage – original file name`. The original name is kept so it can be
  matched to what the counterparty has.
- **Each round is a separate file.** SharePoint version history is not used as the negotiation trail:
  drafts come back under different names, and retention policies can trim old versions.
- **The email is filed with the document**, as `.eml`. The email is usually the evidence of *why*
  ("business asked for a 24-month cap"), and mailbox retention may delete it later.
- Stages: Bank draft · Counterparty comments · Bank round N · Business instruction · Agreed clean ·
  Execution · Ancillary (side letters, amendments, notices given).
- The executed copy gets a retention label that declares it a record: it cannot be edited or deleted.
  Amendments are new contracts linked to the original, or ancillary documents in the same folder.
- Restricted contracts live in a separate library with narrower permissions.

---

## 5. Flows

### Flow 1 — Filing (trigger: new email in `legal-repository@`)

1. Ignore if no attachments and not marked as an instruction (lawyers can forward a business
   instruction email with no attachment; the subject tag `[instruction]` or a forward to the mailbox
   with no attachment counts).
2. Export the email as `.eml`. Collect attachments; drop inline images and signature logos.
3. Send Claude: email headers, body, attachment text (first pages are enough to identify), and a
   shortlist of candidate contracts from the register (same counterparty domain, similar title, same
   Outlook conversation ID as an already filed email).
4. Claude returns (structured output): `contract_id` or `new_contract`, `stage`, `direction`,
   `confidence`, `reasoning`.
   - Direction is mechanical where possible: sender is bank domain and any recipient is external →
     OUT; sender external → IN; all internal → INT.
5. Confidence ≥ threshold and existing contract → file into the folder with the next sequence number.
   Otherwise → To File queue with Claude's guess, and a Teams notification to the paralegal on duty.
6. `new_contract` is never created automatically. It goes to the queue with a pre-filled register entry
   for the paralegal to confirm.
7. Same Outlook conversation ID as an already filed email → treated as high confidence for the same
   contract.

The filing mailbox is opt-in: lawyers CC or BCC it. No one's personal mailbox is monitored.

### Flow 2 — Negotiation log (trigger: new document filed in a contract folder)

1. Find the previous document in the same contract (highest sequence with a document).
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

### Flow 5 — Housekeeping (trigger: weekly)

- Contracts whose Owner or BackupOwner is disabled in Entra ID → reassign queue for the legal contact.
- Contracts Active with no Key Dates → flag.
- To File items older than 3 working days → escalate to the paralegal team lead.
- Expired contracts with Status still Active → flag.

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
| Default | 90, 60, 30, 14, 7, 1 days | Most contracts |
| Lease | 180, 120, 90, 60, 30, 14, 7 days | ATM sites, branches (landlord negotiations take time) |
| Outsourcing | 180, 90, 60, 30, 14, 7 days | Material outsourcing (regulator notification and exit planning) |

### 6.3 Who gets what

- Every reminder goes to the **Owner**. From 30 days onward it also goes to the **BackupOwner**.
- Reminders are a Teams card (with an email copy) showing: contract, the date and what it means, the
  notice clause text and how notice must be given, a link to the folder, and buttons: **Renew**,
  **Renegotiate**, **Terminate**, **Not my contract**.
- No decision by 30 days → also the owner's manager and the legal contact.
  No decision by 7 days → Head of Legal.
- A decision stops the reminders for that date and is written to the register. Terminate or
  Renegotiate opens a task for the legal contact.
- "Not my contract" goes to the legal contact to reassign.

---

## 7. Migration from LEXZUR

1. Request a full export from LEXZUR support: contract records with all fields, reminders, owners, and
   all attached documents. (Check LEXZUR's own licence renewal date and notice period while at it.)
2. Load contract records into the register with `LexzurID`, and documents into folders (as stage
   Execution where the document is the signed copy, Ancillary otherwise).
3. Run Flow 3 extraction over every executed copy and **compare Claude's dates with LEXZUR's**.
4. Each mismatch (different expiry, missing notice period, unrecorded auto-renewal, no owner) goes to a
   lawyer to resolve. Expect to find contracts already at risk.
5. Run both systems in parallel for one to two months. Compare reminders. Then switch LEXZUR off and keep
   the export as an archive.

---

## 8. How Claude is used in the flows

- Every call uses structured outputs, so flows receive validated JSON (contract, stage, dates with
  clause references), never prose to parse.
- Every extracted value carries the clause reference and quoted text, so a lawyer can confirm by reading,
  not hunting.
- Documents and emails from counterparties are untrusted input. Text in them that looks like an
  instruction is ignored and reported, never acted on. This is tested before go-live.
- Claude never sends anything externally, never deletes, and never confirms its own dates.

---

## 9. Connector to Claude (remote MCP server)

### Why a dedicated connector

Claude Enterprise's Microsoft 365 connector can search SharePoint files and Outlook mail, which helps.
It does not know the register, the rounds, the notice-deadline logic, or the confirmation rules. A
dedicated connector gives Claude contract-aware tools, so a lawyer can ask in Claude:

- "Which contracts need a notice decision before the end of Q1, and who owns them?"
- "Show me how the liability cap in the Site 14 lease changed, round by round, and who asked for it."
- "File the attached markup against the [Vendor] SaaS agreement as counterparty comments."
- "Review this NDA against our standard NDA" (Claude pulls the standard from the Standard Documents library).

The two connectors are used side by side.

### Tools

Read (available to all repository users, filtered by their own permissions):

| Tool | Returns |
|---|---|
| `search_contracts(query, counterparty?, type?, status?, owner?)` | Matching register rows |
| `get_contract(contract_id)` | Register row, key dates, owner and backup, folder link |
| `list_versions(contract_id)` | Every filed item in order: seq, date, direction, stage, links |
| `get_document(item_id)` | Document text with paragraph numbers, plus a link to the file |
| `get_email(item_id)` | Filed email: headers and body |
| `compare_versions(item_a, item_b)` | Deterministic paragraph diff between two filed documents |
| `get_negotiation_log(contract_id)` | Change summaries and confirmed/suggested instructions |
| `upcoming_deadlines(days, owner?, type?)` | Open key dates in the window, notice deadlines first |
| `get_standard_document(type)` | Current approved template from the Standard Documents library |

Write (legal and paralegal roles only; Claude asks the user to approve each call):

| Tool | Effect |
|---|---|
| `file_document(contract_id, stage, file, email_ref?)` | Files a document into the folder with the next sequence number |
| `create_contract(fields)` | Creates a register row in status In negotiation |
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
- [ ] Delegated Graph permissions for the connector, consented by an admin.
- [ ] Route to Claude approved (Anthropic API with the bank's data processing terms, or Claude in the
      bank's cloud through Microsoft Foundry or another approved provider), and data residency checked.
- [ ] Connector endpoint exposure approved (IP allowlist, API Management, logging).
- [ ] Retention labels: record label for executed copies; retention period for drafts and emails agreed
      with records management.
- [ ] Privilege and access: restricted library for sensitive matters; business-to-legal emails treated
      as privileged.
- [ ] The extraction and filing steps registered under the model risk policy, with test results.
- [ ] Power Automate premium licences for the flow owners (the HTTP action to call Claude is premium).

---

## 11. Phasing

| Phase | Scope | Replaces in LEXZUR |
|---|---|---|
| A | Register, Key Dates, Flow 3 extraction, Flow 4 reminders, migration with cross-check, read-only connector (`search_contracts`, `get_contract`, `upcoming_deadlines`) | Storage and reminders, so LEXZUR can be switched off at the end of A |
| B | Filing mailbox (Flow 1), version folders, Negotiation Log (Flow 2), connector version tools | Nothing in LEXZUR; this is new |
| C | Write tools in the connector, Flow 5 housekeeping, Power BI view for the GC | — |

Phase A delivers the most value soonest: it removes the risk of missed notice deadlines and gets off
LEXZUR.

---

## 12. Open questions for the department

1. Which contract types are in scope for the first migration? (Suggest: everything with a date.)
2. Who owns the build: the bank's Power Platform team, or an external partner?
3. Which bank-holiday calendar applies to deadlines, and are there entities in other jurisdictions?
4. Reminder profiles: are the schedules in 6.2 right for leases and outsourcing?
5. Who is the paralegal on duty for the To File queue?
6. Retention period for drafts and emails once a contract has expired.
