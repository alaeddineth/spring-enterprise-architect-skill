# Technical Architecture Document — <Project name>

| Version | Date | Author | Status |
|---|---|---|---|
| 0.1 | YYYY-MM-DD | <name> | Draft / Validated |

## 1. Context & goals
- Business problem:
- Consumers (internal apps, partners, customers):
- Constraints (hosting, compliance, deadline, team):
- Out of scope:

## 2. Architecture overview
<Mermaid diagram: consumers → gateway → services/monolith → data>

## 3. Bounded contexts & deployment units
| Context | Unit (service / monolith module) | Owner | Data owned | Consumers |
|---|---|---|---|---|

## 4. Cross-cutting concerns
- Gateway & routing (dev / pre-prod / prod):
- Authentication & authorization:
- Error model, versioning, pagination:
- Observability (logs, metrics, tracing, correlation ID):
- Audit trail:
- Data access & connection pooling:

## 5. Key decisions (ADRs)
| ADR | Decision | Status |
|---|---|---|

## 6. Environments & delivery
- Branching: feature → dev → recette (UAT) → master
- CI/CD pipeline:
- Responsibilities (dev team vs system/platform team):

## 7. Delivery plan
| Lot | Scope | Developers | Effort (days) | Start |
|---|---|---|---|---|

## 8. Risks & mitigations
| Risk | Impact | Mitigation |
|---|---|---|
