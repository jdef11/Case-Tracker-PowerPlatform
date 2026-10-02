# CLAUDE.md

Power Platform rebuild of the OFC Case Tracker. Read `docs/POWER_PLATFORM_PLAN.md` first; it holds the decisions, data model, security model and phases.

## Rules
- **Never commit or paste PHI** (patient names, DOBs, real case data, exports from the SharePoint list). Use `sample-data/` (anonymized) only.
- Legacy business rules come from the Next.js app in `jdef11/Case-Tracker`; Phase 0 verifies them against its API route code before building.
- UI labels use "Time Target" / "Overdue" / "Parts Overdue". Never "SLA". Status `in_progress` displays as "WIP".
- Design phase = stages 1-8 (all subcomponents in a case move together); manufacturing = stages 9-21 (independent).
- Tooling needed to build/deploy: PAC CLI, .NET SDK, a signed-in Dataverse dev environment. Check `pac --version` and `dotnet --version` before promising anything deployable.
- Do not use preview features with PHI.
