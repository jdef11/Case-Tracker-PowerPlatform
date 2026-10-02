# Power Platform Plan — OFC Case Tracker (v2)

**Date:** 2026-10-02
**Status:** Draft for review.
**Supersedes:** v1 (`POWER_PLATFORM_PLAN.md` on branch `claude/sweet-wright-az0d4y` of `jdef11/Case-Tracker`, kept intact) and `AZURE_MIGRATION_PLAN.md` in that repo.
**Source system studied:** the legacy Next.js app in `jdef11/Case-Tracker` (`ofc-case-tracker/`). Business rules below come from its `CLAUDE.md`, `prisma/schema.prisma`, the source tree and `notifications.ts`. The individual API route handlers have **not** been read yet, so Phase 0 verifies every rule against the code.

## What changed from v1

- Power Fx "functions" / low-code plug-ins are **preview** (Microsoft: "not meant for production use"), so they are dropped. A small **C# plug-in** is the only production-grade atomic option.
- The stage board is a **custom page with click-to-move** (no PCF, no drag-and-drop). Drag-and-drop is explicitly out of scope.
- Word documents: **native model-driven Word Templates are the default**, with the Power Automate flow as fallback, decided by a proof of concept (section 8).
- PHI controls tightened after checking Microsoft's docs: secured columns are readable by System Administrators by default, and read-audit logs go to Purview and do not reliably identify records in list queries or exports.
- The BAA and Entra are no longer listed as differentiators (see section 1).

## Decisions already made

| Decision | Choice |
|---|---|
| Front end | Model-driven app on Dataverse |
| Stage movement | Click-to-move. Drag-and-drop not required |
| Word documents | Whatever works best, decided by proof of concept |
| Users | Internal OPM staff only, fewer than ~50 |
| Current system of record | A SharePoint list (exportable to Excel/CSV). The Next.js app was never in production, so there is no encrypted data to carry over |
| Environments | Dedicated Dataverse dev/test/prod already exist |

---

## 1. What is wrong with this plan (read first)

1. **The BAA and Entra do not favor Power Platform over Azure.** Microsoft's HIPAA BAA is included through the Product Terms and Data Protection Addendum and covers in-scope services across Azure, Microsoft 365, Dynamics 365 and Power Platform. The old Azure plan used the same BAA and the same Entra tenant. The real reasons for Power Platform are lower hosting and patching burden plus built-in roles and auditing. A BAA also does not by itself make you compliant, and coverage is per in-scope service, so every connector and feature used must be checked.
2. **It is a rewrite, not a port.** None of the Next.js, API or Prisma code transfers. What transfers is the schema, the business rules, the Word templates and the stage and field-option seed data.
3. **You lose application-layer PHI encryption.** The legacy design encrypts patient names so a database admin cannot read them. In Dataverse, data is encrypted at rest, but **by default only System Administrators can read a secured column** (Microsoft docs), so admins can see PHI. Controls become segregation, field security, role scoping, auditing and a very small admin group. These are weaker against a privileged insider.
4. **Read auditing is weaker than the legacy "reveal" log.** Dataverse read logs go to Microsoft Purview, not Audit History. Microsoft documents that `RetrieveMultiple` and `ExportToExcel` do not reliably include individual record identifiers, so record-level access tracking for list queries and exports is not supported. Opening a single record is reliably logged; views and exports are not. Remove Export to Excel from the roles and keep PHI out of views.
5. **"Entirely Power Platform" is not fully achievable.** The design-phase lock (stages 1–8 move together) is an atomic multi-row rule. A flow cannot make it transactional. It needs a **C# Dataverse plug-in**, which means a .NET toolchain and a small amount of unit-tested pro-code.
6. **The stage board is simplified.** Click-to-move via a custom page. No drag-and-drop.
7. **Word template output is unproven.** Both options (native Word Templates and Power Automate) need a proof of concept on your five real templates (section 8).
8. **PHI is in a SharePoint list today.** If it is plaintext, the exported CSV/Excel contains PHI. Handle the export under your PHI procedures and **never put it in this repo or in a Claude session**. Use the anonymized sample for development and test.
9. **Recurring cost replaces infrastructure cost.** Premium licensing, Dataverse capacity (audit logs consume log capacity) and possibly Power Automate licensing. No prices are quoted here; confirm with your licensing contact.
10. **Validation burden.** The tracker holds design-transfer and qualification checklists (QMSF-1400-x forms), so it is probably a QMS software application. That likely brings in software validation (ISO 13485 §4.1.6, QMSR) and possibly 21 CFR Part 11. QA/RA must decide. SaaS release waves change the platform under a validated state, so you need a supplier assessment and a periodic re-verification process (section 10).
11. **Notifications are new build work.** The legacy email path is a `console.log` stub, and the Teams call posts a legacy MessageCard to an incoming webhook that Microsoft has been retiring (verify dates).

