# Legacy modernization playbook (Oracle Forms / Reports / scripts → Spring Boot + Angular)

## Phases

| # | Phase | Output | Exit criterion |
|---|---|---|---|
| 1 | Inventory | List of screens, reports, triggers, packages, jobs, scripts | Every artefact has an owner + priority |
| 2 | Rule extraction | Business rules written in plain language + test cases | Business validates the rules, not the screens |
| 3 | Data access | Entities / native queries, grants & synonyms for cross-schema reads | Queries return the same rows as legacy |
| 4 | API | REST endpoints, DTOs, validation, security | Contract tests green |
| 5 | UI | Angular screens rebuilt for the workflow (not pixel copies) | UAT sign-off |
| 6 | Parallel run | Old and new side by side, compare outputs | Zero unexplained diff for N days |
| 7 | Cut-over | Decommission legacy, keep read-only archive | Rollback plan documented |

## Extracting logic from Oracle Forms

- Triggers to read first: `WHEN-VALIDATE-ITEM`, `PRE-INSERT`, `POST-QUERY`, `KEY-COMMIT`, `WHEN-BUTTON-PRESSED`.
- Map each trigger to: **validation** (→ Bean Validation / service guard), **derivation** (→ service method), **side effect** (→ transactional service + audit), **UI behaviour** (→ Angular).
- Hardcoded values (rates, codes) → replace with lookups from reference tables (e.g. a tax-rate table) so business can change them without a release.
- Historization/audit logic written in PL/SQL must be preserved explicitly — it is often a regulatory requirement.

## Cross-schema Oracle access

```sql
-- as owner of LEGACY schema
GRANT SELECT ON LEGACY.CLIENT TO APP_USER;
-- as APP_USER (or via DBA)
CREATE SYNONYM CLIENT FOR LEGACY.CLIENT;
```
Document every grant in the migration repo; missing grants are the #1 cause of "works in dev, fails in UAT".

## Migrating scripts (VBS, batch) 

- Re-implement as a small Spring Boot `CommandLineRunner` or scheduled job (with logs, retries, exit codes), or PowerShell if it must stay on a Windows host.
- Keep the same inputs/outputs first; improve after parity.

## Reuse strategy

After the first artefact of a family (e.g. first report), extract the pattern. The second one should reuse ~80% — if it doesn't, the abstraction is wrong.

## Common traps

- Copying PostgreSQL/MySQL syntax into Oracle native queries (`LIMIT`, `ILIKE`, `::type`). Use `FETCH FIRST n ROWS ONLY`, `UPPER(x) LIKE UPPER(:p)`, `CAST`.
- Null parameters in native queries: Oracle can't infer the type → use `(:p IS NULL OR col = :p)` with typed binding (`SqlParameterSource` + `Types.VARCHAR`).
- Assuming the legacy data model is flat: always inspect a real API/DB response before writing DTOs (hierarchies like parent → family → sub-family → item are common).