What works well: this is a relational workflow tracker with roles, an audit trail and a checklist, which is what Dataverse and model-driven apps are built for. Role scoping, auditing, users and auth become configuration instead of code.

---

## 2. Target architecture

```
Entra ID (security groups)
   │  group-backed teams → Dataverse security roles
   ▼
Model-driven app "OPM Case Tracker"  ──  Custom page: stage lanes + click-to-move + bulk move
   │
   ▼
Dataverse (dev → test → prod, managed solution)
   ├─ Tables (section 4), PHI isolated in opm_casepatient
   ├─ Security roles, field security profile, Entra-group teams
   ├─ Auditing (create/update/delete) + Purview read logs
   ├─ C# plug-in + Custom API: opm_AdvanceStage (design lock, transitions)
   ├─ Word documents: native Word Templates (default) or Power Automate (fallback)
   └─ Power Automate flows: checklist creation, Teams/Outlook notifications
```

---

## 3. Component mapping

| Today | Power Platform |
|---|---|
| Next.js UI | Model-driven app; custom page for the stage lanes and bulk move |
| API routes | Dataverse Web API (built in) + one Custom API for stage advance |
| Prisma + PostgreSQL | Dataverse tables |
| NextAuth + Entra, `AUTH_DEV_BYPASS` | Entra ID natively; dev bypass disappears |
| `User` table, role enum | Dataverse `systemuser` + security roles assigned via Entra-group teams |
| Middleware role checks | Security roles and table privileges |
| Designer sees own cases (3 code locations) | User-level read privilege on `opm_case`, owner = designer |
| AES-256-GCM patient name | Encryption at rest + isolated PHI table + field security + role access + Purview read logs |
| `logAudit()` + `AuditLog` | Native Dataverse auditing plus Purview |
| `reveal-patient` endpoint | Purview read log when the PHI record is opened (single-record retrieve only) |
| `FieldOption` + admin page | Lookup tables or choices, edited in an Admin area of the app |
| `DocumentTemplate` + `fileData` | Checklist metadata in `opm_documenttemplate`; the Word file itself lives in Dataverse Document Templates (native option) or a SharePoint library (flow option) |
| `docx-template.ts` | Native Word Templates (default) or Power Automate Word fill (fallback) |
| `notifications.ts` | Power Automate to Teams and Outlook |
| `sla-summary` report | Views, charts, dashboard (Power BI only if needed) |
| Drag-and-drop `StageTable` / `StageNavStrip` | Custom page with stage lanes and Move buttons (click-to-move) |
| Admin: Users / Stages / Audit Log | Entra groups / `opm_stage` table / native audit summary |
| `scripts/*` seeds | Solution configuration data (Configuration Migration Tool) |
| Key Vault / `.env` | Environment variables + connection references. No PHI key to manage |
| GitHub Actions to App Service | Power Platform Pipelines, or GitHub Actions with the Power Platform actions |

---

## 4. Data model (publisher prefix `opm_`, to be confirmed)

| Table | Notes |
|---|---|
| `opm_case` | Primary name = case number (alternate key). Surgeon, finish type, priority, status (choice: WIP / On Hold / Complete / Canceled), surgery/start date, lead time, designer (= **owner**), sales rep, rep company, notes, part count, quoted, country, hospital, current stage (lookup), stage-entered-on, time-target due-on |
| `opm_casepatient` | **PHI, 1:1 with case.** Patient name, patient DOB, both on a field security profile. Own privilege set. Never shown in list views |
| `opm_subcomponent` | Case (parental, cascade), suffix, full number, current stage, stage-entered-on, due-on, notes. Alternate key (case, suffix) |
| `opm_stage` | Stage number (alt key), name, phase (design/manufacturing), `opm_timetargethours`, sort order. Never label as "SLA" |
| `opm_stagetransition` | Case/subcomponent, from/to stage, notes. **Changed by / changed at = system `createdby`/`createdon`.** Create+read only for non-admins |
| `opm_implanttype` | Hierarchical (group / subgroup / value). N:N to case replaces the JSON array |
| `opm_surgeon`, `opm_salesrep`, `opm_finishtype` | Small admin-maintained lookup tables (sort order, active flag). Priority stays a global choice |
| `opm_documenttemplate` | Checklist metadata: name, form number, sort order, active. Linked to the Word template record if native Word Templates are used |
| `opm_casedocument` | Case, template, completed, completed-at, completed-by, notes. Alternate key (case, template). Generated-file column only if the flow option is chosen |
| `opm_notificationpreference` | User, per-event and per-channel flags |

Replaced by platform features: `User` (systemuser), `AuditLog` (auditing + Purview), `Case.designerId` (owner).

Open design choice: surgeons, sales reps and finish types as tables (admin-editable at runtime, as today) versus global choices (simpler, deploy-time changes only). Tables preserve current behavior and are recommended.

---

## 5. Security model

- **Roles:** `OPM Admin` (org-level, config tables), `OPM Manager` (org-level read/write on cases, no delete), `OPM Designer` (user-level on `opm_case`; children inherit via parental relationship), `OPM PHI Reader` (added to whichever roles may see patient identity).
- **Assignment:** Entra security groups mapped to Dataverse group teams, so access is managed in Entra and no custom user-admin screen is needed. The legacy `sales_rep` role is dropped unless told otherwise.
- **PHI columns:** field security profile on patient name and DOB. Because System Administrators can read secured columns by default, keep that group to the minimum, use PIM if available, and review membership regularly. Consider a masking rule so casual views show masked values.
- **Exports:** remove **Export to Excel** and other bulk-export privileges from all non-admin roles. Treat Word template generation as an export of PHI (see section 8).
- **Transition table:** immutable for non-admins.
- **Stage columns:** not directly editable on forms. All changes go through the Custom API; a plug-in guard also catches grid or import edits.
- **Environment:** put dev/test/prod under a Managed Environment with a DLP policy that blocks consumer connectors. Confirm that every connector and feature used is within your BAA scope. Do not use preview features with PHI.
- **Notification content:** case number only. No patient name, DOB or surgeon in Teams or email bodies.

---

## 6. Business logic

1. **Advance stage (`opm_AdvanceStage` Custom API + C# plug-in).** Inputs: case or subcomponent, target stage, optional note. In one transaction:
   - if target stage ≤ 8, move **all** sibling subcomponents and write a transition row for each;
   - if target stage ≥ 9, move only that subcomponent;
   - for standalone cases, move the case;
   - set stage-entered-on and due-on (stage-entered-on + time target).
   Unit-tested in .NET. A second plug-in step blocks direct writes to the stage columns from any other path. A flow-based alternative was rejected because it is eventually consistent and can leave siblings out of sync; revisit only if QA explicitly accepts that.
2. **Time target / Overdue.** Store `due-on` and filter "Overdue" views on `due-on` ≤ today. Verify whether a calculated column can express this.
3. **Document checklist.** An async flow on case create builds a `casedocument` row per active template. The legacy lazy backfill is replaced by a one-time backfill flow. The checklist may take a few seconds to appear on a new case.
4. **Terminology.** Keep "WIP", "Time Target", "Overdue" and "Parts Overdue" in display names. Never use "SLA".

---

## 7. Model-driven app

- **Areas:** Dashboard, Cases, Stages, Admin (Stages, implant types, surgeons, sales reps, finish types, document templates, audit).
- **Case form tabs:** Details, Subcomponents (subgrid), Documents (editable subgrid with a completed toggle), History (read-only transitions), Patient (PHI, gated by `OPM PHI Reader`).
- **Stage movement (click-to-move):** a custom page showing 21 stage lanes with case/part cards, a **Move to stage** control on each card, and multi-select for bulk move. Backward moves require a note. "Undo" becomes a reverse-move action. All moves call `opm_AdvanceStage`.
- **Dashboards:** My cases, Overdue parts, Cases by stage, Cases by status.
- **Not built:** drag-and-drop and the 30-second auto-refresh.

---

## 8. Word documents (decided by proof of concept)

**Default: native model-driven Word Templates.** Templates are authored in Word via the XML Mapping Pane against the Dataverse schema (Developer tab), uploaded under Document Templates, and generated from the Word Templates command on a case. Repeating table rows are supported (useful for the subcomponent list). No flow, no SharePoint hop, no connector scope question.

**Fallback: Power Automate "Populate a Microsoft Word template".** Blank templates in a SharePoint library; a flow fills them and returns the file. I could not confirm from the docs whether this action matches content controls by tag or by title, so do not assume the legacy tags carry over.

**Proof of concept (Phase 0)**, using the five templates in `templates/`:

| Criterion | Pass condition |
|---|---|
| Field coverage | All mapped fields populate, including those in table cells |
| Formatting | Layout, widths and alignment preserved |
| Control types | Templates work with only Plain Text and Picture controls (Microsoft warns other control types can freeze Word when authoring native templates); otherwise native is out |
| PHI handling | Where the generated file lands is acceptable (native: user's device; flow: Dataverse file column) |
| Maintainability | An admin can update a template without code |
| Validation effort | Fewer components and fewer unverified behaviors wins |

Either path means **re-authoring the five templates**; the legacy tag mapping does not carry over. Native templates are chosen from a list on the case rather than from a checklist row, which is a UX change. Do not use the filled "Output Example" document from the old repo as a test input unless confirmed to contain no real patient data.

---

## 9. Data migration

1. Document the SharePoint list schema and map columns (Phase 0).
2. Build and rehearse with the **anonymized** sample (`sample-data/OPMListsDataAnon_sampledata.xlsx`) in dev and test.
3. Reconcile: row counts, case-number uniqueness, orphan subcomponents, stage values, designer matching to Entra users.
4. Production load runs under your PHI procedures, using Power Query dataflows or Excel import into prod, executed by authorized OPM staff. **It must not run through this repo or any Claude session.**
5. Securely delete the exported CSV/Excel afterward. Keep the SharePoint list read-only for a fallback period, then retire it.

---

## 10. Compliance and validation

- Confirm BAA coverage of the tenant and of every service and connector in use (Dataverse, Power Automate, Word, SharePoint, Teams, Outlook, Purview). Do not rely on preview features for PHI.
- Configure Dataverse auditing (retention default is forever; confirm log capacity) and Purview read logging. Verify Purview retention against your policy; I have not checked licensing tiers.
- QA/RA to decide: is this a QMS software application (ISO 13485 §4.1.6 / QMSR) and does Part 11 apply? Assume yes until told otherwise.
- If yes, run it through the OPM validation workflow (`/opm-validation:validate-start` and related skills): validation plan, requirements, FMEA, IQ/OQ/PQ, traceability.
- Change control: dev → test → prod as a **managed** solution; no direct edits in prod. Supplier assessment of Microsoft plus a release-wave review process.
- Questions for counsel or compliance: are surgery date and DOB treated as identifiers (the legacy app stores DOB in plaintext)? Does free-text notes risk containing PHI?

---

## 11. Repo layout and ALM

```
docs/                           ← this plan, validation docs
templates/                      ← blank Word templates (no PHI)
sample-data/                    ← anonymized sample only
power-platform/
  solutions/OPMCaseTracker/     ← pac solution unpack output (source of truth)
  plugins/OPM.CaseTracker/      ← C# plug-in + unit tests
  flows/                        ← exported flow definitions
  migration/                    ← mapping docs, dataflow definitions (no PHI)
```

CI: GitHub Actions running the Power Platform actions (or Power Platform Pipelines), with a solution checker gate and promotion dev → test → prod as managed.

---

## 12. Phases

Estimates are rough and low-confidence for one experienced builder; refine after Phase 0.

| Phase | Scope | Exit criteria | Est. |
|---|---|---|---|
| 0. Foundations | Verify business rules against legacy route code; requirements baseline; BAA/DLP/admin review; validation scoping; POCs (Word path, Purview read logging, system-admin visibility of secured columns, calculated/overdue columns) | POC results documented; scope signed off by QA | 1–2 wk |
| 1. Data model | Solution, tables, relationships, keys, seed config data, security roles, field security, Entra teams | Schema reviewed; role tests pass | 1–2 wk |
| 2. App | Sitemap, forms, views, dashboards, admin area | Walkthrough against every legacy screen | 2 wk |
| 3. Logic | C# plug-in, Custom API, checklist flow, time-target fields, unit tests | Design-lock and independent-advance tests pass | 2 wk |
| 4. Documents and notifications | Word path chosen in Phase 0 implemented for all five templates; Teams/Outlook flows; preferences | All five templates generate correctly; notifications carry no PHI | 1–2 wk |
| 5. Move UX | Custom page: stage lanes, click-to-move, bulk move with notes | Bulk move with notes works | 1 wk |
| 6. Validate and migrate | Rehearsal migrations, UAT, IQ/OQ/PQ, prod load, cutover | Reconciliation clean; validation package approved | 3–4 wk |
| 7. Hypercare | Monitoring, audit review, SharePoint list retirement | No open critical defects | 2 wk |

Overall: roughly 3–4 months, dominated by Phase 6 and validation.

---

## 13. Risks and how to verify

| Risk | Verify by |
|---|---|
| Word path fails on tables, control types or fidelity | Phase 0 POC with the five real templates |
| System Administrators can read secured PHI columns (documented default) | Confirm membership controls; test masking |
| Read logs do not identify records for views/exports (documented) | Remove export privileges; keep PHI out of views |
| Read auditing cost/volume and Purview retention | Enable on test; check log capacity and Purview retention |
| Calculated columns cannot express overdue | Prototype; fall back to stored due-on plus view filter |
| Premium licensing and capacity cost | Licensing contact; Power Platform admin center capacity report |
| Teams webhook retirement | Use Power Automate "Post message" instead of webhooks |
| Validated-state drift from Microsoft release waves | Supplier assessment plus release-wave review SOP |
| PHI exposure in migration files | PHI handling procedure; no PHI in repo, session or personal drives |
| C# toolchain unavailable to the builder | Confirm a machine with the .NET SDK and PAC CLI before Phase 3 |

---

## 14. Decisions still needed

1. Drop the legacy `sales_rep` role? (Plan assumes yes.)
2. Surgeons, sales reps and finish types as tables (recommended) or choices?
3. Has QA/RA decided whether this is in validation scope (13485 §4.1.6, Part 11)?
4. Should DOB and surgery date stay on the standard case form, or move behind the PHI role?
5. Who owns Entra group membership for roles?

---

## 15. Limits of the authoring session

`pac` and `dotnet` were not installed in the session that produced this plan and the canvas-authoring MCP failed to start (`dnx` missing), so nothing could be packed, built or deployed. Building and deploying needs a machine or session with the PAC CLI, the .NET SDK and a signed-in environment. Work that needs no environment: schema and relationship definitions, an ER diagram, the security role matrix, the C# plug-in and its unit tests, solution source files, and migration mapping specs.
